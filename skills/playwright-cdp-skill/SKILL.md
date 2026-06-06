---
name: playwright-cdp-browser-access
description: Use Playwright to connect to an existing Chrome or Chromium browser via Chrome DevTools Protocol (CDP) browser instance. Use when the user needs browser automation, screenshots, page evaluation, or DOM interaction against a browser (running or not). Uses a temp Node project in /tmp/playwright-cdp/.
---

# Playwright CDP Browser Access

Connect Playwright to an existing browser via Chrome DevTools Protocol (CDP). This is useful for automating a browser you already launched (e.g. Chrome with `--remote-debugging-port=9222`) without spawning a new one.

## Project setup

The skill uses a temporary Node project at `/tmp/playwright-cdp/`.

### Create the project (one-time)

```bash
if [ ! -d /tmp/playwright-cdp/node_modules/playwright-core ]; then
mkdir -p /tmp/playwright-cdp
cd /tmp/playwright-cdp

# install only the core library (no bundled browsers)
npm init -y && npm install playwright-core
fi
```

## How to write a script

Scripts are plain Node.js executed with `node -e` or written to a file in `/tmp/playwright-cdp/`. The pattern:

1. `require('playwright-core')`
2. `chromium.connectOverCDP('http://127.0.0.1:9222')` — **use `127.0.0.1` not `localhost`** (Node may resolve `localhost` to IPv6 `::1`, which Chromium often doesn't bind)
3. Pick a context and page (or create new ones)
4. Automate
5. Close the browser

### Minimal example

```bash
cd /tmp/playwright-cdp && node -e "
const { chromium } = require('playwright-core');
(async () => {
  const browser = await chromium.connectOverCDP('http://127.0.0.1:9222');
  const context = browser.contexts()[0];
  const page = context.pages()[0];

  await page.goto('https://example.com');
  await page.waitForTimeout(1000);

  await browser.close();
})();
"
```

## Practical examples

### Screenshot a page after applying dark mode

```bash
cd /tmp/playwright-cdp && node -e "
const { chromium } = require('playwright-core');
(async () => {
  const browser = await chromium.connectOverCDP('http://127.0.0.1:9222');
  const context = browser.contexts()[0];
  const page = context.pages()[0];

  await page.goto('http://localhost:3000/es/evaluate?q=hola');
  await page.waitForTimeout(3000);

  await page.evaluate(() =>
    document.documentElement.classList.add('dark')
  );
  await page.waitForTimeout(500);

  await page.screenshot({
    path: '/tmp/verify-evaluate-dark.png',
    fullPage: false
  });
  console.log('Screenshot saved to /tmp/verify-evaluate-dark.png');

  await browser.close();
})();
"
```

### Full-page screenshot of the current active page

```bash
cd /tmp/playwright-cdp && node -e "
const { chromium } = require('playwright-core');
(async () => {
  const browser = await chromium.connectOverCDP('http://127.0.0.1:9222');
  const page = browser.contexts()[0].pages()[0];

  await page.screenshot({ path: '/tmp/screenshot.png', fullPage: true });
  console.log('Saved /tmp/screenshot.png');

  await browser.close();
})();
"
```

### Open a new tab in the existing browser

```bash
cd /tmp/playwright-cdp && node -e "
const { chromium } = require('playwright-core');
(async () => {
  const browser = await chromium.connectOverCDP('http://127.0.0.1:9222');
  const context = browser.contexts()[0];

  const page = await context.newPage();
  await page.goto('https://github.com', { timeout: 15000 });
  await page.waitForTimeout(2000);

  await page.screenshot({ path: '/tmp/github.png' });
  console.log('Saved /tmp/github.png');

  await browser.close();
})();
"
```

### Evaluate and extract data from the DOM

```bash
cd /tmp/playwright-cdp && node -e "
const { chromium } = require('playwright-core');
(async () => {
  const browser = await chromium.connectOverCDP('http://127.0.0.1:9222');
  const page = browser.contexts()[0].pages()[0];

  const heading = await page.evaluate(() =>
    document.querySelector('h1')?.innerText
  );
  console.log('Heading:', heading);

  await browser.close();
})();
"
```

## Working with existing tabs

`connectOverCDP` attaches to the **browser**, not a single tab. After connecting, iterate `context.pages()` to find the tab you want.

### Script 1: create a tab and exit

```bash
cd /tmp/playwright-cdp && node -e "
const { chromium } = require('playwright-core');
(async () => {
  const browser = await chromium.connectOverCDP('http://127.0.0.1:9222');
  const context = browser.contexts()[0];
  const page = await context.newPage();

  await page.goto('https://example.com', { timeout: 15000 });
  console.log('Tab URL:', page.url());

  await browser.close();  // closes the connection, keeps the browser running
})();
"
```

### Script 2: pick up where script 1 left off

```bash
cd /tmp/playwright-cdp && node -e "
const { chromium } = require('playwright-core');
(async () => {
  const browser = await chromium.connectOverCDP('http://127.0.0.1:9222');
  const page = browser.contexts()[0].pages()
    .find(p => p.url().includes('example.com'));

  if (!page) throw new Error('Tab not found');

  await page.screenshot({ path: '/tmp/resume.png' });
  console.log('Screenshot saved');

  await browser.close();
})();
"
```

### List all open tabs

```bash
cd /tmp/playwright-cdp && node -e "
const { chromium } = require('playwright-core');
(async () => {
  const browser = await chromium.connectOverCDP('http://127.0.0.1:9222');
  const pages = browser.contexts()[0].pages();

  pages.forEach((p, i) =>
    console.log(i, p.url(), await p.title())
  );

  await browser.close();
})();
"
```

> **Why not connect directly to a tab WS?** CDP does expose per-tab endpoints (`ws://.../devtools/page/<id>`), but Playwright's `connectOverCDP` expects a **browser** endpoint and returns a `Browser` object. Use the browser endpoint and select from `pages()`.

## Launching a CDP browser manually

If no browser is listening on port 9222, launch one first.

### Find the browser binary

Browser binary names vary by distro and install method. Resolve with a fallback chain:

```bash
# Linux — try common names in order
BROWSER=$(which google-chrome 2>/dev/null \
  || which chromium-browser 2>/dev/null \
  || which chromium 2>/dev/null \
  || which google-chrome-stable 2>/dev/null)

# macOS — Chromium-based browsers
BROWSER="/Applications/Google Chrome.app/Contents/MacOS/Google Chrome"
# Fallbacks:
#   /Applications/Chromium.app/Contents/MacOS/Chromium
#   /Applications/Google Chrome Canary.app/Contents/MacOS/Google Chrome Canary
#   /Applications/Microsoft Edge.app/Contents/MacOS/Microsoft Edge
#   /Applications/Brave Browser.app/Contents/MacOS/Brave Browser
```

### Pre-launch cleanup

Kill any stale instance still holding port 9222 (otherwise the new launch will fail silently):

```bash
# Kill any existing Chromium on port 9222
pkill -f "chrom.*9222" 2>/dev/null || true
sleep 0.5
```

### Launch and verify

Launch the browser **headless-first** (most reliable across environments). Fall back to non-headless only if you need visual interaction:

```bash
$BROWSER \
  --headless \
  --remote-debugging-port=9222 \
  --user-data-dir=/tmp/chrome-cdp-profile \
  --no-first-run \
  --no-default-browser-check \
  &>/tmp/chromium-cdp.log &

# Wait for CDP endpoint to become available (up to 10s)
for i in $(seq 1 20); do
  if curl -s http://127.0.0.1:9222/json/version >/dev/null 2>&1; then
    echo "CDP ready"
    break
  fi
  sleep 0.5
done
```

> Use `127.0.0.1` not `localhost` — Node may resolve `localhost` to IPv6 `::1` which Chromium often doesn't bind.

## Firefox

`connectOverCDP` supports **Chromium** (stable) and **WebKit** — experimental (Playwright v1.61+, via the `transport` overload). Firefox does not expose a CDP endpoint that Playwright can attach to.

For Firefox, launch a new browser via Playwright instead:

```bash
cd /tmp/playwright-cdp && node -e "
const { firefox } = require('playwright-core');
(async () => {
  const browser = await firefox.launch({ headless: false });
  const page = await browser.newPage();

  await page.goto('https://example.com');
  await page.screenshot({ path: '/tmp/firefox.png' });
  console.log('Saved /tmp/firefox.png');

  await browser.close();
})();
"
```

> ⚠️ `firefox.launch()` requires the Playwright Firefox browser binary. Install it with `npx playwright-core install firefox` (~80 MB). This is **not** the system Firefox; Playwright patches its own copy for automation stability.

## Troubleshooting

### Browser fails to launch (non-headless) on Wayland

If you see Vulkan/Wayland errors like:

```
ERROR:ui/ozone/platform/wayland/gpu/wayland_surface_factory.cc:252]
'--ozone-platform=wayland' is not compatible with Vulkan.
```

This is usually non-fatal — the browser may still start successfully. If it doesn't, add `--ozone-platform=x11` to the launch flags:

```bash
$BROWSER \
  --remote-debugging-port=9222 \
  --user-data-dir=/tmp/chrome-cdp-profile \
  --ozone-platform=x11 \
  ...
```

If X11 is also unavailable (e.g. headless server), use `--headless` (already the recommended default above).

### Port 9222 already in use

If the CDP launch fails, check what's holding the port:

```bash
ss -tlnp | grep 9222
# or
lsof -i :9222
```

Then `pkill -f "chrom.*9222"` and retry. Always use `127.0.0.1` in curl/Node scripts, not `localhost`.

### `networkidle` never resolves on heavy sites

Modern sites (news portals, social media, finance) maintain persistent connections for ads, analytics, and live updates — `waitUntil: 'networkidle'` can hang indefinitely. Use `domcontentloaded` with a generous timeout instead:

```js
await page.goto('https://yahoo.com', {
  waitUntil: 'domcontentloaded',
  timeout: 15000
});
await page.waitForTimeout(3000);
```

This loads the page content then gives JS a few seconds to render before taking the screenshot.

## Notes

- `playwright-core` is used instead of `playwright` to avoid downloading ~100 MB of browser binaries. We connect to an existing browser, so we do not need them.
- `connectOverCDP` attaches to the browser. Calling `browser.close()` disconnects from the browser and cleans up Playwright-owned resources — but does **not** kill the browser process. Omit `browser.close()` if you want the Playwright connection to persist across script runs.
- `browser.contexts()[0]` gets the default browser context. `context.pages()[0]` gets the first open tab. These may not exist if the browser has no tabs; handle with `|| await context.newPage()`.
