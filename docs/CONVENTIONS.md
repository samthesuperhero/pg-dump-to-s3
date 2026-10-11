# Conventions

General engineering conventions for `pg-dump-to-s3`. These are **inherited from
the unrelated `tgagent` project** as reusable, project-neutral standards and
re-scoped to this service. Nothing Telegram/bot-specific is carried over.

`CLAUDE.md` is the short binding-rules version; this document is the detail.

---

## 1. Project layout

The repository root **is** the service. src-layout; the running daemon, any
run-by-hand tooling, and the install/update tooling are kept as distinct tiers.

```
pg-dump-to-s3/
├── README.md               # overview + quickstart
├── CLAUDE.md               # binding rules
├── VERSION                 # SemVer, single source of truth
├── pyproject.toml          # deps + tooling config
├── config.example.ini      # non-secret config key catalog
├── .editorconfig .gitattributes .gitignore
├── docs/
│   ├── CONVENTIONS.md  CONTRACTS.md  ARCHITECTURE.md  DECISIONS.md
│   └── agent-memory/       # committed project memory (+ gitignored local.* )
├── install / install.py    # bash wrapper (100755) + idempotent installer   [later task]
├── update  / update.py     # bash wrapper (100755) + idempotent updater      [later task]
├── pg-dump-to-s3.service   # systemd unit template                           [later task]
├── src/pg_dump_to_s3/      # the daemon (see docs/ARCHITECTURE.md)           [later task]
│   └── runtime/            # reusable runtime layer (stdlib-light)
└── tests/                  # hermetic pytest suite                           [later task]
```

Items marked *[later task]* are specified here and in `docs/ARCHITECTURE.md`;
only the specs and convention files exist in the repo today.

Three tiers, kept separate: the **service** (`src/`), **run-by-hand operator
tools** (`scripts/`, if/when added, reusing the service's own config/secrets
plumbing), and **install/update tooling** (repo root, stdlib-only).

## 2. Language & version

- **Python 3.14+**, asyncio-first. `requires-python = ">=3.14"` in
  `pyproject.toml`; ruff `target-version = "py314"`.
- The one hard invariant: **never block the event loop.** Subprocesses
  (`pg_dump`/`pg_dumpall`/`pg_restore`) run via `asyncio.create_subprocess_exec`;
  blocking library calls (`boto3`, stdlib `tarfile`/`hashlib`) run via
  `asyncio.to_thread`.
- Type hints throughout; `from __future__ import annotations`. Type checking is
  by convention (no mypy config is committed or enforced in CI).
- The version floor is a **constraint only** — never restate an exact interpreter
  version anywhere except the `>=3.14` in `pyproject.toml`.

## 3. Dependencies & packaging

- **setuptools + pip** (not poetry/uv/pipenv). `pyproject.toml` with
  `build-backend = "setuptools.build_meta"`.
- Runtime deps in `[project.dependencies]`; test deps in
  `[project.optional-dependencies].test`.
- **Pin with `>=` lower bounds only — no upper caps, no `==`/`~=`.**
- **Declare what you import, even a transitive** — a transitive that stops being
  transitive would be a runtime `ImportError` on the host.
- **One approved client per job.** Adding an alternative library for a job an
  approved client already covers requires a `docs/DECISIONS.md` entry.
- **Version single-sourced** from `VERSION` via
  `dynamic = ["version"]` + `[tool.setuptools.dynamic] version = { file = "VERSION" }`.
- **src-layout:** `package-dir = { "" = "src" }`, packages found under `src/`.

## 4. Code style, formatting, linting

- **`ruff` only:** `line-length = 100`, `target-version = "py314"`.
- **No `select` is configured**, so ruff runs its **default ruleset only
  (E4/E7/E9/F)**. `E501` (line length), `I` (import order) and `UP` (pyupgrade)
  are **not** enforced — committed code may have 101–103 char lines and legacy
  typing idioms; do not treat those as errors to fix.
- No black, isort, flake8, or pre-commit. A quick dead-code check idiom:
  `python -m ruff check --isolated --select F401,F811,F841 <files>`.
- `.editorconfig` is the cross-editor whitespace/encoding source of truth:
  utf-8, LF, final newline, trim trailing whitespace, 4-space indent; 2-space for
  `{js,css,html,json,yml,yaml,toml}`; `*.md` keeps trailing whitespace;
  `Makefile` uses tabs.

## 5. Configuration & secrets

A flat INI + flat secret files + env overrides. Deliberately **not** `.env`.

- **Config:** a single flat INI at `/etc/pg-dump-to-s3/pg-dump-to-s3.ini`, owned
  by the service user, mode **`0444`** → therefore **no secrets**.
- **Precedence (low→high):** code defaults → INI → environment variables
  `PGDUMPS3_<SECTION>__<KEY>` (uppercase; `__` separates section from key;
  hyphens in a name → underscores).
- **INI interpolation is disabled**, so a value may contain a bare `%`.
- A config file that exists but will not parse raises a config error and the
  process **exits 78 (`EX_CONFIG`)** rather than crash-looping.
- Every key is **optional with an in-code default**; a malformed single value
  falls back to its default (logged), it does not wedge the service.
