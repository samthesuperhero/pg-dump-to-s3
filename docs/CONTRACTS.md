# Contracts

The stable, observable behavior of `pg-dump-to-s3`: configuration surface,
secrets, logging and manifest schemas, the backup-run contract, verification,
the S3 object layout, and exit codes. Keep this document in lockstep with
`config.example.ini` — every config key appears in both.

---

## 1. Versioning

SemVer, single-sourced from `VERSION`. MAJOR = breaking change to this contract
(config keys, object layout, manifest schema, exit codes); MINOR = backward-
compatible addition; PATCH = fix.

## 2. Configuration

- Source precedence (low→high): **code defaults → INI → environment**.
- INI location on the host: `/etc/pg-dump-to-s3/pg-dump-to-s3.ini`, mode `0444`,
  **no secrets**. Interpolation disabled (a `%` is literal).
- Environment override form: **`PGDUMPS3_<SECTION>__<KEY>`** — uppercase; the
  double underscore `__` separates section from key; hyphens in a name become
  underscores. Examples: `PGDUMPS3_POSTGRES__HOST`, `PGDUMPS3_S3__BUCKET`,
  `PGDUMPS3_SERVICE__LOG_LEVEL`.
- Every key is optional (has a default). A single malformed value falls back to
  its default and logs a warning; a file that will not parse at all exits **78**.

### `[service]`

| Key | Type | Default | Meaning |
|---|---|---|---|
| `log_level` | enum | `INFO` | `DEBUG`/`INFO`/`WARNING`/`ERROR`/`CRITICAL`; unknown → INFO (warn once). |
| `scratch_dir` | path | `/var/lib/pg-dump-to-s3/scratch` | Working dir for in-progress dumps + packaging. |
| `max_parallel_databases` | int | `2` | Databases dumped concurrently within a run. |

### `[schedule]`

| Key | Type | Default | Meaning |
|---|---|---|---|
| `interval` | duration | `24h` | Gap between runs (`s`/`m`/`h`/`d`). Blank disables interval triggering. |
| `daily_at` | list[HH:MM] | *(empty)* | Comma-separated wall-clock triggers (service-local time). |
| `jitter` | duration | `0` | Max random delay added to each trigger. |
| `run_on_start` | bool | `false` | Run one backup immediately at daemon startup. |

If `interval` and `daily_at` are both blank, the daemon idles (no scheduled runs).

### `[postgres]`

| Key | Type | Default | Meaning |
|---|---|---|---|
| `host` | str | `localhost` | Cluster host; with `verify-full` it must match the cert domain (not an IP). |
| `port` | int | `5432` | Cluster port. |
| `user` | str | `postgres` | Connect/dump role; its password is the `db-password` secret. |
| `sslmode` | enum | `verify-full` | libpq sslmode. |
| `ca_cert` | path | *(empty)* | Private CA PEM for server verification; blank for a public CA. |
| `connect_timeout` | int (s) | `15` | Enumeration connect timeout. |
| `command_timeout` | int (s) | `30` | Enumeration statement timeout. |
| `include_databases` | list | *(empty)* | Allowlist; blank = all non-template DBs. |
| `exclude_databases` | list | *(empty)* | Denylist, applied after the allowlist. |
| `pg_dump_jobs` | int | `4` | `pg_dump -Fd -j N` per-table workers per database. |
| `compress` | str | `zstd` | Inner dump compression `zstd`/`gzip`/`none` (`:level` optional); zstd needs pg_dump 16+. |

### `[s3]`

| Key | Type | Default | Meaning |
|---|---|---|---|
| `endpoint_url` | url | *(empty)* | S3 endpoint; blank = AWS default for the region. |
| `region` | str | `us-east-1` | Region. |
| `bucket` | str | *(empty, required)* | Existing target bucket. |
| `prefix` | str | `pg-dump-to-s3` | Key prefix under the bucket. |
| `addressing_style` | enum | `path` | `path`/`virtual`/`auto` (path suits MinIO/Ceph). |
| `storage_class` | str | *(empty)* | Optional S3 storage class; blank = bucket default. |
| `multipart_threshold` | size | `64MB` | Size above which multipart upload is used. |
| `multipart_chunksize` | size | `16MB` | Multipart part size. |

### `[backup]`

| Key | Type | Default | Meaning |
|---|---|---|---|
| `cluster_name` | str | `default` | Label for this cluster in object keys and the manifest. |
| `include_globals` | bool | `true` | Dump globals via `pg_dumpall --globals-only`. |
| `verify` | bool | `true` | Verify each dump (TOC read + sha256 + post-upload size check). |
| `tar_format` | enum | `tar` | Package each `-Fd` dir as `tar` (store) or `tar.gz` (recompress). |

## 3. Secrets

Flat `0400` files in the config dir, one per role-named kind. Never committed,
never logged.

| Kind | File | Used for |
|---|---|---|
| `db-password` | `/etc/pg-dump-to-s3/pg-dump-to-s3.db-password` | `PGPASSWORD` for `asyncpg` + `pg_*`. |
| `s3-access-key-id` | `/etc/pg-dump-to-s3/pg-dump-to-s3.s3-access-key-id` | S3 access key id. |
| `s3-secret-access-key` | `/etc/pg-dump-to-s3/pg-dump-to-s3.s3-secret-access-key` | S3 secret access key. |

## 4. Logging schema

One JSON object per line on stdout:

