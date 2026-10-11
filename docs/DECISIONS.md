# Decisions

Architecture/engineering decision log, **newest first**. Append-only: a change
of mind adds a new entry and marks the superseded one, rather than editing or
deleting history. Each entry is dated and has a status.

---

### ADR-0012: Retention is out of v1 — delegate to S3 lifecycle policies
*2026-10-12 · Accepted*

**Context.** A backup system must eventually expire old backups, but app-side
deletion is risk-bearing (a bug could delete live backups).
**Decision.** v1 does **not** delete anything in S3. Expiry is delegated to an
**S3 bucket lifecycle policy** configured on the `<prefix>` by the operator.
**Consequences.** Simpler, safer service; no delete credentials needed in
principle. Retention timing lives in bucket config, documented in `README.md`.
App-side rotation can be added later as its own ADR if required.

### ADR-0011: Verification = TOC read + sha256 + post-upload size check
*2026-10-12 · Accepted*

**Context.** "Backup verification" was requested; a full test-restore per run is
too expensive to mandate.
**Decision.** When `[backup] verify = true`: (a) read each archive's
table-of-contents with `pg_restore --list` (catches truncation/corruption),
(b) compute a sha256 recorded in the manifest, (c) after upload `HEAD` the object
and compare size. A failure marks that database `failed`.
**Consequences.** Cheap, catches the common failure modes, no second Postgres
needed. It is not a guarantee of logical restorability — a periodic real
test-restore remains an operator responsibility (possible future ADR).

### ADR-0010: Verbose structured result logging + run manifest
*2026-10-12 · Accepted*

**Context.** "Log verbose results" was requested; the inherited logging
convention is JSON-lines to stdout/journald.
**Decision.** Each run emits a structured per-database log line and a per-run
summary line (status, counts, bytes, duration), and writes a `manifest.json`
(per-object key, bytes, sha256, timings, verify result) uploaded alongside the
dumps. Schemas in `docs/CONTRACTS.md` §4–§5.
**Consequences.** Runs are auditable from `journalctl` and from the manifest in
S3 without extra infrastructure. Secrets are never logged.

### ADR-0009: Lightweight internal scheduler; full cron deferred
*2026-10-12 · Accepted*

**Context.** The daemon needs a schedule (ADR-0004). Full cron syntax would pull
in a dependency (e.g. `croniter`).
**Decision.** Ship a small in-process async scheduler driven by `[schedule]`:
fixed `interval` and/or `daily_at` wall-clock times, plus optional `jitter` and
`run_on_start`. No cron dependency in v1.
**Consequences.** Zero extra deps, covers the common "every N hours / daily at
HH:MM" needs. If true cron expressions become necessary, add them (and the
`croniter` dependency) under a new ADR.

### ADR-0008: Package dumps with stdlib `tarfile`; compression by pg_dump
*2026-10-12 · Accepted*

**Context.** Directory-format (`-Fd`) dumps are a directory and must become one
S3 object (ADR-0006). Its data files are already compressed by `pg_dump`.
**Decision.** Package each `-Fd` directory into a single object with the stdlib
`tarfile` module (offloaded via `asyncio.to_thread`); default `tar_format = tar`
(store, no recompression). Inner compression is `pg_dump --compress`, default
`zstd` (needs pg_dump 16+), falling back to `gzip`. `tar.gz` is available for
hosts that want an extra outer pass.
**Consequences.** No external `tar`/`zstd` binary dependency; good ratio from
pg_dump's own compression. Needs scratch disk for the directory + the tar.

### ADR-0007: S3 via `boto3` with a configurable endpoint
*2026-10-12 · Accepted*

**Context.** The target must be any S3-compatible backend (AWS, MinIO, Ceph,
Wasabi, B2), not AWS-only.
**Decision.** Use `boto3` with a configurable `endpoint_url`, `region`,
`addressing_style` (default `path` for MinIO/Ceph), and multipart tuning;
credentials come from the `s3-access-key-id` / `s3-secret-access-key` secrets.
boto3 calls are blocking, so they run via `asyncio.to_thread`.
**Consequences.** One client covers every S3 target. Access-key auth only in v1
(no AWS IAM-role path); that can be added later if a deployment needs it.

### ADR-0006: Dump strategy = per-database `pg_dump -Fd -j N` + globals
*2026-10-12 · Accepted*

