# Plan — Unknown Stream Output State on the Live Page (radio-atlantida-frontend)

Status: Approved design (2026-11-14); not started. Depends on backend `docs/output-state-and-resync-plan.md` P1 (merged, with its `api/openapi.json`).
Based on: `docs/tech-design.md` v7-27 (§6.5, §7.1, §8.3), backend `docs/output-state-and-resync-plan.md` (P1, decisions 4–6), `docs/f8-live-control-plan.md` (Live page), `docs/f14-broadcast-status-plan.md` (header state)
Last updated: 2026-11-14

## 1. Goal

The backend now resets the Icecast output state to `unknown` when Liquidsoap's heartbeat is silent, and sends `icecast.state` with `state: "unknown"`; `GET /api/playout/status` may return `output: { state: "unknown", since }` (and still leaves `output` out after a backend start, before the first report). The Live page's "Stream output" row must show this as a neutral "no data" state, not as a warning and not as a stale "Connected".

The header's broadcast state (`broadcast.state`, `core/broadcast/`) already has `unknown` and needs no change; after the backend change it becomes correct in more cases (cut-over, backend restart) without frontend work.

## 2. Decisions

1. **One neutral state for absent and unknown.** Both mean "the backend has no current report from Liquidsoap". Both show the plain `.badge` (grey, `--mat-sys-surface-container-high`, `src/styles.scss:143-152`) without `.warn`, with the same text. Only `disconnected` is a warning.
2. **Text.** The existing key `live.onAir.unknown` ("Not reported yet", `src/assets/i18n/en.json:1304`) becomes "No data" (it now also covers a lost heartbeat). The "since" line is shown for `unknown` when `since` is present ("No data since 10:04:12"). English only (`src/assets/i18n/languages.json` has one language).
3. **No new event handling path.** The existing `icecast.state` subscription (`src/app/features/live/live-page.component.ts:158-163`) already patches `output` with `data.state` and the event time; the generated type gains `"unknown"`, and the template handles it.

## 3. Steps

### 3.1 API snapshot

`npm run api:sync` with the backend `api/openapi.json` that contains P1. `src/app/core/api/generated/api.ts` then has `IcecastStateData.state` and `PlayoutOutput.state` as `"connected" | "disconnected" | "unknown"` (now at `:1883-1886`, `:2743-2748`). No hand edits.

### 3.2 Live page (`src/app/features/live/on-air.component.ts:53-67`)

- Badge: `[class.warn]="output.state === 'disconnected'"` instead of `!== 'connected'`, so `unknown` is neutral.
- The text stays `'live.onAir.' + output.state | translate` (`unknown` → `live.onAir.unknown`).
- The `@else` branch (no `output`): show the same neutral badge `live.onAir.unknown` instead of the muted text, so absent and unknown look the same.
- `src/app/features/live/live-page.component.ts`: no change; update the comment at `:38` if it lists the states.
- `src/app/core/realtime/realtime-events.ts:34`: no change (the event type is already registered).

### 3.3 Texts (`src/assets/i18n/en.json`)

`live.onAir.unknown`: "No data". If a help text describes the stream output row, add: "No data: the playout engine has sent no heartbeat for a few seconds, or the backend has just started."

## 4. Tests

`src/app/features/live/live.spec.ts` (`follows the playout events`, `:155-181`, or a new case next to it):

- an `icecast.state` event with `state: 'unknown'` shows `live.onAir.unknown` in a badge without the `warn` class;
- `disconnected` still has the `warn` class; `connected` has neither warning nor unknown text;
- a status without `output` shows the same neutral badge;
- a status with `output: { state: 'unknown', since }` shows the badge and the "since" line.

`npm test`, `npm run lint`, `npm run typecheck` (the new union member must compile everywhere `PlayoutOutput` is used).

## 5. Done when

- The snapshot is the backend's with P1; `npm test`, lint, and typecheck pass.
- On the dev stack: `docker compose stop liquidsoap` → the Live page's "Stream output" shows a grey "No data" within 15 s; starting it again shows "Connected" with its time; the header follows its own `broadcast.state` as before.
