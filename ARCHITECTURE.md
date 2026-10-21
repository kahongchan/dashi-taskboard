# Architecture

A codemap of Codex Taskboard. It says where things live and which rules span several directories. It does not describe every file: search for the names given here. Process rules for agents are in [AGENTS.md](AGENTS.md); user-facing setup is in [README.md](README.md).

## Bird's-eye view

Codex Taskboard is a local issue board for humans and coding agents. One Node server owns the data. Several clients talk to it over the same HTTP API:

```
 web UI (browser, or iframe inside the Codex App) ─┐
 taskctl CLI (agents, scripts) ────────────────────┼─> server/ (node:http + node:sqlite) ─> .data/
 Tauri Launcher (starts everything) ───────────────┘          │
                                                              └─> Linear / Jira (pull, limited write-back)

 cloud/ (Cloudflare Worker + D1 + R2) serves the same web UI and a parallel /api for shared boards.
```

A typical change: a user drags a card, `web/src/api.ts` sends `POST /api/tasks/:id/move` with the row `version`, the server validates it with `shared/task-input.mjs`, updates SQLite and bumps `version`, then pushes an event on `/api/events` (SSE). Every open UI refetches. `taskctl` reads the same data through the same endpoints.

## Code map

### `server/`
The local backend. Plain `node:http`, no framework.
- `index.mjs` starts the server. `app.mjs` holds config resolution, static file serving, the SSE `EventHub`, and the router: one long chain of `pathname` checks in `createTaskboardServer`.
- Defaults: port `47823` (`CODEX_TASKBOARD_PORT`), host `0.0.0.0` (`CODEX_TASKBOARD_HOST`, only `127.0.0.1` or `0.0.0.0`), data dir `<repo>/.data` (`CODEX_TASKBOARD_DATA_DIR`). When the Launcher runs it, routes sit under a secret `/<instanceToken>` prefix.
- `database.mjs` owns SQLite (`<dataDir>/taskboard.sqlite`, WAL). Schema changes are additive `ALTER TABLE ADD COLUMN` steps guarded by `PRAGMA table_info`; there is no migration table.
- `linear-*.mjs`, `jira-*.mjs`: integrations. Config is JSON in the data dir. Sync pulls at most every 60 s, lazily on `GET /api/tasks`. Write-back on move is limited to `WRITE_BACK_STATE` and never overrides a newer change made in Linear.
- `ai-chat*.mjs`, `ai-turn-owner.mjs`, `codex-app-server.mjs`: the in-board AI chat. Each turn runs `codex exec --json` in a detached owner process, or talks JSON-RPC to `codex app-server`.
- Static assets come from `dist/web` (Vite output). `npm run build:web` must have run for `npm start` to show a UI.

### `shared/`
Plain `.mjs` helpers imported by the server, `cli/`, `web/`, and `cloud/`. Highlights:
- `domain.mjs`: statuses, priorities, fixed project ids (`local`, `jira-my-tasks`, `linear-my-tasks`).
- `task-input.mjs`, `api-fields.mjs`: request parsing and field allowlists, with separate local and cloud variants.
- `agent-loop.mjs`: `taskctl next` ranking, loop markers (`[decision-request]`, `/rework`, `[merged]`, `[released]`), and the `taskboard.agentLoop.v1` settings key.
- `linear-states.mjs`: Linear state name to board status mapping.
- `process-tree.mjs`, `codex-*.mjs`, `executable-command.mjs`: process helpers for the server and scripts.

`web/tsconfig.json` sets `allowJs: false`, so every shared module that `web/` imports has a hand-written `.d.mts` next to it.

### `web/`
React 19 + Vite. No router or store library.
- `src/App.tsx` holds most state, effects, and host wiring. URL state is `?project=` / `?issue=` (`issueRoute.ts`).
- `src/api.ts` is the only fetch wrapper. All URLs are relative to `document.baseURI` (Vite `base: "./"`), because the embedded and Launcher modes serve the app under a prefix. Never hard-code `/api/...` as an absolute URL.
- Live updates: SSE on `/api/events` locally; `revisionPolling.mjs` (poll or WebSocket) in cloud mode only.
- Embedded mode: `?host=codex` (or `deepseek-harness`) inside an iframe. `embeddedHost.mjs` and the `message` handler in `App.tsx` speak the `taskboard:*` postMessage protocol with the injected script.
- `src/i18n.tsx`: two languages, inline `text("中文", "English")` pairs, no translation files.
- `src/storage.ts`: localStorage with a server mirror (`/api/client-storage`). Keys `taskctl` must read (agent loop settings, board display settings) always live on the server.
- Components by area: board (`BoardColumn`, `TaskCard`, `IssueListView`), detail (`TaskDetail`, `IssueRelations`), editor (`InlineMediaComposer`, ProseMirror), Markdown (`MarkdownDocument`, mermaid sanitized with DOMPurify), `GanttView`, `AiChat`, `DashboardView`, integration dialogs.
- Styles are mostly one global `src/styles.css`.