```json
{"ts":"2026-10-12T02:00:00.123456+00:00","level":"INFO","logger":"pg_dump_to_s3.service","service":"pg-dump-to-s3","msg":"...", "...extra": "..."}
```

Always present: `ts` (ISO-8601 UTC), `level`, `logger`, `service`, `msg`.
Structured `extra` fields are merged in at the top level. `exc` carries a
formatted traceback on exceptions. Secret values never appear.

**Per-database line** (`extra`): `run_id`, `database`, `status` (`ok`/`failed`),
`bytes`, `sha256`, `dump_seconds`, `upload_seconds`, `verify`
(`ok`/`failed`/`skipped`), `object_key`, and `error` on failure.

**Per-run summary line** (`extra`): `run_id`, `cluster_name`, `status`
(`ok`/`partial`/`failed`), `databases_total`, `databases_ok`,
`databases_failed`, `bytes_total`, `duration_seconds`, `manifest_key`.

## 5. Run manifest schema

Written per run and uploaded as `manifest.json` alongside the dumps:

```json
{
  "service": "pg-dump-to-s3",
  "version": "0.1.0",
  "cluster_name": "default",
  "run_id": "2026-10-12T020000Z",
  "started_at": "2026-10-12T02:00:00+00:00",
  "finished_at": "2026-10-12T02:07:41+00:00",
  "status": "ok",
  "postgres": { "host": "db.example.com", "server_version": "16.3" },
  "globals": {
    "object_key": "pg-dump-to-s3/default/2026/10/2026-10-12T020000Z/globals.sql",
    "bytes": 20480, "sha256": "…", "status": "ok"
  },
  "databases": [
    {
      "name": "app",
      "object_key": "pg-dump-to-s3/default/2026/10/2026-10-12T020000Z/app.tar",
      "bytes": 183500800, "sha256": "…",
      "dump_seconds": 61.2, "upload_seconds": 12.9,
      "verify": "ok", "status": "ok", "error": null
    }
  ],
  "counts": { "total": 1, "ok": 1, "failed": 0 }
}
```

`status`: `ok` = every database and globals succeeded; `partial` = at least one
succeeded and at least one failed; `failed` = nothing usable was produced.

## 6. Error envelope

Structured error payloads (in logs and any future surface) use:

```json
{ "error": { "code": "config_invalid", "message": "Human-readable summary.", "details": {} } }
```

`code` is a stable machine-readable string; `message` is for humans; `details`
is optional structured context.

## 7. Backup-run contract

A single run, triggered by the scheduler (or `run_on_start`), does the following.
Runs never overlap: if one is still active when the next trigger fires, the new
trigger is **skipped with a warning**.

1. **Enumerate** databases over `asyncpg`:
   `SELECT datname FROM pg_database WHERE datallowconn AND NOT datistemplate`,
   then apply `include_databases` (if set) and remove `exclude_databases`.
2. **Globals** (if `include_globals`): `pg_dumpall --globals-only` → `globals.sql`.
3. **Per database**, bounded by `max_parallel_databases`:
   `pg_dump -Fd -j <pg_dump_jobs> --compress=<compress>` into a per-DB scratch
   directory, then **package** that directory into a single object
   (`<db>.tar` / `<db>.tar.gz` per `tar_format`) via stdlib `tarfile`.
4. **Verify** (if `verify`): read the archive's table of contents
   (`pg_restore --list`) and compute the object's sha256 before upload.
5. **Upload** each object to S3 (multipart per `[s3]` tuning), using the
   `s3-access-key-id` / `s3-secret-access-key` credentials and `endpoint_url`.
6. **Post-upload check** (if `verify`): `HEAD` the object and compare size.
7. **Manifest**: assemble and upload `manifest.json` (§5).
8. **Clean up** the scratch directory (always, even on failure; bounded).
9. **Log** a per-database line per DB and one per-run summary line (§4).

Failure semantics: a single database's failure does not abort the run — other
databases still proceed; the run's overall `status` becomes `partial` or
`failed` accordingly. The daemon stays up regardless of a run's outcome
(outcomes are logged, not fatal).

## 8. Verification contract

When `[backup] verify = true`, each produced archive is checked three ways:
(a) its TOC is readable via `pg_restore --list` (a corrupt/truncated archive
fails here); (b) a sha256 is computed and recorded in the manifest; (c) after
upload, the S3 object is `HEAD`-ed and its size compared to the local size. Any
check failing marks that database `failed` in the manifest and the summary log.

## 9. S3 object layout

```
s3://<bucket>/<prefix>/<cluster_name>/<YYYY>/<MM>/<run_id>/
    globals.sql
    <database>.tar            # or <database>.tar.gz
    manifest.json
```

`<run_id>` is a UTC timestamp token (`YYYY-MM-DDTHHMMSSZ`). A single database is
restored with `pg_restore` (e.g. `-j 4`) after extracting its archive.

## 10. Exit codes

| Code | Meaning |
|---|---|
| `0` | Normal: `--check` passed, or the daemon shut down gracefully on a signal. |
| `78` | `EX_CONFIG`: config missing/invalid (also from `ExecStartPre --check`). Never restarted into (`RestartPreventExitStatus=78`). |
| other non-zero | Unexpected fatal error at startup. |

Note: an individual backup run's failure is reported in logs/manifest and does
**not** change the daemon's process exit — the daemon keeps running and will
attempt the next scheduled run.
