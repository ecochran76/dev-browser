# CLAUDE.md

This repository ships `dev-browser`: a Rust CLI plus a Node.js daemon for browser automation with a QuickJS sandbox. Use this file as the repo-specific guide when making code changes.

## Tooling

- Use Node.js tooling for `daemon/` and Cargo for `cli/`. Do not use Bun.
- The daemon package uses `pnpm`.
- The repo root contains packaging glue (`bin/`, `scripts/`, `README.md`), but most runtime behavior lives in `cli/` and `daemon/`.

## Validation

Run these before finishing changes that touch runtime code:

```bash
cd daemon && npx tsc --noEmit
cd daemon && pnpm vitest run
cd cli && cargo build
```

If you change daemon runtime code that is embedded into the Rust binary, rebuild the bundles first:

```bash
cd daemon && pnpm bundle
cd daemon && pnpm bundle:sandbox-client
```

`cli/src/daemon.rs` embeds `daemon/dist/daemon.bundle.mjs` and `daemon/dist/sandbox-client.js` via `include_str!`, so `cargo build` only sees the latest daemon changes after those bundles are regenerated.

## Current Architecture

### High-level flow

1. `cli/src/main.rs` parses arguments, reads the script from stdin or a file, and sends a newline-delimited JSON request to the local daemon.
2. `cli/src/daemon.rs` makes sure the embedded Node daemon is extracted under `~/.dev-browser/`, installs runtime dependencies when requested, and starts the daemon if it is not already running.
3. `daemon/src/daemon.ts` listens on a Unix socket (`~/.dev-browser/daemon.sock` on Unix, named pipe on Windows), manages browser instances, and runs scripts one at a time per browser.
4. `daemon/src/sandbox/quickjs-sandbox.ts` executes user scripts inside QuickJS, exposes the allowed globals, and bridges Playwright objects back into the sandbox.

### Important modules

- `cli/src/main.rs`
  CLI entrypoint, help text, request formatting, and output handling.
- `cli/src/connection.rs`
  Local transport to the daemon. Unix socket on Unix, local named pipe on Windows.
- `cli/src/daemon.rs`
  Embedded daemon extraction, `npm install` / Playwright install flow, and daemon process spawning.
- `cli/llm-guide.txt`
  Embedded long-form usage guide shown in `dev-browser --help`. Keep it aligned with CLI behavior.
- `daemon/src/daemon.ts`
  Node daemon entrypoint. Handles `execute`, `browsers`, `status`, `install`, and `stop` requests.
- `daemon/src/protocol.ts`
  Zod schemas and helpers for the newline-delimited JSON protocol between CLI and daemon.
- `daemon/src/browser-manager.ts`
  Launches persistent Chromium profiles, auto-connects to existing Chrome instances, and keeps named pages alive across script runs.
- `daemon/src/sandbox/quickjs-sandbox.ts`
  QuickJS host runtime. Loads the sandbox client bundle, wires `browser`, `console`, `saveScreenshot`, `writeFile`, and `readFile`, and enforces time/memory limits.
- `daemon/src/sandbox/host-bridge.ts`
  Exposes Playwright server-side objects to the sandbox bridge.
- `daemon/src/sandbox/sandbox-transport.ts`
  Creates the sandbox-side Playwright client connection.
- `daemon/src/temp-files.ts`
  Restricts file I/O to `~/.dev-browser/tmp/`.

## Sandbox Model

- User scripts run in QuickJS, not Node.js.
- Do not assume `require`, `import`, `process`, `fs`, arbitrary network access, or other Node globals are available inside scripts.
- The daemon injects a limited API:
  - `browser.getPage(nameOrId)`
  - `browser.newPage()`
  - `browser.listPages()`
  - `browser.closePage(name)`
  - `console.log/info/warn/error`
  - `saveScreenshot`, `writeFile`, `readFile`
- Page objects exposed in the sandbox are bridged Playwright `Page` objects.

## Repo-specific Notes

- Named pages are persistent within a daemon-managed browser. Changes to page lifecycle usually belong in `daemon/src/browser-manager.ts`.
- Auto-connect behavior for external Chrome instances also lives in `daemon/src/browser-manager.ts`. The probe flow uses DevTools discovery and ports `9222` through `9229`.
- Generated daemon bundles are consumed by the CLI. If you change `daemon/src/**` or the sandbox client bundle inputs, verify whether `daemon/dist/*` must be updated too.
- Runtime state lives under `~/.dev-browser/`:
  - `daemon.sock` or the Windows pipe name for transport
  - `daemon.pid` for the background process
  - `browsers/` for persistent Chromium profiles
  - `tmp/` for sandbox file I/O

## Editing Guidance

- Keep CLI help text, `README.md`, and `cli/llm-guide.txt` consistent when behavior or supported flags change.
- Prefer updating the daemon protocol and CLI request handling together; they are tightly coupled.
- When changing sandbox behavior, review both the host side (`daemon/src/sandbox/*.ts`) and the tests under `daemon/src/sandbox/__tests__/`.
