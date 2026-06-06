# playwright-cdp-skill

Connect Playwright to an existing Chrome/Chromium browser via CDP. Automate, screenshot, and interact with a running browser — no new instance needed.

## Use cases

- **Alternative to DevTools MCP** — lightweight, script-based browser automation without a full MCP server.
- **Browser access for agents** — navigate, extract DOM, fill forms, click elements in a live session.
- **DevTools** — evaluate JS, toggle dark mode, modify styles, read computed properties.
- **Screenshots** — capture full-page or viewport screenshots of any open tab.

## Installation

```bash
npx skills add rstacruz/playwright-cdp-skill
```

## Requirements

- Node.js
- Chromium-based browser with `--remote-debugging-port=9222`
- `playwright-core` (auto-installed on first use)

## Quick start

```bash
google-chrome --remote-debugging-port=9222

cd /tmp/playwright-cdp && node -e "
const { chromium } = require('playwright-core');
(async () => {
  const browser = await chromium.connectOverCDP('http://127.0.0.1:9222');
  const page = browser.contexts()[0].pages()[0];
  await page.screenshot({ path: '/tmp/screenshot.png' });
  await browser.close();
})();
"
```

## Notes

- Uses `playwright-core` to avoid ~100 MB of bundled browsers.
- Use `127.0.0.1` not `localhost` — Node may resolve to IPv6 `::1`.
- `browser.close()` disconnects Playwright but doesn't kill the browser.
