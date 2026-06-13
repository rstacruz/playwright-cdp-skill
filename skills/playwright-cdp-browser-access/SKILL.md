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

## Common actions

Assuming `page` is already set up (see [How to write a script](#how-to-write-a-script)):

```js
// open
await page.goto('https://example.com');

// click
await page.click('button.submit');

// dblclick
await page.dblclick('#item');

// type (keystroke-by-keystroke, appends to existing text)
await page.type('input[name="q"]', 'hello world');

// fill (clears field first)
await page.fill('input[name="email"]', 'user@example.com');

// press a key
await page.keyboard.press('Enter');
await page.keyboard.press('Control+a');

// hover
await page.hover('#menu-trigger');

// focus
await page.focus('input[name="search"]');

// check / uncheck
await page.check('#agree');
await page.uncheck('#newsletter');

// select dropdown option
await page.selectOption('select#country', 'US');          // by value
await page.selectOption('select#country', { label: 'United States' }); // by label

// drag and drop
await page.dragAndDrop('#source', '#target');

// upload file(s)
await page.setInputFiles('input[type="file"]', '/path/to/file.pdf');
await page.setInputFiles('input[type="file"]', ['/a.pdf', '/b.pdf']); // multiple

// download (click trigger, wait for download)
const [download] = await Promise.all([
  page.waitForEvent('download'),
  page.click('#download-btn'),
]);
await download.saveAs('/tmp/file.csv');

// scroll
await page.evaluate(() => window.scrollBy(0, 500));      // down 500px
await page.evaluate(() => window.scrollBy(0, -500));      // up 500px
await page.evaluate(() => window.scrollTo(0, document.body.scrollHeight)); // bottom

// scroll into view
await page.locator('#footer').scrollIntoViewIfNeeded();

// wait
await page.waitForSelector('.result');                   // element appears
await page.waitForTimeout(2000);                        // fixed delay (last resort)
await page.waitForNavigation();                           // nav completes
await page.waitForURL('**/dashboard');                   // URL matches
await page.waitForFunction(() => document.readyState === 'complete'); // JS condition

// screenshot
await page.screenshot({ path: '/tmp/shot.png' });
await page.screenshot({ path: '/tmp/full.png', fullPage: true });

// pdf
await page.pdf({ path: '/tmp/page.pdf', format: 'A4' });

// eval (run JS in page context, return value)
const title = await page.evaluate(() => document.title);
const count = await page.evaluate(() => document.querySelectorAll('.item').length);
```

## Working with existing tabs

`connectOverCDP` attaches to the **browser**, not a single tab. Use `context.pages()` to list tabs, `.find()` to locate one by URL, or `context.newPage()` to create one:

```js
const context = browser.contexts()[0];

// List open tabs
const pages = context.pages();
for (const [i, p] of pages.entries()) console.log(i, p.url(), await p.title());

// Create a new tab
const newTab = await context.newPage();
await newTab.goto('https://example.com', { timeout: 15000 });

// Find an existing tab by URL
const existing = context.pages().find(p => p.url().includes('example.com'));
if (!existing) throw new Error('Tab not found');
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

For Firefox, launch a new browser via Playwright instead — use the same boilerplate as the minimal example, but replace `chromium.connectOverCDP(...)` with:

```js
const { firefox } = require('playwright-core');
const browser = await firefox.launch({ headless: false });
const page = await browser.newPage();
// ... then use Common actions as usual
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
