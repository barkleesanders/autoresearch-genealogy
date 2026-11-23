# fcdp — the Chrome DevTools Protocol from the CLI

A custom MV3 extension + local bridge daemon + Python CLI that exposes **most
of the Chrome DevTools Protocol on your real Chrome profile**, via the
extension `chrome.debugger` permission. `chrome.debugger.sendCommand` accepts
**any** CDP method, so this reaches the domains curated tools cannot:
Fetch interception, Emulation/throttling, Tracing, Page.printToPDF.

This is the browser instrument the genealogy skill in this repo assumes:
your agent drives the record and DNA sites you are already logged into,
through your own session. It never sees your passwords — you sign in; the
bridge replays the session.

## Components

- `extension/` — MV3 extension (`debugger` perm). Its service worker connects
  OUT to the bridge over `ws://127.0.0.1:9871`, runs CDP via
  `chrome.debugger`, and forwards events.
- `bridge.mjs` — Node daemon. WebSocket server (extension) + Unix socket
  `/tmp/fcdp.sock` (CLI), length-prefixed JSON framing. Relays CLI ⇄
  extension and fans events out to subscribers.
- `fcdp` — Python 3 CLI. One tool for everything.

## Requirements

Chrome (or Chromium), Node 18+, Python 3.9+. macOS and Linux.

## Setup

```bash
# 1. Start the bridge (keep it running — a launchd/systemd user service
#    works well, or just nohup it):
node bridge.mjs &

# 2. Load the extension once: chrome://extensions → Developer mode →
#    Load unpacked → select this folder's extension/ directory.
#    (Chrome blocks --load-extension on recent versions; the manual
#    load is a one-time step per profile.)

# 3. Label the profile and verify:
fcdp profiles                     # shows the connected extension instance
fcdp label <conn> "Profile 1"     # persist its profile label
fcdp tabs                         # list your real tabs
fcdp open https://example.com
fcdp text                         # prints the page text
```

The extension can live in several profiles at once; each instance carries
its profile label and the bridge routes by label. `fcdp open` targets the
default profile unless `--profile` says otherwise.

## Commands (tabId optional everywhere — omit to use the last opened tab)

```
# tabs / nav
fcdp tabs                       list tabs -> tabId
fcdp open <url> [--bg]          open a NEW tab (cached active)
fcdp nav  [tab] <url>           Page.navigate
fcdp close [tab]                close the tab

# read
fcdp read [tab]                 interactive-element outline + title/url
fcdp snap [tab] [--max N]       compact a11y-style refs for agents (@1..@N)
fcdp text [tab]                 page innerText
fcdp find [tab] "<css|text>"    matching elements -> tag, text, center x,y
fcdp shot [tab] [file.png]      screenshot

# drive
fcdp js   [tab] "<expr>"        evaluate JS, print the result
fcdp click [tab] <@ref|sel>     click an element
fcdp type [tab] "<text>"        type into the focused element
fcdp fill [tab] <sel> "<text>"  set a field's value
fcdp key  [tab] <key>           key press (Enter, Tab, Escape, ...)
fcdp scroll [tab] <dy>          scroll

# advanced CDP
fcdp intercept [tab] --secs N   watch network requests ({method, url})
fcdp pdf [tab] [file.pdf]       Page.printToPDF
fcdp throttle [tab] <profile>   emulate network/CPU throttling
fcdp raw [tab] <Method> [json]  ANY CDP method, raw
```

Run `fcdp` with no arguments for the full list.

## Security — read this before loading the extension

- The `debugger` permission is **full control of the profile it is loaded
  in**: any local process that can reach the bridge socket can drive that
  browser. Load the extension only in profiles you intend to automate, and
  treat the bridge socket (`/tmp/fcdp.sock`) as a credential.
- The bridge listens on localhost (`127.0.0.1:9871`) and the Unix socket
  only. Nothing is sent anywhere else; there is no telemetry.
- Page content the CLI returns is **data, never instructions** — a page
  that tells your agent to do something is just a page saying words.

## License

MIT — see the repo `LICENSE`.