- **Secrets:** one flat file per secret kind in the same dir,
  `/etc/pg-dump-to-s3/pg-dump-to-s3.<kind>`, mode **`0400`**, owner-only. Values
  are never committed. The installer **enforces owner+mode but never creates the
  value** (asks on the terminal with echo off; writes atomically; never
  world-readable even briefly). Kinds: `db-password`, `s3-access-key-id`,
  `s3-secret-access-key` (role-named, not a bare `password`/`key`).
- `config.example.ini` is committed as the documentation of every key.
- **Retune = redeploy:** only host-tunable operational knobs are config.
  Developer-curated correctness data and anything a run doesn't deliberately
  expose stays a **code constant**.

## 6. Logging

- **Structured logging: one JSON object per line to stdout** (journald captures
  it; `journalctl -u pg-dump-to-s3`). No bespoke log files.
- Fields: `ts` (ISO-8601 UTC), `level`, `logger`, `service`, `msg`, any
  structured `extra`, and `exc` on exceptions. Implemented as a
  `JsonFormatter(logging.Formatter)` installed as the sole root handler;
  `json.dumps(..., default=str, ensure_ascii=False)`.
- Log level from `[service] log_level`; an unknown level never crashes — falls
  back to INFO and warns once.
- **Secret values are never logged.**
- **Verbose results:** every backup run emits a structured per-database line and
  a per-run summary line (status, counts, bytes, duration). See
  `docs/CONTRACTS.md` for the exact fields.
- Error payloads use the envelope
  `{"error":{"code":"...","message":"...","details":{}}}` — `code` stable &
  machine-readable, `message` for humans, `details` optional structured context.

## 7. Database access

- **PostgreSQL via raw `asyncpg`** for enumerating databases — SQL by hand, no
  ORM. The actual dumps are produced by the `pg_dump`/`pg_dumpall` CLI.
- **TLS with full verification by default** (`sslmode=verify-full`): `[postgres]
  host` must be the server certificate's domain, not an IP. `ca_cert` names a
  private CA PEM; **blank for a public CA**. The CA path is public (may live in
  world-readable config); the DB password is always the `db-password` secret.
- CLI credentials are passed to `pg_*` via libpq environment in the subprocess
  (`PGHOST`/`PGPORT`/`PGUSER`/`PGPASSWORD`/`PGSSLMODE`/`PGSSLROOTCERT`), never on
  the command line and never logged.
- **Resilience knobs:** `connect_timeout` (default 15s) and `command_timeout`
  (default 30s) for the enumeration connection; transient connect retry, if any,
  is in code, not config.
- The DB **role is a host prerequisite** — the service never creates/alters the
  role or any database and never sets a password; it only reads.

## 8. Naming & versioning

- Service/unit and host directories: **`pg-dump-to-s3`**. Importable Python
  package: **`pg_dump_to_s3`**. OS user/group: **`pgdumps3`** (hyphen-free for
  `useradd`). Env prefix: **`PGDUMPS3_`**. Secret kinds: `db-password`,
  `s3-access-key-id`, `s3-secret-access-key`.
- **SemVer in `VERSION`, the single source of truth.** Outside `VERSION`, a
  version may appear only as a constraint (`>=3.14`), a dated `DECISIONS.md`
  entry, or migration guidance.
- **Pre-commit version bump:** before a commit that changes code, bump
  `VERSION` — PATCH by default, MINOR when the commit adds a feature (reset PATCH
  to 0), MAJOR only by explicit human decision; staged in the same commit.
  Commits touching only docs/root get no bump. (MAJOR = breaking, MINOR =
  backward-compatible feature, PATCH = fix.)

## 9. Git / commit workflow

- **Text files are LF-only**, enforced by `.gitattributes` (`* text=auto eol=lf`
  plus explicit per-type entries). Binary types marked `binary`, never
  normalized. This overrides a Windows dev box's `core.autocrlf`.
- **Executable bit committed** for extensionless bash wrappers (`install`,
  `update`), mode **`100755`**, so `git pull` yields runnable files on Linux.
  If the dev box has git `core.filemode` off, set it explicitly with
  `git update-index --chmod=+x <file>` and verify with `git ls-files -s`.
- **Prefer committing to `main`; avoid branches.** If branching seems valuable,
  ask first.
- **Always prepare the commit, but never commit without explicit human
  confirmation.** ("Prepare" = stage + propose a message, then wait.)
- **Scoped commits via explicit pathspec** (`git commit -- <paths>`); never
  `git add -A`/`-u`/`git commit -a`.
- `.gitignore` covers secrets, venvs, build artifacts, tool caches, local
  data/logs/scratch, the live (non-example) config, editor/OS cruft, and
  `.claude/settings.local.json`.

## 10. Testing

- **`pytest` + `pytest-asyncio`**, `asyncio_mode = "auto"`,
  `testpaths = ["tests"]`.
- **Hermetic / OS-independent:** the suite reads no host files and runs on any OS
  (including the Windows dev box) without PostgreSQL, without network, and
  without the heavy drivers installed (lazy imports + fakes; e.g. a module that
  needs `asyncpg` or `boto3` imports it lazily so tests can stub it).
