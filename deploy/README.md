# radio-atlantida-deploy

Production stack of Radio Atlantida: the Compose file, configuration copies,
the Windows host installer, and the update tool. No application code; the
backend and frontend run as prebuilt images from GitHub Container Registry.

- **Operators:** [`docs/runbook.md`](docs/runbook.md) (install, daily
  operation, updates, maintenance, cut-over, backups, restore,
  troubleshooting, the Windows acceptance checklist).
- **Developers:** this file, [`docs/dry-run.md`](docs/dry-run.md) (the Mac
  rehearsal of a release), `docs/deploy-init-plan.md` (the plan), and
  `docs/tech-design.md` §9 (the design).

## Layout

| Path | What |
|------|------|
| `docker-compose.yml` | the production stack; infrastructure images pinned literally, application images by tag and digest from `runtime/applied.env` |
| `release.env` | the release this repository installs (`BACKEND_TAG`, `BACKEND_DIGEST`, `FRONTEND_TAG`, `FRONTEND_DIGEST`); a station release is a PR that changes it |
| `.env.example` | the template of the station's `.env` (written by `radioctl first-run`, never committed) |
| `config/` | copies of the backend's `dev/` files at the release: `liquidsoap/radio.liq` (+ `production.patch`), `grafana/`, `loki/`, `alloy/`; `backend-source.env` records the backend commit. Also this repository's own `stream-defaults.json`: the stream settings `radioctl first-run` proposes (the current Icecast server; the panel settings are the truth afterwards) |
| `install.bat`, `install.ps1` | the Windows half of the host install (Windows PowerShell 5.1) |
| `install/` | the Linux half: `setup-distro.sh` (its generic part in `lib/host-setup.sh`), `wsl.conf`, `radio-keepalive`, the systemd units, `versions.env` (pinned Ubuntu rootfs, Docker Engine, Compose, buildx, WSL) |
| `update.bat`, `bin/radioctl` | the update tool (bash, inside the WSL distribution); the host facts per platform in `bin/lib/platform/{wsl,linux,macos}.sh` (`RADIOCTL_PLATFORM`, default `wsl`) |
| `bin/radioctl-ops`, `ops/`, `ops.env` | the alternative Linux/macOS host (`docs/unix-host-plan.md`): `radioctl` in the ops container (POSIX sh wrapper; it also writes `runtime/scheduler.yml`, the host part of the scheduler service); the image's `Dockerfile`, `versions.env` (base image, supercronic), `build.sh`, `smoke.sh`; `ops.env` pins the published image (empty until the first `ops-vX.Y.Z`) |
| `install.sh`, `update.sh`, `install/install-linux.sh` | the Linux host install and entry point (U3): `install.sh` (POSIX sh, asks for sudo) runs `install/install-linux.sh` (Docker Engine at the pins on Ubuntu 24.04 and Debian 12/13, user `radio`, `/srv/radio`, the clone, `radioctl first-run`); `update.sh` takes the arguments of `update.bat` and runs `bin/radioctl-ops` as `radio` |
| `bin/radioctl-scheduler`, `config/scheduler/crontab` | the scheduler service of Linux and macOS hosts (U2, compose profile `scheduler`): supercronic runs `radioctl backup` at 04:30 and `restore-drill` at 05:30 on the 1st (Asia/Yerevan); at its start the boot check (after a boot) and a missed backup |
| `scripts/` | the CI checks (below) and the shared bash libraries |
| `tests/` | bats (`radioctl`), Pester (`install.ps1`), the distro test, the Liquidsoap smoke test, the Linux install test (`linux-install/`), fixtures, the dry-run compose override |
| `docs/` | the runbook and the dry run (maintained here), and the plans and design (copies pushed from `radio-atlantida-docs`) |

Runtime files (never committed), inside the distribution under
`/opt/radio-atlantida/deploy`: `.env`, `state.json`, `state.lock`,
`runtime/` (`applied.env`, `target.env`, `network.env`), `logs/`. The
database is the directory `/srv/radio/pgdata` (a bind mount, not a Docker
volume). Backups are under `/srv/radio/backups` with a copy in
`C:\RadioAtlantida\backups` (`BACKUP_COPY_DIR`, formerly `BACKUP_WINDOWS_COPY`,
which is still read). A Linux host has the same paths on the machine itself,
plus `runtime/scheduler.yml`; its second copy is a folder on another disk,
or none.

