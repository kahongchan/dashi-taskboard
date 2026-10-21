# inject/ rules

Applies to `inject/codex-taskboard.user.js` and its host side, `scripts/codex-injector.mjs` (plus `scripts/codex-*.mjs`, `scripts/windows-codex.mjs`, `scripts/taskboard-supervisor.mjs`). Risk level: high (Codex injection, process management). Root [AGENTS.md](../AGENTS.md) still applies.

## How it fits

The injector starts or finds a Codex App with a CDP port, prepends the managed origin, capability, and page URL to `codex-taskboard.user.js`, and installs it with `Page.addScriptToEvaluateOnNewDocument`. The script adds an iframe that loads `http://127.0.0.1:<port>/<instanceToken>/?host=codex`. In `--watch` mode the injector also runs `server/index.mjs` as a child and writes the runtime descriptor (`launcher-runtime.json`, mode 0600, contains the secret instance token).

## Do not break the user's running Codex

- Never reuse the coordinator's CDP port, Codex profile, data dir, or runtime descriptor for worktree testing. Set all of these to your own values:
  `CODEX_TASKBOARD_CODEX_PROFILE`, `CODEX_TASKBOARD_DATA_DIR`, `CODEX_TASKBOARD_RUNTIME_FILE`, `CODEX_TASKBOARD_PORT`, and `--port`.
- The default profile is a fixed `/tmp` path. Two runs on that default profile but different CDP ports will SIGTERM, then SIGKILL, each other's Codex.
- `--refresh`, `--refresh-if-running`, and `npm run build` re-inject into every debuggable Codex window and restart the resident injector, including the coordinator's. Use `npm run build:web` for web-only builds.
- Do not pass `--app-path` to an installed app you do not own.

## Keep both sides in sync

- The `__codexTaskboard*V1` binding names and the `taskboard:*` postMessage types must match across `user.js`, `codex-injector.mjs`, and `web/src/App.tsx` / `web/src/embeddedHost.mjs`.
- Bump `VERSION` in `user.js` when its behaviour changes, so a stale injected copy is detectable.

## Verify

```bash
node --test test/inject.test.mjs test/injector.test.mjs test/injector-host-runtime.test.mjs test/codex-cdp-pipe.test.mjs test/windows-codex.test.mjs
```

A live check is `npm run codex` with the isolated environment above. Stop only the processes you started, by exact PID.