**Context.** Backing up a whole cluster; options were per-DB custom (`-Fc`),
per-DB directory parallel (`-Fd -j`), or whole-cluster `pg_dumpall`.
**Decision.** Per-database **directory format with parallel workers**
(`pg_dump -Fd -j <pg_dump_jobs>`), each packaged into one object, plus
`pg_dumpall --globals-only` for roles/tablespaces/grants.
**Consequences.** Parallel per-table dumps are fast on large databases and allow
selective/parallel restore; cost is a packaging step and scratch disk (handled
by ADR-0008). `-Fd` cannot stream to stdout, unlike `-Fc` — accepted trade-off.
*(Alternatives `-Fc` and `pg_dumpall` considered and rejected.)*

### ADR-0005: Enumerate with `asyncpg`; dump/verify with the `pg_*` CLI; TLS verify-full
*2026-10-12 · Accepted*

**Context.** There is no pure-Python `pg_dump`; the CLI is the only faithful
dumper. Database discovery needs a query.
**Decision.** Use `asyncpg` to enumerate databases
(`datallowconn AND NOT datistemplate`, then include/exclude) and read the server
version; run `pg_dumpall`/`pg_dump`/`pg_restore` as async subprocesses.
Credentials pass via libpq env (`PGPASSWORD` from the `db-password` secret, never
on the command line, never logged). TLS defaults to `sslmode=verify-full` with a
configurable CA path (blank = public CA).
**Consequences.** Host must have `postgresql-client` (major ≥ server). Clean
separation: `asyncpg` for discovery, CLI for the heavy lifting.

### ADR-0004: Run model = long-running asyncio daemon with an internal scheduler
*2026-10-12 · Accepted*

**Context.** Choice between a oneshot unit + systemd timer and a persistent
daemon. (Inherited convention favors a `Type=notify` asyncio daemon.)
**Decision.** A persistent, single-process **asyncio daemon** (`Type=simple`
semantics realized as the inherited `Type=notify` readiness model) with an
internal scheduler; `Restart=on-failure`.
**Consequences.** Matches the inherited runtime layer directly (signal handling,
graceful shutdown, sd_notify). Scheduling logic lives in-process (ADR-0009)
rather than in systemd timers.

### ADR-0003: Naming scheme
*2026-10-12 · Accepted*

**Context.** The inherited conventions parameterize service name, OS user, env
prefix, and secret kinds; concrete names are needed.
**Decision.** Unit/dirs `pg-dump-to-s3`; importable package `pg_dump_to_s3`;
OS user/group `pgdumps3` (hyphen-free for `useradd`); env prefix `PGDUMPS3_`;
secret kinds `db-password`, `s3-access-key-id`, `s3-secret-access-key`.
**Consequences.** Consistent across docs, config, and host layout. The OS user
is deliberately hyphen-free to satisfy default `useradd` name rules.

### ADR-0002: Python 3.14+, asyncio-first
*2026-10-12 · Accepted*

**Context.** Inherited baseline is Python 3.14+ and "never block the loop."
**Decision.** Adopt `requires-python = ">=3.14"`; asyncio-first; subprocesses and
blocking library calls offloaded off the event loop.
**Consequences.** Host needs Python 3.14+. The floor is stated only in
`pyproject.toml`; it can be lowered later (new ADR) if a deployment target lacks
3.14.

### ADR-0001: Inherit general engineering conventions from `tgagent`
*2026-10-12 · Accepted*

**Context.** A new, unrelated service should not reinvent engineering standards.
The `tgagent` project (https://github.com/samthesuperhero/tgagent.git) documents
mature, project-neutral conventions.
**Decision.** Inherit `tgagent`'s **general, project-unrelated** conventions,
rules, and decisions wholesale — deps/packaging, ruff/EditorConfig style, INI +
`0400`-secret-file config model, JSON-lines logging, SemVer-in-`VERSION`, LF-only
git + human-confirmed commits, hermetic pytest, idempotent install/update, and
the `Type=notify` systemd + graceful-shutdown model. Record them in
`docs/CONVENTIONS.md` / `CONTRACTS.md`, re-scoped to this service.
**Consequences.** There is **no other relationship** between the projects.
Nothing Telegram/bot-specific (Telethon, the jobs platform, the web panel, its
SQL schema, its concrete names) is inherited.
