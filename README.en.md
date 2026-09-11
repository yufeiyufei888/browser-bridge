# browser-bridge

> **Give any AI agent your real browser.** One command to CDP — logged-in sessions, cross-platform, cross-agent.

`browser-bridge` is a **SKILL.md skill package** loadable by DSH, Claude Code, Codex CLI, and any agent that reads `SKILL.md`. It adds what built-in web tools lack: **a browsing strategy** plus **direct control of the browser you already use** (Chrome DevTools Protocol, so your cookies and logins apply).

Zero dependencies: plain Node.js ESM (Node **22+**, native WebSocket). Works on Windows, macOS and Linux.

## Use it when

- the page needs a **login**, or is rendered by JS, or blocks scrapers;
- you must **click / fill / upload / paginate** like a human;
- you need a **screenshot** or a video frame;
- WebSearch / WebFetch came back empty — CDP is the fallback.

For plain static pages, prefer the agent's built-in WebSearch/WebFetch: cheaper and faster.

## Install

```bash
# DSH
git clone https://github.com/yufeiyufei888/browser-bridge.git ~/.dsh/skills/browser-bridge

# Claude Code
git clone https://github.com/yufeiyufei888/browser-bridge.git ~/.claude/skills/browser-bridge

# Codex CLI
git clone https://github.com/yufeiyufei888/browser-bridge.git ~/.codex/skills/browser-bridge
```

On Windows use `%USERPROFILE%\.dsh\skills\browser-bridge` etc. No git? Grab the ZIP from the GitHub page.

## Requirements

1. Node.js **22+**.
2. Enable remote debugging in your browser: open `chrome://inspect/#remote-debugging` (or `edge://inspect/#remote-debugging`) and tick **Allow remote debugging for this browser instance**. Click **Allow** on the one-time consent dialog.
3. Self-check: `node "<SKILL_DIR>/scripts/check-deps.mjs"` → expect `browser: ok (...)`, `proxy: ready (...)`.

## Quick check

```bash
curl -s http://localhost:3456/targets
curl -s -X POST --data-raw 'https://example.com' http://localhost:3456/new
curl -s -X POST "http://localhost:3456/eval?target=<ID>" -d 'document.title'
curl -s "http://localhost:3456/close?target=<ID>"
```

## Windows notes

- `curl` is an alias of `Invoke-WebRequest` in PowerShell 5.1 — always call **`curl.exe`**.
- There is no `pkill`; stop the proxy with `Get-CimInstance Win32_Process` filtering on `cdp-proxy`, then `Stop-Process`.
- For JS containing quotes/Chinese/multiline, write a **UTF-8 no-BOM** file and use `--data-binary "@file"`.
- `/screenshot` hangs (30s timeout) while the browser window is **minimized or occluded** — restore the window first.

Details: [`docs/`](docs).

## Attribution & license

- Upstream: [**web-access**](https://github.com/eze-is/web-access) · MIT · by **一泽Eze** (commit `33eef84`).
- This distribution: **browser-bridge** — cross-platform / cross-agent packaging and docs. `scripts/`, `references/` and `templates/` are **byte-identical** to upstream.
- Licensed MIT. See [`LICENSE`](LICENSE) and [`NOTICE.md`](NOTICE.md).

> Already using upstream `web-access`? Capabilities overlap — **pick one** (both bind the same proxy port).

## Safety

It drives your real, logged-in browser. Some platforms detect automation and may rate-limit or ban accounts — use a throwaway account for social sites. The scripts never phone home; `find-url.mjs` only reads local bookmark/history files. Use at your own risk.
