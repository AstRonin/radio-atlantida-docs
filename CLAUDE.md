# radio-atlantida-docs: rules for AI agents

This repository is the central copy of the project documents. Sources: `radio-atlantida-backend`, `radio-atlantida-frontend`, `radio-atlantida-deploy`.

## Layout

- `backend/`, `frontend/`, `deploy/` hold each source's documents under their original relative paths (`docs/...`, `README.md`, `CLAUDE.md`).
- Shared documents that are identical across sources live once, at the repository root: `base.md`, `tech-design.md`.
- Do not keep the same document in more than one folder. Before storing a copy, check its hash against the other sources. If the copies are identical, store one copy at the root (or in one agreed place) and remove the others.
- If the copies differ, do not pick one silently. Show the difference and ask the developer which version to keep.

## Pull (sources to this repo)

- Compare each source file with its copy here before overwriting. Never copy `.env` files, secrets, `node_modules`, `dist/`, or `.git`.
- Identical shared documents go to the root. Divergent copies stay in their source folder until the developer decides.

## Push (this repo to sources)

- A root shared document is pushed to `docs/` in each source that uses it. Use the rules in the sync task: compare each source with the last synced version (`git show HEAD:<path>`) and stop on conflicts.
- Never write outside the sources' `docs/` (or `README.md`) and do not run git commands that change a source repository.

## Placement decisions

- `base.md`, `tech-design.md`: root. The backend/frontend version is canonical; it is also pushed to deploy.
- `deploy-init-plan.md`: stored once in `deploy/docs/`. The deploy version is canonical and is pushed to backend and frontend `docs/`.
- `backend-init-plan.md`, `frontend-init-plan.md`: intentionally kept in their subfolders (`backend/docs/` + `deploy/docs/`, `frontend/docs/` + `deploy/docs/`), even though the copies are identical. Do not deduplicate.

## Subagent usage

- Start a fresh `docs` subagent for every sync task, so each task begins with a clean context. Do not resume a previous `docs` subagent (SendMessage) unless the developer explicitly asks to continue it.
- Exception: if the `docs` subagent stopped with a question or an unresolved conflict, pass the developer's answer to that same subagent so it continues from where it stopped. Once that task is done, the next task gets a fresh subagent again.
