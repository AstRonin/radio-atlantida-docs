# Plan — Alternative Host: Docker on Linux or macOS (radio-atlantida-deploy)

Status: Approved (2026-11-14); not started. Starts after the first production install on Windows.
Based on: `docs/deploy-init-plan.md` (D1–D7, 6.11), `docs/runbook.md`, `docs/dry-run.md`, `docs/tech-design.md` v7-28 §9.
Last updated: 2026-11-14

## 1. Goal

Windows 10/11 with Docker Engine in WSL2 stays the **primary** production host (`deploy-init-plan.md`). This plan adds an **alternative**: the same stack on a Linux machine or a Mac, if the station ever needs it, and on the way splits the deploy repository into:

- a **common core** that runs the same everywhere (all application and infrastructure containers, `radioctl`'s logic, backups, restore drill, boot check, update, rollback, maintenance, cut-over), and
- a **thin host layer** per platform: what is needed only to get Docker running, start it at boot, and call the core (`install.bat`/`update.bat` on Windows; `install.sh`/`update.sh` on Linux and macOS).

Everything stays in Docker; nothing of the application runs natively on the host. The shape is chosen so a later move to Kubernetes maps one to one (§8).

## 2. Decisions (2026-11-14)

1. **Apps in Docker on every host.** The host provides only a Docker engine. Linux: Docker Engine. macOS: **Colima** (free, headless, started by launchd; Docker Desktop and OrbStack were considered: Desktop needs a signed-in user, OrbStack is paid for commercial use).
2. **The core moves into an ops container.** `radioctl` and its tools (bash 5, jq, git, coreutils, curl, Docker CLI and Compose plugin) run in a pinned image `radio-atlantida-ops`, started with the host's Docker socket. The host needs no bash version, jq, or GNU tools of its own (macOS ships bash 3.2 and BSD tools).
3. **Scheduled jobs move into the stack.** A `scheduler` compose service (the ops image with supercronic) runs `radioctl backup` daily at 04:30 and `radioctl restore-drill` at 05:30 on the 1st, in `Asia/Yerevan` (as the systemd timers do now), and `radioctl boot-check` once when it starts. This replaces the systemd timers on Linux and macOS; in Kubernetes it becomes CronJobs.
4. **Linux first**, then macOS. Linux can be tested end to end in CI (§7); macOS only by a manual dry run.
5. **Windows unchanged by this plan.** U0 refactors without changing behaviour (bats and Pester stay green). Moving Windows onto the ops container and scheduler too is a possible later step (§9), not part of this plan.
6. **Linux distributions.** The installer installs Docker Engine (pinned in `install/versions.env`) on Ubuntu 24.04 LTS and Debian 12/13; on any other distribution it requires Docker Engine ≥ the pinned version to be installed already and checks it (confirmed 2026-11-14).
7. **Database location per platform.** Linux: the bind-mounted directory `/srv/radio/pgdata` (6.11). macOS: a named volume inside Colima's VM, because a bind mount from the macOS file system goes through virtiofs, which is slow and unreliable for PostgreSQL's `fsync` and permissions; the dumps are copied to a folder on the Mac (the counterpart of the Windows copy).

## 3. What is common and what is per host

| Part | Common core | Windows (primary, unchanged) | Linux | macOS (Colima) |
|---|---|---|---|---|
| Containers (`docker-compose.yml`) | all services, pins, health checks | — | — | — |
| `radioctl` logic | update, rollback, restore, maintenance, cut-over, status, backup, restore drill, boot check | runs in WSL bash (as now) | runs in the ops container | runs in the ops container |
| Platform facts (`bin/lib/platform/*.sh`) | interface | `wsl.sh` | `linux.sh` | `macos.sh` |
| Docker engine | — | Docker Engine in WSL (`install.ps1`, `setup-distro.sh`) | Docker Engine (`install.sh`) | Colima (`install.sh`) |
| Start at boot | `restart: unless-stopped` | Task Scheduler + keep-alive | `systemctl enable docker` | launchd daemon starts Colima |
| Schedules | backup, restore drill, boot check | systemd timers in WSL (as now) | `scheduler` service | `scheduler` service |
| Database | PostgreSQL 17 | `/srv/radio/pgdata` in WSL (6.11) | `/srv/radio/pgdata` | named volume in Colima's VM |
| Second backup copy | `BACKUP_COPY_DIR` (renamed from `BACKUP_WINDOWS_COPY`, old name still read) | `C:\RadioAtlantida\backups` | a path on another disk, or empty | a folder on the Mac |
| Media | `MEDIA_ROOT` | `D:\…` via `/mnt/d` | any host folder | a folder under a Colima mount |
| Live voice UDP | `LIVE_UDP_BIND`, `LIVE_PUBLIC_IPS` | NAT or mirrored (`radioctl network`) | `127.0.0.1` | `127.0.0.1` through Colima's port forward (to verify, U4) |
| Entry points | — | `install.bat`, `update.bat` | `install.sh`, `update.sh` | `install.sh`, `update.sh` |
| Host hardening | — | `install.ps1` (sleep, updates, active hours) | none required (a server distribution) | `pmset`: no sleep, `autorestart 1` after a power loss; auto login of a dedicated user is not needed with launchd |

## 4. The ops container

- **Image.** `ops/Dockerfile` in this repository (a deliberate exception to "no build steps": it is tooling, not application code), Debian slim + bash, jq, git, curl, tzdata, Docker CLI and Compose plugin at the versions pinned in `install/versions.env`, supercronic. Built for `linux/amd64` and `linux/arm64` by this repository's CI on a tag `ops-vX.Y.Z`, pushed to `ghcr.io/astronin/radio-atlantida-ops`, and pinned by digest in `ops.env` (like `release.env`). The image holds tools only; the deploy repository stays a checkout on the host, so `git pull` keeps working as the update source of `radioctl`.
- **Same paths inside and outside.** Compose resolves bind-mount sources on the host, while `radioctl` reads the same files from inside the container. So every host path the stack uses is mounted at the **same path** in the ops container: the deploy checkout (`/opt/radio-atlantida/deploy`), `/srv/radio` (journal, backups, pgdata on Linux), `MEDIA_ROOT` (read-only), and `BACKUP_COPY_DIR`. On macOS these paths are under a Colima mount (the default mounts the user's home; the installer adds the deploy and data folders explicitly).
- **Docker socket.** `/var/run/docker.sock` is mounted into the ops container and the `scheduler`. Access to the socket is root on the host; acceptable here because the stack is local only, nothing in the ops container listens on a port, and the image is pinned by digest. Written down in the runbook.
- **Typed confirmations.** `update.sh` runs the container with `-it` when a terminal is attached, so `CUTOVER`, `MAINTENANCE`, `RESTORE` work as on Windows; the `scheduler` runs only non-interactive commands.
- **Identity and files.** The container runs as the host's `radio` user id (Linux) or the installing user (macOS), so files in the checkout and in `/srv/radio` keep their owners; the Docker group id is passed for the socket.

## 5. Phases

### U0 — Platform layer, no behaviour change

1. Collect every host-specific spot in `bin/radioctl` (about a dozen: the Windows copy and `/mnt/c` checks, `win_path`, `network` NAT/mirrored, keep-alive log, boot-check uptime source, `GRAFANA_URL` derivation, free-disk check) behind small functions in `bin/lib/platform/{wsl,linux,macos}.sh`, chosen by `RADIOCTL_PLATFORM` (`wsl` default when `/proc/sys/fs/binfmt_misc/WSLInterop` exists, else from the ops container's environment).
2. Rename `BACKUP_WINDOWS_COPY` to `BACKUP_COPY_DIR`, keep reading the old name.
3. `setup-distro.sh` keeps the WSL setup; its generic part (user `radio`, `/srv/radio/*`, Docker Engine install) becomes `install/lib/host-setup.sh`, used by `setup-distro.sh` and later by `install.sh` on Linux.
4. Tests: bats run every platform-sensitive test under `wsl`, `linux`, `macos`; Pester unchanged; shellcheck.

**Done when:** all existing tests pass unchanged, the Windows dry-run path is unchanged, and the new platform tests pass.

### U1 — Ops image

1. `ops/Dockerfile`, `ops.env`, CI job: build both architectures, smoke (`radioctl --help`, `docker compose version`, `jq`, `git`, bash ≥ 5), push on `ops-v*` tags; `scripts/check-versions.sh` covers its pins.
2. `bin/radioctl-ops` (host side, POSIX sh): builds the `docker run` line (same-path mounts, socket, user and group ids, `-it` when a terminal is attached, `RADIOCTL_PLATFORM`, `TZ`) and runs `radioctl` inside.
3. `radioctl` inside the container: no other change than U0's platform functions.

**Done when:** on a Linux machine with only Docker installed, `bin/radioctl-ops status` and every read-only command work.

### U2 — Scheduler service

1. Compose service `scheduler` (profile `scheduler`, enabled by the Linux and macOS installers; the Windows stack keeps its systemd timers and does not enable the profile): the ops image, supercronic with a crontab in `config/scheduler/crontab`, `TZ=Asia/Yerevan`, same mounts as §4, `restart: unless-stopped`, runs `radioctl boot-check` once at start.
2. `radioctl status` reads the last backup and drill results the same way for both sources (they already write to `/srv/radio/backups` and the state file).
3. Missed runs: supercronic does not catch up; at start the scheduler runs a backup if the last daily backup is older than 26 h (the systemd timers' `Persistent=true` counterpart).

**Done when:** in a Linux test environment the backup runs at the configured time, a stopped-then-started scheduler catches up a missed backup, and a restore drill runs.

### U3 — Linux installer and update script

1. `install.sh` (POSIX sh, root via sudo): checks the distribution and Docker (installs it on Ubuntu 24.04 / Debian 12–13 from the pinned versions, else requires it), creates user `radio`, `/srv/radio/{journal,backups,pgdata}` (6.11 rules), `/opt/radio-atlantida/deploy` (a clone of this repository, by the ops image's git), asks for the media folder and the second backup folder, `systemctl enable --now docker`, then `radioctl first-run` through `radioctl-ops` with the scheduler profile on. Idempotent; a second run repairs.
2. `update.sh [command|tag]`: the same arguments as `update.bat` (a bare tag means `update`), runs `bin/radioctl-ops` as `radio`.
3. Live voice: `LIVE_UDP_BIND=127.0.0.1`, `LIVE_PUBLIC_IPS=127.0.0.1` (no NAT layer); `radioctl network` is a no-op with a message on Linux.
4. CI job `linux-install` on `ubuntu-24.04`: run `install.sh` non-interactively against locally built or pinned images and a test Icecast (as `tests/liquidsoap`), wait for the stack, run `update.sh status`, a backup, a restore drill, an `update` to the same release, and a `rollback`. This is the first full install test in CI.

**Done when:** the CI job passes and a manual dry run in a fresh Linux VM (OrbStack, a new VM next to `radio-dryrun`) passes the rehearsal of `dry-run.md` §6 that applies (no Windows steps), including a VM reboot.

### U4 — macOS with Colima

1. `install.sh` on macOS: checks Apple Silicon or Intel and free disk, installs Colima and the Docker CLI at pinned versions (Homebrew if present, else the release binaries), writes Colima's profile `radio-atlantida` (vz, CPUs and memory as the WSL rule: half the RAM, 4–16 GB; disk size; mounts: the deploy folder, the data folder, the media folder, the backup copy folder), a launchd daemon `net.atlantida.radio.colima` that starts the profile at boot as the installing user, `pmset` (no system sleep, `autorestart 1`), then the same core steps as Linux through `radioctl-ops`.
2. Database: the compose file selects the named volume on macOS (a small override file written by the installer, `runtime/host.override.yml`); the 6.11 guards run only on Linux and WSL.
3. Live voice: verify that Colima forwards the published `127.0.0.1:8189/udp` to the Mac (the dev stack already works this way with Docker Desktop); if it does not, use Colima's network address mode and the NAT path of `radioctl network` (VM address in `LIVE_PUBLIC_IPS`, bind `0.0.0.0`).
4. `update.sh` is the same file as on Linux.
5. CI: shellcheck, a launchd plist lint, and a Colima profile check only (GitHub's macOS runners have no nested virtualisation).

**Done when:** a manual dry run on a Mac passes: install, cold boot without signing in (stack up, on air), a power cut (`autorestart`), update, rollback, maintenance, cut-over, a live-voice check from the Mac's browser, backup and its copy in the Mac folder, restore drill.

### U5 — Documentation

Runbook: a "Host" chapter per platform (install, start at boot, where data lives, what must never be deleted: `/srv/radio/pgdata` on Linux, the Colima profile `radio-atlantida` on macOS — `colima delete` deletes the database like `wsl --unregister`), the shared chapters unchanged. `README.md`: the three platforms with Windows as the primary. `dry-run.md`: Linux and macOS sections. `deploy-init-plan.md`: a pointer to this plan.

## 6. Safety rules carried over

- Never run `docker compose down -v`, `docker volume prune`, `colima delete`, or remove `/srv/radio` (runbook "Never" box per platform).
- Updates never restart `db` or `liquidsoap` outside a maintenance window; the scheduler never runs `update`, `maintenance`, `cutover`, or `restore`.
- Secrets only in `.env` in the deploy folder (mode 0600, owner `radio` or the installing user); the ops container reads it through the same-path mount and never prints it.

## 7. Testing summary

| Level | Windows | Linux | macOS |
|---|---|---|---|
| Unit (bats, platform matrix) | yes | yes | yes |
| Pester | yes | — | — |
| CI end-to-end install | no (Windows runner has no WSL2 nesting) | **yes** (`linux-install`) | no |
| Manual dry run | Mac OrbStack VM (`dry-run.md`) + Windows acceptance | OrbStack Linux VM | a real Mac |

## 8. Kubernetes later

The split maps directly: application services → Deployments/StatefulSet (PostgreSQL), `scheduler` jobs → CronJobs, `radioctl` steps → Jobs run from the ops image, `.env` → Secrets, `config/` → ConfigMaps, the bind mounts → PersistentVolumes. The rules of this plan that keep that open: no logic on the host, every scheduled job is a container, configuration only in env files and `config/`.

## 9. Open points and later

1. Decision 6 and the Docker socket in the ops container and the scheduler (§4) confirmed by the developer on 2026-11-14.
2. Later, not in this plan: Windows on the ops container and scheduler too (would remove the systemd units and most of `setup-distro.sh`), only after U3 has run in production-like use.
3. The macOS live-voice path is verified in U4 step 3 before the rest of U4 is built.
