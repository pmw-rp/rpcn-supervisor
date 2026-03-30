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
├── bin/
│   ├── start-all       ← sync + launch root s6-svscan (background)
│   ├── stop-all        ← stop all pipelines then stop s6-svscan
│   ├── start-pipeline  ← start or register a single pipeline
│   ├── stop-pipeline   ← stop a single pipeline
│   ├── status          ← show pipeline states and PIDs
│   ├── logs            ← tail logs for one or all pipelines
│   ├── sync            ← reconcile internal/pipelines-run/ from pipelines/ (also runs every 5s)
│   └── internal/
│       └── make-service    ← create pipeline + health-check service dirs
├── pipelines/              ← source of truth: one .yaml per pipeline
└── internal/               ← all runtime and config internals (gitignored except etc/)
    ├── etc/
    │   ├── services/       ← static root svscan service definitions (committed)
    │   │   ├── pipelines-svscan/   ← supervises internal/pipelines-run/
    │   │   └── internal-svscan/    ← supervises internal/internal-run/
    │   └── internal/       ← static internal service definitions (committed)
    │       └── sync-watcher/       ← polls bin/sync every 5s
    ├── services/           ← runtime: populated from etc/services/ by start-all
    ├── pipelines-run/      ← runtime: generated pipeline service dirs
    ├── internal-run/       ← runtime: generated health-check dirs + sync-watcher
    └── logs/
        ├── pipelines/<name>/current   ← pipeline output logs
        └── internal/                  ← health-check and sync-watcher logs
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

Drop a Redpanda Connect YAML config into `pipelines/`:

```bash
cp my-pipeline.yaml pipelines/my-pipeline.yaml
```

The `sync-watcher` service picks it up automatically within 5 seconds. To
register it immediately:

```bash
bin/sync
```

### Remove a pipeline

```bash
rm pipelines/my-pipeline.yaml
```

Again, auto-removed within 5 seconds, or immediately with `bin/sync`.

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
in `internal/logs/pipelines/<name>/current` and rotate automatically (up to 20 files, 1 MB each).

---

## How it works

```
pipelines/*.yaml
       │
       ▼  (every 5s via sync-watcher, or manually via bin/sync)
bin/sync  ← reconciler: diffs pipelines/ against internal/pipelines-run/, idempotent
       │
       ├── make-service alpha   → internal/pipelines-run/alpha/        (pipeline)
       │                          internal/internal-run/alpha-health/  (health-check)
       └── (remove) s6-svc -d  → rm -rf internal/pipelines-run/old/

s6-svscan internal/services/              ← root supervisor
├── s6-supervise pipelines-svscan  →  s6-svscan internal/pipelines-run/
│       ├── s6-supervise alpha     →  rpk connect run alpha.yaml
│       ├── s6-supervise beta      →  rpk connect run beta.yaml
│       └── s6-supervise gamma     →  rpk connect run gamma.yaml
└── s6-supervise internal-svscan  →  s6-svscan internal/internal-run/
        ├── s6-supervise alpha-health  →  polls /ready, restarts on failure
        ├── s6-supervise beta-health
        ├── s6-supervise gamma-health
        └── s6-supervise sync-watcher  →  sleep 5; bin/sync; loop
```

**`bin/sync`** diffs `pipelines/` against `internal/pipelines-run/` and creates or
removes service dirs to match. It is idempotent and safe to run at any time. It
also signals the sub-svscans to rescan immediately after any change.

**`bin/internal/make-service`** resolves the absolute paths to `rpk` and `s6-log` from
`$PATH` at creation time and bakes them into the generated `run` scripts. This means the
scripts are portable across platforms (macOS Homebrew, Linux system packages, custom
installs) without any code changes — the resolution happens once, when you run `sync`.

**HTTP ports** are auto-assigned and persisted in `internal/ports`. Each pipeline gets a
unique port starting from 4195 (Redpanda Connect's default), incrementing by 1 for each
new pipeline. The port is injected at runtime via `--set http.enabled=true --set
http.address=0.0.0.0:<port>` — the pipeline YAML files are not modified. Assignments are
stable: once written to `internal/ports` the same port is reused on every `sync`
run. To reassign a port, remove the line from `internal/ports`, delete the service dir
from `internal/pipelines-run/`, and run `bin/sync`.

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
`bin/internal/make-service`:

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

All three write JSON lines to stdout, which s6-log captures into `internal/logs/pipelines/<name>/current`.

---

## Limitations

**Sync latency.** Pipeline configs are picked up automatically by the `sync-watcher`
service every 5 seconds. For immediate registration, run `bin/sync` manually.

**One process per pipeline.** Each pipeline is a single `rpk connect` process. There is
no horizontal scaling or work distribution. For high-throughput pipelines you would need
multiple instances or a proper orchestrator.

**No dependency ordering.** All pipelines start concurrently. If one pipeline depends on
another being ready first, nothing here enforces that ordering.

**Crash loop backoff.** Each pipeline's `run` script implements exponential backoff
matching Kubernetes CrashLoopBackOff timings (10s → 20s → 40s → 80s → 160s → 300s cap).
The counter resets after 10 minutes of clean uptime. State is stored in
`internal/pipelines-run/<name>/state/`. To manually clear the backoff for a pipeline:

```bash
rm internal/pipelines-run/my-pipeline/state/crash_count
s6-svc -u internal/pipelines-run/my-pipeline
```
