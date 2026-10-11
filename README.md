# pg-dump-to-s3

A headless Python service that runs on Linux as a **systemd unit** and backs up
every database in a **PostgreSQL cluster** to **S3-compatible** object storage
using `pg_dump` / `pg_dumpall`.

- **Long-running asyncio daemon**, single process, systemd `Type=notify`, with an
  internal scheduler.
- Per-database `pg_dump -Fd -j N` (directory format, parallel per-table workers),
  each packaged into one tar object, plus `pg_dumpall --globals-only` for
  roles/tablespaces/grants.
- Any S3 backend via a configurable `endpoint_url` (AWS S3, MinIO, Ceph, Wasabi,
  Backblaze B2).
- Backup **verification** (TOC read + checksum + post-upload size check) and
  **verbose structured logging** of every run.

> **Project status:** this repository currently contains the project
> *specifications* (binding rules, conventions, contracts, architecture,
> decisions) and the convention-encoding config files. The application code and
> the `install` / `update` / systemd deployment tooling described below are
> implemented in later tasks.

## Documentation map

| Document | Purpose |
|---|---|
| [`CLAUDE.md`](CLAUDE.md) | Binding rules for anyone (human or agent) working in this repo. |
| [`docs/CONVENTIONS.md`](docs/CONVENTIONS.md) | General engineering conventions (language, deps, style, logging, config, layout, git, tests, deploy). |
| [`docs/CONTRACTS.md`](docs/CONTRACTS.md) | Behavioral contracts: config/env, secrets, log & manifest schemas, the backup-run contract, exit codes. |
| [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) | Daemon shape, module layout, run data-flow, concurrency model. |
| [`docs/DECISIONS.md`](docs/DECISIONS.md) | Decision log (ADR-style, newest first). |
| [`config.example.ini`](config.example.ini) | The full catalog of configuration keys. |

## Deployment model (intended)

The service targets **two commands on a fresh box**:

```sh
sudo git clone <this-repo> /opt/pg-dump-to-s3
cd /opt/pg-dump-to-s3 && sudo ./install
```

`install` is idempotent: safe to re-run; a re-run on an already-correct host
changes nothing and exits 0. Updates are:

```sh
cd /opt/pg-dump-to-s3 && sudo ./update
```

`update` pulls the latest code, reconciles the host, and restarts the unit only
if the unit, code, config, or anything it provisions changed.

### What `install` asks you

The installer only *asks* for values it cannot know; it never invents a secret,
and reads each with echo off:

| Prompt | Stored as | Mode |
|---|---|---|
| PostgreSQL password for `[postgres] user` | `/etc/pg-dump-to-s3/pg-dump-to-s3.db-password` | `0400` |
| S3 access key id | `/etc/pg-dump-to-s3/pg-dump-to-s3.s3-access-key-id` | `0400` |
| S3 secret access key | `/etc/pg-dump-to-s3/pg-dump-to-s3.s3-secret-access-key` | `0400` |

Everything else is non-secret and lives in
`/etc/pg-dump-to-s3/pg-dump-to-s3.ini` (see `config.example.ini`).

### Who-does-what on the host

| Path | Purpose | Owner:Group | Mode |
|---|---|---|---|
| `/opt/pg-dump-to-s3/` | prod code (git checkout) | dir → `pgdumps3`; files root-owned | `0750` |
| `/opt/pg-dump-to-s3/.venv/` | virtualenv | `pgdumps3:pgdumps3` | `0750` |
| `/etc/pg-dump-to-s3/` | config + secrets dir | `root:root` | `0755` |
| `/etc/pg-dump-to-s3/pg-dump-to-s3.ini` | config (no secrets) | `pgdumps3:pgdumps3` | `0444` |
| `/etc/pg-dump-to-s3/pg-dump-to-s3.<kind>` | a secret | `pgdumps3:pgdumps3` | `0400` |
| `/var/lib/pg-dump-to-s3/` | persistent data + dump scratch | `pgdumps3:pgdumps3` | `0750` |
| `/run/pg-dump-to-s3/` | runtime dir (systemd `RuntimeDirectory`) | `pgdumps3:pgdumps3` | `0750` |

### Host prerequisites

- Python **3.14+**.
- `postgresql-client` whose **major version ≥ the server's** (`pg_dump`,
  `pg_dumpall`, `pg_restore` must be on `PATH`). For `compress = zstd`, the
  client must be PostgreSQL 16+ with libzstd.
- A reachable PostgreSQL cluster and a role that can read every target database.
- An existing S3 bucket and credentials that can write (and HEAD) objects in it.

## Backups in S3

Each run writes, under `s3://<bucket>/<prefix>/<cluster_name>/<YYYY>/<MM>/<run-timestamp>/`:

- `globals.sql` — cluster roles/tablespaces/grants (`pg_dumpall --globals-only`).
- `<database>.tar` — one archive per database (packaged `pg_dump -Fd`).
- `manifest.json` — the run manifest (per-object size, sha256, timings, verify result).

**Restore** a single database with `pg_restore` (parallelizable, e.g. `-j 4`)
after unpacking its `.tar`. See `docs/CONTRACTS.md` for the exact object and
manifest layout.

### Retention

The service does **not** delete old backups (v1). Expiry is delegated to an **S3
bucket lifecycle policy** on the `<prefix>` — configure it on the bucket to
expire or transition objects after your retention window.
