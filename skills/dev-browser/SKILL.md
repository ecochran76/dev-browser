---
name: dev-browser
description: "Use for browser-only tasks on this machine: navigating sites, clicking UI, filling forms, taking screenshots, scraping rendered pages, or testing web-app flows in a real browser. Trigger when the user explicitly wants website interaction or browser automation. Do not use for general shell automation, API-only work, or local file manipulation."
---

# Dev Browser

Use the locally installed `dev-browser` CLI on this machine for sandboxed browser automation.

## Local Setup

- Binary path: `/home/ecochran76/.cargo/bin/dev-browser`
- If `dev-browser` is not on PATH in the current shell, run `. "$HOME/.cargo/env"` first.
- The embedded runtime is already installed under `~/.dev-browser`.

## When To Use

Use this skill when the task requires a real browser, for example:

- Open a website and inspect or interact with it
- Click buttons, fill forms, and submit flows
- Capture screenshots
- Scrape content that depends on client-side rendering
- Test a web app manually through browser actions

Do not use this skill when:

- The task can be done with HTTP requests or an API
- The task is generic automation unrelated to a browser
- The user is asking for local filesystem or terminal work

## Runtime Model

- Scripts run in a sandboxed QuickJS runtime, not Node.js
- Named pages from `browser.getPage("name")` persist between runs
- Default mode launches a separate managed Chromium profile
- `--connect` attaches to an already running Chrome with remote debugging enabled
- `--port` targets a specific CDP port during auto-discovery
- `--profile-path` reads `DevToolsActivePort` from a custom Chrome user-data/profile root
- In WSL, `--connect` can target Windows Chrome profiles under `/mnt/c/Users/.../AppData/Local/...`

## Common Commands

```bash
. "$HOME/.cargo/env"

dev-browser --headless <<'EOF'
const page = await browser.getPage("main");
await page.goto("https://example.com", { waitUntil: "domcontentloaded" });
console.log(await page.title());
EOF
```

```bash
. "$HOME/.cargo/env"

dev-browser --connect <<'EOF'
const tabs = await browser.listPages();
console.log(JSON.stringify(tabs, null, 2));
EOF
```

```bash
. "$HOME/.cargo/env"

dev-browser --connect --port 9333 <<'EOF'
const tabs = await browser.listPages();
console.log(JSON.stringify(tabs, null, 2));
EOF
```

```bash
. "$HOME/.cargo/env"

dev-browser --connect --profile-path ~/.config/google-chrome-agent <<'EOF'
const tabs = await browser.listPages();
console.log(JSON.stringify(tabs, null, 2));
EOF
```

```bash
. "$HOME/.cargo/env"

dev-browser --connect --profile-path "/mnt/c/Users/<WindowsUser>/AppData/Local/Google/Chrome/User Data" <<'EOF'
const tabs = await browser.listPages();
console.log(JSON.stringify(tabs, null, 2));
EOF
```

## Operating Notes

- Prefer direct Playwright actions when the target is known
- Use persistent named pages to avoid re-navigation across turns
- Use `--connect` only when the user wants to work inside an existing Chrome session
- For command details and API reference, run `dev-browser --help`
