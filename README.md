# Playwright CDP Browser Access

Connect Playwright to your existing Chrome/Chromium browser via CDP for automation, screenshots, and DOM interaction. Works with Claude Code, Codex, Pi, OpenCode, or anything else that supports skills.

## Installation

1. Install the skill: `npx skills add rstacruz/playwright-cdp-skill`
2. Test it out by telling your agent: *Take a screenshot example.com via playwright/chrome*


<!--
2. Launch Chrome with remote debugging:

```bash
# Linux
google-chrome --remote-debugging-port=9222 --user-data-dir=/tmp/chrome-cdp-profile

# macOS
/Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome \
  --remote-debugging-port=9222 --user-data-dir=/tmp/chrome-cdp-profile
```
-->

See [SKILL.md](./skills/playwright-cdp-browser-access/SKILL.md) for full examples, including multi-tab workflows and Firefox launch.

## License

MIT
