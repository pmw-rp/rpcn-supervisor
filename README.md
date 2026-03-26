# rpcn-supervisor

Run multiple [Redpanda Connect](https://docs.redpanda.com/redpanda-connect/) ETL pipelines as independent, supervised processes using [s6](https://skarnet.org/software/s6/).

Each pipeline is defined by a single YAML config file. Dropping a file into `pipelines/` registers it; removing the file tears it down. s6 handles process supervision, automatic restart on crash, and structured log rotation.

---

## Requirements

- [`rpk`](https://docs.redpanda.com/current/get-started/rpk-install/) (includes `rpk connect`)
- [`s6`](https://skarnet.org/software/s6/)

```bash
# macOS
brew install s6

# Debian/Ubuntu
apt-get install s6

# Alpine
apk add s6
```

---

## Directory layout

```
rpcn-supervisor/
├── pipelines/          ← source of truth: one .yaml per pipeline
├── services/           ← s6 service dirs (generated, do not edit)
├── logs/
│   └── <name>/
│       └── current     ← active log file (rotated automatically)
└── bin/
    ├── start-all       ← sync + launch s6-svscan (foreground)
    ├── stop-all        ← stop all pipelines (svscan keeps running)
    ├── start-pipeline  ← start or register a single pipeline
    ├── stop-pipeline   ← stop a single pipeline
    ├── status          ← show all pipeline states and PIDs
    ├── logs            ← tail logs for one or all pipelines
    ├── sync            ← reconcile services/ from pipelines/
    └── internal/
        └── make-service    ← create one s6 service dir (called by sync)
```

---

## Quickstart

```bash
# 1. Install dependencies (macOS)
brew install s6

# 2. Start all pipelines (returns to shell prompt)
bin/start-all

# 3. Check status
bin/status
```

---

## Managing pipelines

### Add a pipeline

Drop a Redpanda Connect YAML config into `pipelines/` and run `sync`:

```bash
cp my-pipeline.yaml pipelines/my-pipeline.yaml
bin/sync
```

`sync` creates the s6 service dir. If `s6-svscan` is already running it will
detect the new directory within a few seconds and start the pipeline automatically.

### Remove a pipeline

```bash
rm pipelines/my-pipeline.yaml
bin/sync
```

`sync` signals s6-supervise to stop the process (waiting up to 5 seconds for a
clean exit) and then removes the service dir.

### Stop and start individual pipelines

```bash
# Stop one pipeline (service dir is preserved; svscan keeps running)
bin/stop-pipeline my-pipeline

# Start it again
bin/start-pipeline my-pipeline

# Stop all pipelines at once (svscan keeps running)
bin/stop-all

# Start all back up after stop-all
bin/start-pipeline --all
```

### Stop everything

```bash
bin/stop-all
```

This stops all pipelines cleanly, then terminates `s6-svscan`.

### View logs

```bash
# Follow one pipeline
bin/logs my-pipeline

# Follow all pipelines (lines prefixed with pipeline name)
bin/logs
```

Timestamps are decoded from s6's TAI64N format to local time automatically if
`s6-tai64nlocal` is available (it is included with the `s6` package). Raw logs are
in `logs/<name>/current` and rotate automatically (up to 20 files, 1 MB each).

---

## How it works

```
pipelines/*.yaml
       │
       ▼
bin/sync          ← reconciles filesystem, idempotent
       │
       ├── bin/internal/make-service alpha pipelines/alpha.yaml
       │       └── services/alpha/run        exec rpk connect run ...
       │           services/alpha/log/run    exec s6-log ...
       │
       └── (remove) s6-svc -d services/old → rm -rf services/old
                │
                ▼
          s6-svscan services/
          ├── s6-supervise alpha  →  rpk connect run alpha.yaml
          │       └── s6-supervise alpha/log  →  s6-log → logs/alpha/
          ├── s6-supervise beta   →  rpk connect run beta.yaml
          │       └── s6-supervise beta/log   →  s6-log → logs/beta/
          └── ...
```

**`bin/sync`** is a reconciler. It diffs `pipelines/` against `services/` and
creates or removes service dirs to match. It is idempotent and safe to run at any time.

**`bin/make-service`** resolves the absolute paths to `rpk` and `s6-log` from `$PATH` at
creation time and bakes them into the generated `run` scripts. This means the scripts are
portable across platforms (macOS Homebrew, Linux system packages, custom installs) without
any code changes — the resolution happens once, when you run `sync` or
`make-service` directly.

**HTTP ports** are auto-assigned and persisted in `pipelines/ports`. Each pipeline gets a
unique port starting from 4195 (Redpanda Connect's default), incrementing by 1 for each
new pipeline. The port is injected at runtime via `--set http.enabled=true --set
http.address=0.0.0.0:<port>` — the pipeline YAML files are not modified. Assignments are
stable: once written to `pipelines/ports` the same port is reused on every `sync`
run. To reassign a port, remove the line from `pipelines/ports`, delete the service dir,
and run `bin/sync`.

**`s6-svscan`** scans its directory every 5 seconds. Any subdirectory containing an
executable `run` script is treated as a service. It spawns a dedicated `s6-supervise`
process for each, which starts the program and restarts it automatically if it exits.

**Lifecycle is driven by the filesystem.** There is no daemon, no API, no database.
A config file's presence in `pipelines/` is the only signal needed to run a pipeline.

---

## Secrets management

Redpanda Connect supports `${ENV_VAR}` substitution in YAML configs. How you supply
those variables depends on your environment:

### `.env` file (recommended for local/dev)

```bash
# .env  — never commit this
KAFKA_PASSWORD=secret123
API_KEY=abc456
```

```yaml
# pipelines/my-pipeline.yaml
input:
  kafka_franz:
    seed_brokers: ["${KAFKA_BROKER}"]
    sasl:
      - mechanism: PLAIN
        password: "${KAFKA_PASSWORD}"
```

To pass the env file through to `rpk connect`, update the `run` script line in
`bin/make-service`:

```sh
exec ${RPK} connect run --env-file ${BASE_DIR}/.env ${CONFIG} 2>&1
```

### Environment variables in the process environment

Set variables before calling `bin/start`, or export them in your shell profile.
s6-supervise inherits the environment of the `s6-svscan` process.

### HashiCorp Vault / AWS Secrets Manager / GCP Secret Manager

Redpanda Connect has [native secrets manager support](https://docs.redpanda.com/redpanda-connect/components/secrets/).
Secrets are fetched at pipeline startup — no env files, no plaintext on disk:

```yaml
# config snippet — Vault example
secrets:
  - label: vault
    hashicorp_vault:
      address: "${VAULT_ADDR}"
      token: "${VAULT_TOKEN}"

input:
  kafka_franz:
    sasl:
      - mechanism: PLAIN
        password: "${vault:secret/kafka#password}"
```

### SOPS

Encrypt your `.env` or config files with [SOPS](https://github.com/getsops/sops) and
decrypt at deploy time. Works well with git-committed secrets and AWS KMS / GCP KMS /
age keys.

---

## Example pipelines

| Config | Description |
|--------|-------------|
| `pipelines/alpha.yaml` | Generates a UUID + random integer every 2s |
| `pipelines/beta.yaml` | Generates a UUID + random string payload every 3s |
| `pipelines/gamma.yaml` | Picks a random word from a fixed list every 5s |

All three write JSON lines to stdout, which s6-log captures into `logs/<name>/current`.

---

## Limitations

**Manual sync required at runtime.** `s6-svscan` detects new service directories
automatically, but `bin/sync` must be called to *create* those directories from
new config files. There is no inotify/FSEvents watcher — adding a config while the
supervisor is running requires a manual `bin/sync` call.

**One process per pipeline.** Each pipeline is a single `rpk connect` process. There is
no horizontal scaling or work distribution. For high-throughput pipelines you would need
multiple instances or a proper orchestrator.

**No dependency ordering.** All pipelines start concurrently. If one pipeline depends on
another being ready first, nothing here enforces that ordering.

**Crash loop backoff.** Each pipeline's `run` script implements exponential backoff
matching Kubernetes CrashLoopBackOff timings (10s → 20s → 40s → 80s → 160s → 300s cap).
The counter resets after 10 minutes of clean uptime. State is stored in
`services/<name>/state/`. To manually clear the backoff for a pipeline:

```bash
rm services/my-pipeline/state/crash_count
s6-svc -u services/my-pipeline
```

**Foreground only.** `bin/start` runs `s6-svscan` in the foreground. For a background
daemon, wrap it in a system service (launchd on macOS, systemd on Linux) or run it inside
a terminal multiplexer.