Line endings: LF everywhere except `*.bat` and `*.ps1` (CRLF; `*.ps1` also
UTF-8 with BOM), set by `.gitattributes` and checked in CI.

## Tests and CI

`.github/workflows/ci.yml` runs on every push to `main` and every pull
request. Every job calls scripts of this repository, so each one runs the
same way on the Mac (bash, Docker; the Liquidsoap job also needs Go).

| Job (runner) | Command | Checks |
|--------------|---------|--------|
| `lint` (ubuntu) | `scripts/check-line-endings.sh` | `.gitattributes` eol per file (CRLF `*.bat`/`*.ps1`, LF elsewhere), LF in the index, the working tree matches, UTF-8 BOM on `*.ps1` only, no case-only path duplicates |
| | `scripts/check-shell.sh` | shellcheck 0.10.0 (pinned image) on every tracked shell file: `*.sh`, `*.bash`, and every file with a bash or sh shebang (`bin/radioctl`, `bin/radioctl-ops`, `install/radio-keepalive`, the test stubs) |
| | actionlint 1.7.7 (pinned image) | this workflow |
| `compose` (ubuntu) | `scripts/check-compose.sh` | renders with `.env.example` and the NAT and mirrored fixtures, with and without `--profile monitoring`, and once with `tests/fixtures/scheduler.yml` and `--profile scheduler` (the scheduler: no network, no ports, read-only, init, `TZ=Asia/Yerevan`, `RADIOCTL_SCHEDULER=1`, the other services unchanged; never rendered without its profile): no `icecast`; application images are the release references, all others literal `name:tag`, no `latest` anywhere (compose files, Dockerfiles, script image variables); published ports only panel and Grafana on `127.0.0.1` and the UDP port on `LIVE_UDP_BIND`; media mounts are binds with `create_host_path` not true (Compose drops a false value), backend read-write, liquidsoap read-only; journal read-only in backend; one-shot jobs disable the health check; backend grace period 30 s and group 10001 |
| | `scripts/check-config.sh` | `loki -verify-config`, `alloy validate`, `alloy fmt` equal to the file (the pinned images, as the backend CI); Grafana provisioning YAML parses, dashboard JSON parses, every data source uid a dashboard uses is provisioned |
| `release` (ubuntu) | `scripts/check-release.sh` | tags X.Y.Z, at least 1.0.0, same MAJOR.MINOR; digests set once the backend tag exists (a warning before the first release); with digests, both Git tags exist; with the `GHCR_USER` variable and `GHCR_READ_TOKEN` secret, each digest is the tag's manifest list digest in GHCR |
| `backend-sync` (ubuntu) | `scripts/check-backend-sync.sh` | `config/` equals the backend's `dev/` at the backend ref (the script with `production.patch`), the contract header, the image pins equal `docker-compose.dev.yml` and `dev/liqsmoke/compose.yml` |
| `liquidsoap` (ubuntu, Go from the backend's `go.mod`) | `tests/liquidsoap/run.sh` | `liquidsoap --check` in the pinned image, then the backend's smoke tool (`dev/liqsmoke`) at the backend ref: every codec x channels x sample rate, the encoder limits, rapid output switches (4b), TLS, restart from the state file, the crash-loop guard (7) |
| `radioctl` (ubuntu) | `tests/radioctl/run.sh` | the bats tests of `bin/radioctl` (bats-core 1.11.1; docker, git, the clock stubbed), the platform matrix (`platform.bats`: wsl, linux, macos), the scheduler (`scheduler.bats`: compose arguments, start and refresh, catch-up, refused commands, `bin/radioctl-scheduler`), and the docker run line and `runtime/scheduler.yml` of `bin/radioctl-ops` and `update.sh` (`ops.bats`) |
| `distro` (ubuntu) | `tests/distro/run.sh`, `tests/distro/run.sh --with-apt` | `setup-distro.sh` twice in `ubuntu:24.04` (files, users, units), `systemd-analyze verify` of the units; then the pinned Docker apt install (key, pin, hold) |
| `versions` (ubuntu) | `scripts/check-versions.sh --download` | `install/versions.env`: the rootfs URL answers, `SHA256SUMS` lists the pinned checksum, the downloaded file has it; Docker's apt key fingerprint; every pinned package version exists for noble amd64, and (for `install.sh`) all five packages for noble arm64 and Debian bookworm and trixie (amd64, arm64) at the versions built for them. `ops/versions.env`: the base image index lists amd64 and arm64, the Docker CLI/Compose/buildx versions exist in Docker's Debian repository for both, the supercronic binaries hash to their pins; `ops.env` is empty or a tag with a digest |
| `ops` (ubuntu-24.04, ubuntu-24.04-arm) | `ops/build.sh`, `ops/smoke.sh` | the ops image built natively per architecture; bash 5, the pinned Docker CLI, Compose, buildx, supercronic, jq, git, curl, flock, tzdata, `radioctl --help` in the image, `supercronic -test` of the crontab, `bin/radioctl-ops --help`. A tag `ops-vX.Y.Z` runs `.github/workflows/ops-release.yml`, which pushes the manifest list |
| `linux-install` (ubuntu-24.04) | `sudo tests/linux-install/run.sh --throwaway ...` | the first full install test: `install.sh --yes` on the runner in one pass (Docker at the pins, then the local ops image built by `install.sh`), two locally built releases (1.0.0, 1.0.1) in a local registry, the TLS test Icecast; the stack healthy, `update.sh status` and `sudo update.sh status` (no switches), backup with its second copy, restore drill; the scheduler: a scheduled backup (a crontab of every minute), a caught-up missed backup, a drill inside it, an update refused inside it, the boot check at its start; update to the same release, to 1.0.1, rollback, a Docker restart, a repair run of `install.sh`. On the Mac: `tests/linux-install/container.sh --backend-dir ... --frontend-dir ...` (a throwaway privileged Ubuntu container with systemd and its own Docker) |
| `install` (ubuntu) | `tests/install/run.sh` | the Pester tests of `install.ps1` (Pester 5.7.1, Windows mocked) and PSScriptAnalyzer 1.25.0 in PowerShell 7.5 (pinned image) |
| `windows` (windows-latest) | `tests\install\run-windows.ps1` | the same Pester tests and PSScriptAnalyzer settings under Windows PowerShell 5.1 |

Locally, the same commands; for example `tests/radioctl/run.sh -f restore`
runs the bats tests by name. Runners other than `windows` need only Docker.
Nothing touches the dev stack (`radio-atlantida-dev`): the smoke test uses
the compose project `radio-deploy-liqsmoke` on `127.0.0.1` ports 18000 and
11234 and a stub backend on port 18090, and removes it.

## Config sync with the backend

`config/liquidsoap/radio.liq` is the backend's `dev/liquidsoap/radio.liq` at
the release with `config/liquidsoap/production.patch` applied (the patch is
empty: the dev script reads every startup value from the environment, and
`docker-compose.yml` sets the production values). `config/grafana/`,
`config/loki/loki.yaml`, and `config/alloy/config.alloy` equal the backend's
`dev/` files. They are never edited here: a change is made in the backend,
released, and copied.

```sh
# Check (exit 1 on drift), or rewrite config/ from the backend with --update.
scripts/check-backend-sync.sh [--ref REF] [--backend-dir ../radio-atlantida-backend] [--update]
# liquidsoap --check, then the runtime smoke test at the same ref.
tests/liquidsoap/run.sh [--ref REF] [--backend-dir ../radio-atlantida-backend] [--check-only]
```

**The backend ref** (one rule for both scripts and CI,
`scripts/lib/backend-source.sh`; `scripts/backend-ref.sh` prints it):

1. `--ref`, else `BACKEND_REF` from the environment (CI: the workflow-level
   `env.BACKEND_REF`, empty by default);
2. else `v${BACKEND_TAG}` of `release.env` when that tag exists in the
   backend repository (or in `--backend-dir`);
3. else `BACKEND_SOURCE_COMMIT` of `config/backend-source.env`, with a
   warning (a `::warning::` annotation in CI). This covers the time before
   the first backend release tag. Once `release.env` has digests, the
   `release` job requires the tag, so the fallback cannot hide a missing
   tag.

Without `--backend-dir` the backend is fetched from its public GitHub
repository (no token); CI checks it out once per job with
`actions/checkout` at the resolved ref. `--update` records the commit in
`config/backend-source.env`. The contract header of the script
(`# radio-atlantida-contract: N`, `# min-backend: X.Y.Z`) is read by
`radioctl`: a new contract needs a Liquidsoap restart in a maintenance
window; a backend below `min-backend` is refused.

## A release of the station

1. Backend and frontend tag their releases (`vX.Y.Z`); the images are
   multi-arch manifest lists in GHCR.
2. A PR here sets `release.env` (tags and the manifest list digests from
   `docker buildx imagetools inspect`) and, if the backend's `dev/` changed,
   runs `scripts/check-backend-sync.sh --update` at the new tag.
3. CI is green; the dry run (`docs/dry-run.md`) passes, with the
   migration rehearsal on a production dump if the release has migrations.
4. Merge. On the station, the operator runs "Radio update"
   (`docs/runbook.md` §6).

A change of `install/versions.env` (Docker Engine, Compose, the Ubuntu
image) is applied with `install.bat` in a maintenance window (runbook
§6.3); `radioctl` refuses to update until the engine matches. `install.bat`
installs the pins of its own (Windows) copy of this repository and stops
when they differ from the copy inside the distribution, which `update.bat`
keeps current, so a stale Windows copy cannot install old versions
(`-UseLocalVersions` overrides it on purpose).

## `radioctl` dry-run and test switches

Environment only, never `.env`: `RADIOCTL_PLATFORM` (`wsl`, `linux`,
`macos`), `COMPOSE_PROJECT_NAME`,
`RADIOCTL_BACKEND_IMAGE`, `RADIOCTL_FRONTEND_IMAGE` (a test registry),
`RADIOCTL_COMPOSE_OVERRIDE` (an extra compose file, e.g.
`tests/dryrun/compose.override.yml`), `RADIOCTL_STREAM_OVERRIDE` (a JSON
object merged over `config/stream-defaults.json`, the stream settings the
first run proposes; the cut-over uses whatever the panel says), `RADIOCTL_HEALTH_TIMEOUT`, `RADIOCTL_LOCK_WAIT`, `RADIOCTL_NET_DEV`,
`RADIOCTL_FAKE_CLOCK` and the other test switches listed at the top of
`bin/radioctl`. `DOCKER_DEFAULT_PLATFORM=linux/amd64` forces amd64 images
for a parity run. `bin/radioctl-ops` passes the `RADIOCTL_*` switches into
the container and has its own: `RADIOCTL_OPS_IMAGE` (another image; while
`ops.env` has no digest it uses the local build `radio-atlantida-ops:local`
by itself when it exists, which `install.sh` makes with `ops/build.sh`), `RADIOCTL_OPS_SOCKET`,
`RADIOCTL_OPS_DOCKER_GID`, `RADIOCTL_OPS_PRINT=1` (print the docker command);
it passes `DRYRUN_CERTS` too. The scheduler service reads the switches of
the scheduled commands from `runtime/scheduler.env` (dry run only), and has
`RADIOCTL_SCHEDULER_CRONTAB`, `RADIOCTL_SCHEDULER_BOOT_WINDOW`,
`RADIOCTL_SCHEDULER_FOREGROUND` (`bin/radioctl-scheduler`). `install.sh` and
`update.sh` keep the switches across their own `sudo` (run them without
`sudo` in front, or with `sudo -E`: a plain `sudo` drops them); `update.sh` has
`RADIO_DEPLOY_DIR` and `RADIO_USER`.
