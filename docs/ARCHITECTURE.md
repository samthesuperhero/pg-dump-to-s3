# Architecture

How `pg-dump-to-s3` is put together. This describes the intended implementation;
the code lands in a later task. Behavior is contracted in `docs/CONTRACTS.md`;
the rules are in `CLAUDE.md`.

---

## Shape

A single-process, **asyncio-first** daemon run as a systemd `Type=notify` unit.
One event loop owns everything. The one hard invariant is **never block the
loop**: external processes (`pg_dump`/`pg_dumpall`/`pg_restore`) are driven with
`asyncio.create_subprocess_exec`, and blocking library work (`boto3`, stdlib
`tarfile`/`hashlib`) is offloaded with `asyncio.to_thread`.

Composition is explicit — no framework, no DI container. A `Service` object is
built by an `async with build_service() as svc:` context manager and exposes
`start()` / `shutdown()`.

```
systemd (Type=notify)
  └─ python -m pg_dump_to_s3 --config /etc/pg-dump-to-s3/pg-dump-to-s3.ini
       └─ asyncio loop
            Service
              ├─ Scheduler ── fires ──▶ one BackupRun at a time (no overlap)
              └─ BackupRun
                   catalog → dump(globals, per-db) → package → verify → s3 upload → manifest
```

## Module layout (`src/pg_dump_to_s3/`)

| Module | Responsibility |
|---|---|
| `__main__.py` | Entrypoint: arg parsing (`--config`, `--check`), logging setup, signal handlers, build + run the service, `sd_notify` readiness. `--check` validates config/secrets and exits (0 or 78). |
| `service.py` | The `Service`: wires config + scheduler + the backup orchestration; `start()`/`shutdown()`. |
| `scheduler.py` | Lightweight internal scheduler: computes the next trigger from `[schedule]` (`interval` and/or `daily_at`, plus `jitter`), awaits it, and invokes one backup run. Guarantees no overlapping runs. |
| `catalog.py` | Enumerate target databases over `asyncpg`; apply include/exclude; read the server version. |
| `dump.py` | Run `pg_dumpall --globals-only` and per-database `pg_dump -Fd -j N --compress=…` as async subprocesses, with libpq env set from config + the `db-password` secret. |
| `package.py` | Package a per-DB `-Fd` directory into one tar object via stdlib `tarfile`, offloaded with `asyncio.to_thread`. |
| `verify.py` | `pg_restore --list` TOC check and sha256 (`hashlib`, offloaded). |
| `s3.py` | boto3 client/session from `[s3]` + S3 secrets; multipart upload (offloaded) and post-upload `HEAD` size check. |
| `manifest.py` | Build and serialize the run manifest (`docs/CONTRACTS.md` §5). |
| `runtime/` | Reusable, project-neutral runtime layer (below). |

### `runtime/` (reusable layer)

Generic infrastructure, kept **import-light / pure-stdlib** where it must load
before the venv exists (notably `provision.py`, imported by `install.py` as
root):

| Module | Responsibility |
|---|---|
| `config.py` | Load/merge config: defaults → INI (`configparser`, interpolation off) → `PGDUMPS3_*` env; typed getters with per-key fallback; raise → exit 78. |
| `secrets.py` | Read a secret by kind from the config dir; enforce presence; never log the value. |
| `logging.py` | `JsonFormatter` + root-handler setup; level coercion (unknown → INFO). |
| `version.py` | Read `VERSION`; expose service name + version for logs/manifest. |
| `sdnotify.py` | `sd_notify` readiness/stopping messages (`NOTIFY_SOCKET`). |
| `runner.py` | asyncio entrypoint: signal handlers → `asyncio.Event`, bounded graceful shutdown. |
| `app.py` | `Service` base + `build_service()` context manager. |
| `provision.py` | Idempotent install/update engine (stdlib-only): users, dirs/modes, venv, config merge, secret prompts, systemd unit. |

## Backup run data-flow

See `docs/CONTRACTS.md` §7 for the normative sequence. In short: enumerate →
globals → per-database (dump → package → verify) → upload → manifest → cleanup →
summarize. Each database is an independent unit of work; one failing does not
abort the others (the run ends `partial`/`failed` accordingly).

## Concurrency model

- **One run at a time.** The scheduler will not start a run while another is
  active (it skips the trigger and warns) — avoids doubling cluster and disk load.
- **Within a run**, databases are dumped concurrently up to
  `max_parallel_databases` (an `asyncio.Semaphore`). Each database's `pg_dump`
  may itself use `pg_dump_jobs` parallel workers, so plan host CPU/IO and the
  Postgres connection budget around roughly `max_parallel_databases ×
  pg_dump_jobs`.
- **Scratch disk**: each in-flight database needs room for its `-Fd` directory
  *and* its tar in `scratch_dir`; cleanup is prompt and always runs. Size
  `scratch_dir` for the largest concurrent set.
- **Shutdown** is bounded: on SIGTERM/SIGINT the scheduler stops, the in-flight
  run is allowed to drain (or is cancelled) and resources close within
  `TimeoutStopSec`; partial scratch is cleaned.

## External dependencies at runtime

- `asyncpg` — database enumeration only (no ORM, SQL by hand).
- `boto3` — S3-compatible uploads and `HEAD` checks.
- `pg_dumpall` / `pg_dump` / `pg_restore` CLI — the dump + TOC verification.
- stdlib `tarfile` (packaging), `hashlib` (checksums), `configparser`,
  `asyncio`, `subprocess`, `logging`, `json`.
