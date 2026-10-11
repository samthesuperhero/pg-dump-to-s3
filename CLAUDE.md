# CLAUDE.md — binding rules for pg-dump-to-s3

These are the binding rules for anyone working in this repository, human or
agent. They are deliberately short; the reasoning and the full detail live in
`docs/CONVENTIONS.md`, `docs/CONTRACTS.md`, and `docs/ARCHITECTURE.md`. Every
non-trivial choice is recorded in `docs/DECISIONS.md`.

The general engineering conventions here are **inherited from the unrelated
`tgagent` project** as reusable, project-neutral standards. The *only* thing
inherited is those general conventions — never any Telegram/bot domain logic,
names, or data model.

## Rules

1. **Docs are a deliverable.** Update the relevant doc in the *same change* as
   the code it describes. Record every meaningful decision in `docs/DECISIONS.md`
   (newest-first, append-only; supersede, never delete).

2. **Python 3.14+, asyncio-first. Never block the event loop.** All I/O is async.
   Run `pg_dump`/`pg_dumpall`/`pg_restore` via `asyncio.create_subprocess_exec`;
   offload blocking library work (`boto3`, `tarfile`, `hashlib`) with
   `asyncio.to_thread`. The floor `>=3.14` is a constraint, stated only in
   `pyproject.toml` — never restate an exact interpreter version elsewhere.

3. **Dependencies: `>=` lower bounds only** — no upper caps, no `==`/`~=`.
   Declare every import, even a currently-transitive one. **One approved client
   per job**; introducing an alternative library for a job an approved one
   already covers requires a `docs/DECISIONS.md` entry. Approved clients:
   `asyncpg` (DB enumeration), `boto3` (S3), the `pg_*` CLI (dump/verify),
   stdlib `tarfile` (packaging), stdlib `hashlib` (checksums).

4. **Style: `ruff` only** (`line-length = 100`, `target-version = "py314"`,
   default ruleset). E501/import-order/pyupgrade are **not** enforced — don't
   "fix" long-ish lines or legacy typing imports on sight. Whitespace/encoding
   follow `.editorconfig`.

5. **Config is a flat INI**, installed at `/etc/pg-dump-to-s3/pg-dump-to-s3.ini`
   mode `0444`, and **contains no secrets**. Precedence: code defaults → INI →
   env `PGDUMPS3_<SECTION>__<KEY>`. A config file that will not parse exits **78**
   (`EX_CONFIG`). **Retune = redeploy**: only host-operational knobs are config
   (each with an in-code default); correctness data stays a code constant.

6. **Secrets are flat `0400` files**, one per role-named kind, in the config dir
   (`db-password`, `s3-access-key-id`, `s3-secret-access-key`). Never committed,
   never logged, never placed in world-readable config. The installer enforces
   owner+mode and *asks* for each value (echo off) — it never invents one.

7. **Logging: one JSON object per line to stdout** (journald captures it). Fields:
   `ts` (ISO-8601 UTC), `level`, `logger`, `service`, `msg`, structured `extra`,
   and `exc` on exceptions. Unknown log level → INFO (never crash). **Secret
   values are never logged.** Error payloads use the envelope
   `{"error":{"code","message","details"}}`.

8. **SemVer lives in the single `VERSION` file** — nothing else restates the
   current value. Before a commit that changes code, bump `VERSION`: PATCH by
   default, MINOR when it adds a feature (reset PATCH to 0), MAJOR only by
   explicit human decision. Docs-only / root-only commits get no bump.

9. **Git.** Text is LF-only (`.gitattributes`). Prefer committing to `main`;
   ask before branching. **Always prepare the commit (stage + propose a message)
   but never commit without explicit human confirmation.** Stage via explicit
   pathspec (`git commit -- <paths>`); never `git add -A`/`-u`/`commit -a`.
   Extensionless bash wrappers are committed `100755`.

10. **Install/update are idempotent** (`install.py`/`update.py` + thin bash
    wrappers). They run as root *before the venv exists*, so their import chain
    (and `runtime/provision.py`) is **pure stdlib only**. They refuse to run
    outside the canonical prod checkout. `install` provisions an already-cloned
    tree (no `git pull`); `update` pulls, re-execs once if `HEAD` advanced, then
    reconciles and restarts only if something changed.

11. **systemd `Type=notify`** with `sd_notify` readiness. `ExecStartPre` runs
    `--check` and a bad config exits 78; `RestartPreventExitStatus=78` so a bad
    config parks the unit in `failed` instead of crash-looping. **Graceful
    shutdown:** SIGTERM/SIGINT set an `asyncio.Event`; stop new work, let the
    in-flight run drain, close pools (each close bounded/isolated), exit 0 within
    `TimeoutStopSec`.

12. **Tests are hermetic and OS-independent** (`pytest` + `pytest-asyncio`,
    `asyncio_mode = "auto"`). They run on the Windows dev box with no PostgreSQL,
    no network, and without the heavy drivers installed (lazy imports + fakes).

13. **Never introduce overlapping backup runs.** If a run is still active when
    the next trigger fires, skip the new trigger and warn.
