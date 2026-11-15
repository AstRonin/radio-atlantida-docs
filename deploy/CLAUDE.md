# CLAUDE.md — radio-atlantida-deploy

## Project context

Radio Atlantida is a radio automation system (Go backend, Angular frontend). This repository is its production deployment: the Compose stack, configuration copies, the host installers, and the update tool `radioctl`. **No application code**: the backend and frontend run as prebuilt multi-arch images from GHCR (`ghcr.io/astronin/radio-atlantida-{backend,frontend}`), referenced by tag and digest.

- **Hosts** (`docs/unix-host-plan.md`):
  - **Windows 10/11, the primary host**: Docker Engine in a dedicated WSL2 distribution `radio-atlantida` (pinned Ubuntu 24.04, systemd), no Docker Desktop. `install.bat` → `install.ps1` (Windows half) → `install/setup-distro.sh` (Linux half, generic part in `install/lib/host-setup.sh`). `update.bat` runs `bin/radioctl` inside the distribution. Schedules are systemd timers in the distribution. Runbook chapters 1–11.
  - **Linux** (Ubuntu 24.04, Debian 12/13; other distributions with Docker at least the pin): `install.sh` (POSIX sh, asks for sudo) → `install/install-linux.sh`; `update.sh` runs `bin/radioctl-ops` as user `radio`. Runbook chapter 12.
  - **macOS 13+** with Colima: `install.sh` → `install/install-macos.sh` as the installing user (never root), Colima profile `radio-atlantida`, launchd daemon `net.atlantida.radio.colima`, `pmset`. `update.sh` runs as the checkout's owner. Implemented and tested with mocks only; the real-Mac dry run (`docs/dry-run.md` §9) and the live-voice check (runbook 13.6) are still open. Runbook chapter 13.
  - On Linux and macOS, `radioctl` runs in the ops container (`bin/radioctl-ops`, image built from `ops/`, pinned in `ops.env`; while `ops.env` is empty the local build `radio-atlantida-ops:local` is used). Scheduled jobs are the compose service `scheduler` (profile `scheduler`, supercronic, `config/scheduler/crontab`): backup 04:30 daily, restore drill 05:30 on the 1st, `Asia/Yerevan`.
- **Images**: `release.env` pins the station release (`BACKEND_TAG/DIGEST`, `FRONTEND_TAG/DIGEST`); a station release is a PR that changes it. Infrastructure images are pinned literally in `docker-compose.yml`, equal to the backend dev stack (checked in CI). No `latest` anywhere. Host tools are pinned in `install/versions.env` (Windows/Linux), `install/versions-macos.env`, and `ops/versions.env`.
- **Data**: media stay in a host folder (`MEDIA_ROOT`), never in an image, a container, or a named volume. The database is the bind-mounted directory `/srv/radio/pgdata` on WSL and Linux, and the external named volume `radio-atlantida-db` in Colima's VM on macOS (`runtime/host.override.yml`). Backups under `BACKUP_ROOT` with a second copy in `BACKUP_COPY_DIR` (old name `BACKUP_WINDOWS_COPY` still read). macOS uses `/opt/radio-atlantida/{deploy,data,logs}` (`/srv` cannot be created there).
- **Exposure**: the panel has no authentication (tech-design §1), so the panel and Grafana are published on `127.0.0.1` only (no variable can widen them); the live-voice UDP port is bound to `LIVE_UDP_BIND`, written only by `radioctl network`. Nothing else is published, no listener port. There is no Icecast container: production streams to an external Icecast server configured in the panel (`stream.*` settings); only the source password is in `.env`.

## Required reading at the start of every session

1. `README.md` — layout, CI jobs and how to run them locally, config sync with the backend, how a station release is made, the `radioctl` test switches.
2. `docs/deploy-init-plan.md` — the deploy plan (ground rules §1, conventions §1.1, update steps §7.3, migrations and the downgrade guard §10.3–10.4, decisions §11).
3. `docs/unix-host-plan.md` — the Linux and macOS hosts, the ops container, the scheduler, safety rules (§6), and what is still unverified (5.3).
4. `docs/runbook.md` — the operator runbook (behaviour as the operator sees it; the "Never" boxes).
5. `docs/dry-run.md` — the rehearsal of a release on the developer's Mac.
6. `docs/tech-design.md` §9 (the authority on how the system is built) and `docs/base.md` (requirements), as needed.

**If the code, the design, and the docs disagree, stop and ask.** Do not work around a cross-repo task (deploy-init-plan §12) in this repository.

