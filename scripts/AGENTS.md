# scripts/ rules

This folder mixes three kinds of scripts. Follow the rules for the kind you are editing.

- `codex-*.mjs`, `windows-codex.mjs`, `taskboard-supervisor.mjs`: Codex injection and process management (the injector and the runtime helpers the Launcher bundles with it). Follow [inject/AGENTS.md](../inject/AGENTS.md).
- `prepare-tauri-app.mjs`, `tauri-build.mjs`, `create-macos-updater.mjs`, `sign-*`, `verify-*`, `preflight-*`, `release-*`: packaging and release. Follow [src-tauri/AGENTS.md](../src-tauri/AGENTS.md).
- `migrate-to-cloud.mjs`, `wrangler-cloud-adapter.mjs`: cloud data import and Wrangler helpers. Follow [cloud/AGENTS.md](../cloud/AGENTS.md); never run them against a remote target without the user's request.

`dev.mjs` only runs `node --watch server/index.mjs` and Vite side by side.
