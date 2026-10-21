# src-tauri/ rules

The Launcher: a tray-only Tauri app, all logic in `src/main.rs`. Risk level: high (process lifecycle, updater, release, writes to user state). Root [AGENTS.md](../AGENTS.md) still applies; run only one Launcher, updater, signing, or release task at a time.

## What it does to the user's machine

On startup the Launcher can:
- quit the user's ordinary Codex App (after a dialog) and relaunch it with a managed profile;
- replace `~/.agents/skills/manage-taskboard` with the bundled skill (delete, then copy);
- symlink `/opt/homebrew/bin/taskctl` to the bundled CLI;
- install an autostart LaunchAgent;
- rename beta and stable bundles under `/Applications`.

Treat changes to any of these as destructive operations. Never overwrite the installed `/Applications/Codex Taskboard.app` while testing; keep it for update-check verification.

## Process ownership

The Launcher runs bundled Node on `scripts/codex-injector.mjs --launch --watch`, passes the server socket as inherited fd 5, and records children in `launcher-child.json`. A stop kills a PID only when its command matches that record. Keep that check when changing stop logic.

## Packaging

- `scripts/prepare-tauri-app.mjs` wipes and rebuilds `resources/app` (server, shared, `dist/web`, skill, injector scripts, `taskctl`) and downloads a pinned Node into `binaries/`. A target can only be prepared on its own OS.
- If you add a file the packaged app needs at runtime, add it there and to the checks in `scripts/verify-packaged-taskctl.mjs` / `scripts/preflight-macos-app.mjs`.
- Platform configs: `tauri.conf.json` (macOS, updater artifacts on), `tauri.linux.conf.json`, `tauri.windows.conf.json`. Rust version: root `rust-toolchain.toml`.

## Versions and updater

- A version bump touches `package.json`, `package-lock.json` (two places), `src-tauri/Cargo.toml`, `src-tauri/Cargo.lock`, and `src-tauri/tauri.conf.json`. Release preflight fails if they differ.
- The updater public key and endpoint in `tauri.conf.json` are release-critical. The endpoint points at the upstream repository's releases.
- Signing secrets (`TAURI_SIGNING_*`, `APPLE_*`, `SIGNPATH_API_TOKEN`, `RELEASE_RULESET_TOKEN`) exist only in CI. Do not add them locally or to files.

## Verify (macOS)

```bash
npm run app:prepare -- --target universal-apple-darwin
```

```bash
npm run tauri -- build --target universal-apple-darwin --bundles app --no-sign --config '{"bundle":{"createUpdaterArtifacts":false}}'
```

Then run `npm run app:verify-packaged-taskctl -- <App>` and `npm run app:preflight -- <App>` on the built app, and `node --test test/launcher-release.test.mjs`. CI (`check.yml`) builds macOS, Windows, and Linux launchers on every non-web PR, so local packaging is needed only when the task's direct path depends on it.