## Commands

Every CI job calls a script of this repository, so it runs the same way locally. All need only Docker unless noted.

| Command | What it does (CI job) |
|---------|-----------------------|
| `scripts/check-line-endings.sh` | `.gitattributes` eol per file, LF in the index, BOM on `*.ps1` only, no case-only duplicates (`lint`) |
| `scripts/check-shell.sh` | shellcheck 0.10.0 (pinned image) on every tracked sh/bash file, shebang files included (`lint`) |
| `docker run --rm -v "$PWD:/repo:ro" -w /repo rhysd/actionlint:1.7.7 -color` | actionlint of the workflows (`lint`; CI pins the image by digest) |
| `scripts/check-compose.sh` | renders the stack with every fixture and profile and checks its rules (ports, pins, mounts, health checks) (`compose`) |
| `scripts/check-config.sh` | Loki, Alloy, Grafana configuration in the pinned images (`compose`) |
| `scripts/check-release.sh` | `release.env` tags and digests (`release`) |
| `scripts/check-backend-sync.sh [--ref REF] [--backend-dir DIR] [--update]` | `config/` equals the backend's `dev/` at the backend ref; `--update` rewrites it (`backend-sync`) |
| `tests/liquidsoap/run.sh [--ref REF] [--backend-dir DIR] [--check-only]` | `liquidsoap --check`, then the backend's runtime smoke test (needs Go) (`liquidsoap`) |
| `tests/radioctl/run.sh [-f NAME]` | bats tests of `radioctl`, `radioctl-ops`, `update.sh`, the scheduler, the platform matrix, and `install-macos.sh` under bash 3.2 with stubs (`radioctl`) |
| `tests/distro/run.sh [--with-apt]` | `setup-distro.sh` twice in `ubuntu:24.04`, `systemd-analyze verify`; the pinned apt install (`distro`) |
| `scripts/check-versions.sh [--download]` | every pin of `install/versions.env`, `ops/versions.env`, `install/versions-macos.env` exists upstream and hashes to its pin; `ops.env` format (`versions`) |
| `tests/install/run.sh [--pester]` | Pester 5.7.1 tests of `install.ps1` and PSScriptAnalyzer 1.25.0 in PowerShell 7.5 (`install`) |
| `tests\install\run-windows.ps1` | the same under Windows PowerShell 5.1 (`windows`, Windows only) |
| `scripts/check-macos.sh` | macOS only, read-only: bash 3.2 and sh parse, `plutil -lint` of the plist, Colima profile keys, host override (`macos`) |
| `ops/build.sh --platform P -t TAG`, `ops/smoke.sh TAG P` | build and smoke test the ops image (`ops`) |
| `tests/linux-install/container.sh --backend-dir DIR --frontend-dir DIR` | the full Linux install test in a throwaway privileged container (CI runs `sudo tests/linux-install/run.sh --throwaway ...` on the runner). **Never run `run.sh` on a real machine**: it installs the station on the host |

Workflows: `.github/workflows/ci.yml` (every push to `main` and every PR), `.github/workflows/ops-release.yml` (tag `ops-vX.Y.Z` publishes the ops image; a PR then sets `ops.env`).

## Rules for changes

**Stack and update tool**

- The update tool is the only way to change the production stack; manual `docker compose` is for diagnostics. A normal update never restarts `db` or `liquidsoap`; that happens only in a maintenance window (`maintenance`, typed `MAINTENANCE`). Stream output settings apply at runtime, no restart.
- Migrations are forward-only: never `migrate down`. No install below 1.0.0. An older target replaces only the images and is refused below `schema_compat.min_app_version` (read with `psql`), offering the restore path; frontend and backend must share MAJOR.MINOR, and the backend must meet the Liquidsoap script's `min-backend` (deploy-init-plan §10.4, §7.3).
- Destructive steps always need the typed word (`RESTORE`, `MAINTENANCE`, `CUTOVER`); every other question has a `--yes`-style answer for tests. The scheduler runs only `backup`, `restore-drill`, `boot-check`, `status`, `logs`, never `update`, `maintenance`, `cutover`, or `restore`.
- Never `docker compose down -v`, `docker volume prune`, `colima delete`, `wsl --unregister`, or removal of `/srv/radio` (`pgdata`) or `/opt/radio-atlantida`, and never write a script step that does.
- `config/` (Liquidsoap script, Grafana, Loki, Alloy) is never edited here: change the backend, release, then `scripts/check-backend-sync.sh --update`. `config/stream-defaults.json` is this repository's own.
- Installers are idempotent: a second run changes nothing, never restarts a running VM, never reloads an unchanged unit or daemon. On the media folder, a new folder gets group 10001 with group write without a question; only an existing folder whose files lack it is asked about.