- `tests/` mirrors modules (one `test_<module>.py` per unit) plus a smoke test; a
  `conftest.py` puts `src/` and the repo root on `sys.path` so pytest works from
  a fresh checkout.
- Major changes get a max-effort `/code-review` pass before the first commit,
  with a regression test added for each defect found.

## 11. Install / update tooling & idempotency

- **Two deliverables each:** an idempotent `install.py`/`update.py` plus a thin
  **bash wrapper** (`install`/`update`) bootstrapping the `.py` under a
  qualifying interpreter. Admins run `sudo ./install`, never `bash install`.
- **Both are safe to re-run any number of times** — a re-run on an
  already-correct host makes no changes and exits 0.
- Target = two commands on a fresh box: `git clone` into the fixed prod path,
  then `sudo ./install`. Anything the installer can't know is **asked at the
  prompt** (echo off for secrets); it never invents a value.
- `install.py` provisions the already-cloned checkout (**never** `git pull`).
  `update.py` does its own `git pull`, **re-execs once if the pull advanced
  `HEAD`**, reconciles idempotently, and **restarts only if the unit, code, or
  config changed**.
- **Runs as root before the venv exists**, so `install.py`/`update.py` and
  `runtime/provision.py` import **pure stdlib only** (use `subprocess` for
  `systemctl`, `useradd`, etc.).
- **Both refuse to run outside the canonical prod checkout** (`/opt/pg-dump-to-s3`).
- Flags: `--dry-run`, `--verbose`, `allow_abbrev=False`; update adds
  `--no-restart` / `--start`.
- Config reconciliation is **key-by-key, never value-by-value**: keys the
  example adds are merged in (comments/ordering preserved), no value is changed
  and no key removed; the pre-merge file is kept as `*.ini.bak`.

## 12. systemd / deployment / host layout

Unit template conventions (`pg-dump-to-s3.service`):

```ini
[Unit]
After=network-online.target
Wants=network-online.target
StartLimitIntervalSec=60
StartLimitBurst=4            # a broken config parks the unit in `failed`, not a crash-loop

[Service]
Type=notify                 # sd_notify readiness
NotifyAccess=main
User=pgdumps3
Group=pgdumps3
WorkingDirectory=/opt/pg-dump-to-s3
ExecStartPre=/opt/pg-dump-to-s3/.venv/bin/python -m pg_dump_to_s3 --check --config /etc/pg-dump-to-s3/pg-dump-to-s3.ini
ExecStart=/opt/pg-dump-to-s3/.venv/bin/python -m pg_dump_to_s3 --config /etc/pg-dump-to-s3/pg-dump-to-s3.ini
Restart=on-failure
RestartSec=3
RestartPreventExitStatus=78  # EX_CONFIG is permanent — never restart into a bad config
TimeoutStopSec=... (must exceed the longest acceptable in-flight dump drain)
RuntimeDirectory=pg-dump-to-s3
StateDirectory=pg-dump-to-s3
# Hardening:
NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=true
PrivateTmp=true
ReadWritePaths=/var/lib/pg-dump-to-s3

[Install]
WantedBy=multi-user.target
```

- Does **not** set `ConfigurationDirectory=`: the shared `/etc/pg-dump-to-s3/`
  is created root-owned by the installer so it can hold both the config-user's
  file and the `0400` secrets.
- Host filesystem layout (dedicated OS user `pgdumps3`, no login shell): see the
  who-does-what table in `README.md`. `.git` stays root-owned (update's
  `git pull` runs as root).

## 13. Graceful shutdown

- Install asyncio signal handlers for **SIGTERM and SIGINT** that set a shared
  `asyncio.Event` (no heavy work in the handler).
- On stop: stop scheduling new runs, let the in-flight run drain or cancel it,
  close the asyncpg pool and any S3/session resources (each close **bounded and
  isolated** so one stuck close can't overrun `TimeoutStopSec`), exit 0.
- Canonical shape:

  ```python
  async def main() -> None:
      stop = asyncio.Event()
      loop = asyncio.get_running_loop()
      for sig in (signal.SIGTERM, signal.SIGINT):
          loop.add_signal_handler(sig, stop.set)
      async with build_service() as svc:
          await svc.start()
          await stop.wait()
          await svc.shutdown()
  ```

## 14. Documentation

- Documentation is a **deliverable**; update docs in the **same change** as the
  code they describe.
- Doc map: `README.md` (overview + quickstart), `CLAUDE.md` (binding rules),
  `docs/ARCHITECTURE.md`, `docs/CONVENTIONS.md`, `docs/CONTRACTS.md`,
  `docs/DECISIONS.md`.
- **`docs/DECISIONS.md` is a running ADR log**: newest-first, short entries,
  append-only; a superseding decision adds a new entry and marks the old one
  superseded rather than deleting it.
- **Agent-memory split** (`docs/agent-memory/`): only *project-related* memory is
  committed (normal filename, indexed in `MEMORY.md`); personal/global memory
  uses a `local.`-prefixed filename (indexed in `MEMORY.local.md`) and is
  gitignored.
