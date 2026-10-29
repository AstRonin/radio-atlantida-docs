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

## Task delegation

For tasks in `radio-atlantida-backend`, `radio-atlantida-frontend` and `radio-atlantida-deploy`, the main agent does not do the implementation itself. It writes the task and hands it to a subagent through the Agent tool:

- writing or changing code, tests, builds, running commands in a repository → `general-purpose`
- searching code, answering questions about project structure → `Explore`
- planning a large task before implementation → `Plan`

In the brief, state the goal, the files or directories involved, the constraints, and what the subagent must return. The main agent is responsible for writing the brief, checking the subagent's result, and communicating with the developer.

Workflow for implementation tasks:

1. The discussion and the general plan are created in this repository. The plan is saved as `<source>/docs/<task>-plan.md` (for example `backend/docs/<task>-plan.md`). Changes to an existing plan are made in that file here.
2. When the developer approves the plan, it is copied into the source repository's `docs/` by a fresh `docs` subagent, following the push rules (compare with the existing target file, stop on conflicts).
3. The source subagent (for example `general-purpose` in `radio-atlantida-backend`) is started with the path to the plan in its brief. It implements the plan in the source repository and reports back.

Deep analysis (investigating a bug, auditing code, comparing options) is also delegated to a subagent in the relevant repository. The subagent returns its findings to the main agent, which summarizes them for the developer. Analysis results are not saved as documents unless the developer asks for it.

Exceptions: one-step edits to a single file and questions that do not require reading code are done by the main agent. Document sync is still done by the `docs` subagent as described under "Subagent usage".

## Subagent usage

- Start a fresh `docs` subagent for every sync task, so each task begins with a clean context. Do not resume a previous `docs` subagent (SendMessage) unless the developer explicitly asks to continue it.
- Exception: if the `docs` subagent stopped with a question or an unresolved conflict, pass the developer's answer to that same subagent so it continues from where it stopped. Once that task is done, the next task gets a fresh subagent again.