**Secrets**

- `.env`, `state.json`, `state.lock`, `runtime/`, `logs/`, `backups/`, `*.dump` are git-ignored and never committed. `.env.example` lists every variable with a placeholder and comment.
- Never print, log, or copy secret values (name the missing variable, never its value). `.env` is parsed, never `source`d. The GHCR token goes to `docker login --password-stdin`.

**Shell**

- `bin/radioctl` and scripts that run in the distribution or the ops image: bash 5, `set -Eeuo pipefail`, shellcheck clean, `"$var"` quoting, small functions named as verbs; external command output to the run log.
- `install.sh`, `update.sh`, `bin/radioctl-ops`: POSIX sh (`set -eu`); `ops.bats` runs `radioctl-ops` under busybox sh.
- Anything that runs on the Mac host (`install/install-macos.sh`, `ops/build.sh`, and the sh files above): **bash 3.2 and BSD tools**. No `set -E` (bash 3.2 runs an inherited ERR trap in every failing command substitution), no bash 4+ features, no GNU-only forms (`sed -i` without a suffix argument, `date -d`, `readlink -f`, ...). `scripts/check-macos.sh` and `macos.bats` check this.
- On macOS the station addresses Colima by its socket (`DOCKER_HOST`), never by switching the user's current Docker context.

**PowerShell** (`install.ps1`, wrappers)

- Windows PowerShell 5.1; PowerShell 7 must not be required. `Set-StrictMode -Version Latest`, `$ErrorActionPreference = 'Stop'`, `$LASTEXITCODE` checked, `WSL_UTF8=1` before `wsl.exe`, `Join-Path`, no aliases. PSScriptAnalyzer clean (suppressions only with a comment).

**Files and versions**

- LF everywhere except `*.bat` and `*.ps1` (CRLF; `*.ps1` UTF-8 with BOM), per `.gitattributes` and `.editorconfig`.
- A pin change goes through `scripts/check-versions.sh`; the macOS CLI, Compose, and buildx versions must equal the `install/versions.env` pins. With Homebrew, the macOS pins are minimums (`brew pin`).
- When behaviour changes, update `README.md`, `docs/runbook.md` (operator-visible behaviour), and `docs/dry-run.md` (rehearsal steps) in the same change, and the plan's status or deviation lines if the change implements or departs from a plan.
- A change to `bin/radioctl`, `install/`, or `docker-compose.yml` is rehearsed in the dry run before it reaches the station (`docs/dry-run.md`).

## Docs

`docs/` is synced with the central repository `../radio-atlantida-docs` by its `docs` agent; do not copy files there yourself, tell the developer a document changed.

- `base.md`, `tech-design.md`: shared, canonical in the backend/frontend; pushed here.
- `deploy-init-plan.md`: canonical here; pushed to backend and frontend.
- `backend-init-plan.md`, `frontend-init-plan.md`: copies kept here intentionally.
- `unix-host-plan.md`, `runbook.md`, `dry-run.md`: this repository's own documents (also kept centrally).
- Implemented plans are archived in `../radio-atlantida-docs/archive/` (`backend/`, `frontend/`); references point there. Do not restore them. Code and `tech-design.md` are the source of truth; archived plans are history.

## Git

- Commit locally only when the developer asks. Never push, never tag (no `ops-v*` tags), never change GitHub settings: the developer pushes and checks CI.

## Dry-run hosts and the developer's machine

- OrbStack machines on the developer's Mac: `radio-dryrun` (Windows/WSL-path rehearsal, `docs/dry-run.md` §1–6; keep it running; images in its local registry `127.0.0.1:5000`) and `radio-linux` (Linux-host rehearsal, §8; stopped). Never delete or recreate either one without asking, even though `docs/dry-run.md` §7 describes cleanup.
- Never touch the dev stack (compose project `radio-atlantida-dev`) or the Mac's current Docker context. The Liquidsoap smoke test uses its own project `radio-deploy-liqsmoke` and removes it.
- Do not install Colima on the developer's Mac: the macOS dry run needs a separate Mac (`docs/dry-run.md` §9).
- Do not run `docker push`, `docker compose up` against a live environment, or anything that changes a running station unless the developer asks.
