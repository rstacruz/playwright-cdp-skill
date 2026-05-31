# Playwright CDP Browser Access

Connect Playwright to your existing Chrome/Chromium browser via CDP for
automation, screenshots, and DOM interaction. Works with Claude Code,
Codex, Pi, OpenCode, or anything else that supports skills.

## Installation

1. Install the skill: `npx skills add rstacruz/playwright-cdp-skill`
2. Launch Chrome with remote debugging:

```bash
# Linux
google-chrome --remote-debugging-port=9222 --user-data-dir=/tmp/chrome-cdp-profile

# macOS
/Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome \
  --remote-debugging-port=9222 --user-data-dir=/tmp/chrome-cdp-profile
```

Then test it out:

- Verify the endpoint: `curl -s http://localhost:9222/json/version`
- Tell your agent: *Take a screenshot of the current page*

See [SKILL.md](./skills/playwright-cdp-browser-access/SKILL.md) for full
examples, including multi-tab workflows and Firefox launch (not via CDP).

## License

MIT