### `cli/taskctl.mjs`
The agent-facing CLI. It finds the server in this order: `CODEX_TASKBOARD_URL`, a runtime descriptor file (`--runtime-file`, `CODEX_TASKBOARD_RUNTIME_FILE`, platform app-data paths, `<repo>/.data/launcher-runtime.json`), then `http://127.0.0.1:47823`. `next --claim` ranks candidates and moves one to `in_progress` with optimistic locking. The agent skill that teaches its use is `skills/manage-taskboard/`.

### `inject/` and `scripts/codex-injector.mjs`
How the board appears inside the Codex desktop App. `inject/codex-taskboard.user.js` is a DOM userscript that adds an entry button and an iframe. The injector launches or finds Codex with a CDP port, installs the script with `Page.addScriptToEvaluateOnNewDocument`, runs the server as a child in `--watch` mode, and writes the runtime descriptor. See [inject/AGENTS.md](inject/AGENTS.md).

### `src-tauri/`
The "Launcher": a tray-only Tauri app (`src/main.rs`) that ships a bundled Node, the server, the web build, `taskctl`, and the skill. It starts the injector, owns process lifecycle, installs the skill and a `taskctl` symlink, and checks for updates. `scripts/prepare-tauri-app.mjs` assembles `src-tauri/resources/app`. See [src-tauri/AGENTS.md](src-tauri/AGENTS.md).

### `cloud/`
Optional shared deployment on Cloudflare: `cloud/src/index.mjs` (Worker plus `RealtimeHub` Durable Object), D1 migrations in `cloud/migrations/`, R2 for attachments, config in `wrangler.jsonc`. It reimplements the `/api` surface on top of `shared/`. See [cloud/AGENTS.md](cloud/AGENTS.md).

### Other
- `integrations/deepseek-harness/`: a sidebar plugin that opens the running board via the runtime descriptor.
- `scripts/`: dev runner, injector, packaging, release, and verify scripts. See [scripts/AGENTS.md](scripts/AGENTS.md).
- `test/`: `node --test` files plus one vitest component test. Server tests create a temp data dir and call `createTaskboardServer({ dataDirectory })` with `listen({ port: 0 })`. Cloud tests run the Worker under Miniflare (`test/helpers/cloud-worker-harness.mjs`).
- `.github/workflows/`: `check.yml` (PR and main; skips native builds when only web, docs, or README files changed), `release-macos.yml` (tag `v*`), `signpath-windows-test.yml` (manual).

## Cross-cutting rules

These are easy to break because the code that must change together lives in different directories.

- **Adding a task field.** Touch the SQLite column (and any `CHECK`), the allowlists and parsers in `shared/task-input.mjs` (local and cloud variants), a new D1 migration plus `cloud/src/index.mjs`, `web/src/types.ts` and `web/src/api.ts`, and `cli/taskctl.mjs` if agents need it.
- **Adding or renaming a status or priority.** `shared/domain.mjs`, the SQL `CHECK` constraints in `server/database.mjs`, the D1 schema, `shared/linear-states.mjs`, Linear `WRITE_BACK_STATE`, the Jira mapping, and the labels in `web/src/i18n.tsx`.
- **Optimistic locking.** Every task, comment, and README write increments `version`; clients send the version they read and get `409 VERSION_CONFLICT` on mismatch. Comment and attachment writes also take a new `change_revision` from the global counter, or `?after=` readers miss them.
- **Shared module types.** Changing the exported shape of a shared `.mjs` that `web/` imports means updating its `.d.mts` in the same change.
- **Skill text is tested.** `test/manage-taskboard-skill.test.mjs` reads `skills/manage-taskboard/SKILL.md`.

## Commands

| Goal | Command |
|---|---|
| Local dev (server with `node --watch` on 47823, Vite on 5173 proxying `/api`) | `npm run dev` |
| Run against Codex App with injection | `npm run codex` |
| Full local check (typecheck, web build, tests) | `npm run check` |
| Tests only | `npm test` |
| Cloud worker locally | `npm run dev:cloud` |

Node `>=22.5` is required (`node:sqlite`). Rust toolchain is pinned in `rust-toolchain.toml`. The `--dev` flag that `scripts/dev.mjs` passes to the server is currently ignored.
