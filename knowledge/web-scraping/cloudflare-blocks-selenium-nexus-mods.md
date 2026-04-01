---
title: Cloudflare blocks all automation on Nexus Mods -- use real browser with userscript
tags: [cloudflare, selenium, nexus-mods, violentmonkey, greasemonkey, bot-detection]
verified: 2026-04-01
platform: linux
---

## Problem
Automating mod downloads from nexusmods.com for large collections (900+ mods). Need to click "Slow Download" on each file page for free accounts.

## Solution
Use a **Violentmonkey/Greasemonkey userscript** in the real Firefox browser combined with a Python **URL feeder** script:

1. Install Violentmonkey in Firefox
2. Add a userscript that auto-clicks "Slow Download" when a Nexus download page loads:

```javascript
// ==UserScript==
// @name         Nexus Auto Slow Download
// @match        https://www.nexusmods.com/*/mods/*
// @run-at       document-idle
// ==/UserScript==
(function() {
    function tryClick(attempts) {
        if (attempts <= 0) return;
        var btn = document.getElementById('slowDownloadButton');
        if (btn) { btn.click(); return; }
        setTimeout(function() { tryClick(attempts - 1); }, 1000);
    }
    setTimeout(function() { tryClick(15); }, 2000);
})();
```

3. Python feeder opens URLs one at a time in the existing Firefox:

```python
import subprocess, time
subprocess.Popen(["firefox", url], stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL)
time.sleep(wait_time_based_on_file_size)
```

The real browser handles Cloudflare transparently. The userscript handles the clicking. The Python script handles sequencing and state tracking.

## What didn't work

1. **Selenium with Firefox** -- Even with the user's real Firefox profile, `dom.webdriver.enabled=False`, spoofed user-agent, and `navigator.webdriver` overridden via JS injection. Cloudflare detects Selenium's Marionette protocol. Gets stuck in infinite "Verify you are human" loop.

2. **Python `requests` with stolen cookies** -- Extracted `cf_clearance` and `nexusmods_session` cookies from Firefox's `cookies.sqlite`. Cloudflare still returns 403 "Just a moment..." because `cf_clearance` is bound to the browser's TLS fingerprint and other signals that `requests` can't replicate.

3. **Nexus Mods API direct download** -- The v1 API endpoint `/v1/games/{domain}/mods/{id}/files/{fid}/download_link.json` only works for Premium accounts. Free accounts need `key` + `expires` params that come from clicking "Download with manager" on the website.

## Context
- Nexus Mods with free (non-premium) account
- Firefox 149.0 (snap) on Ubuntu 24.04
- Selenium 4.41.0
- Cloudflare bot protection as of April 2026
- The Nexus Mods v2 GraphQL API *does* work for reading collection mod lists (use `User-Agent: Vortex/1.12.0`)
