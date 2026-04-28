---
name: capture-via-proxy
description: Capture network traffic via a local mitmproxy instance and emit a normalized capture file the rest of the plugin can analyze. Use when the user wants to capture traffic from a desktop app, mobile app on the same network, or a browser without using DevTools — anywhere a HAR export isn't practical. The plugin starts mitmdump in the background, walks the user through trusting the local CA, runs for a chosen duration or until stopped, then converts the flow file into the same normalized shape analyze-har produces.
---

# capture-via-proxy

Use mitmproxy as the capture path when a HAR export from DevTools isn't workable — desktop apps, mobile apps proxied to the user's machine, browsers where DevTools won't stay open across navigations.

This is **bring-your-own-traffic**: the user is capturing traffic on their own machine for their own integration work. The plugin runs the proxy and normalizes the output; it does not bypass anything.

## Prerequisites

- `mitmproxy` installed locally (verified by `setup`).
- The user has the right and the intent to capture this traffic — typically against a public site they want to ingest from, an open-data portal, or a system they own.

## Inputs

- Optional: target hostname filter (only record flows where the host matches). Strongly recommended — without it the capture picks up every HTTP request the machine makes.
- Optional: duration cap (default: until the user stops it).
- Optional: session label, used to name the output directory.

## What to produce

Write under `${CLAUDE_USER_DATA:-${XDG_DATA_HOME:-$HOME/.local/share}/claude-plugins}/browser-data-capture/data/<session-label-or-timestamp>/`:

- `capture.flows` — raw mitmproxy flow file (kept for replay/debug).
- `endpoints.json` + `schemas/` + `summary.md` — same normalized shape `analyze-har` produces, so `generate-openapi` and `create-domain-map` consume it transparently.
- `redactions.log` — record of cookie/auth values that were stripped before writing the normalized outputs.

## Method

1. **Choose a port.** Default 8080; if taken, increment.
2. **Start `mitmdump`** in the background, writing to `capture.flows`. Use `--set confdir=$DATA_ROOT/state/mitmproxy` so the local CA lives under the plugin's state dir, not in `~/.mitmproxy`.
3. **First-run CA trust.** If this is the first time mitmproxy has run on this machine, walk the user through visiting `http://mitm.it` from the device whose traffic they're capturing and installing the CA. Do **not** attempt to install the CA into the system trust store automatically — that's a sudo-grade decision the user makes deliberately.
4. **Tell the user how to point traffic at the proxy.**
   - For Chrome on the same machine: `google-chrome --proxy-server=http://localhost:<port>` (fresh profile recommended).
   - For a phone on the same LAN: WiFi settings → manual proxy → host = workstation IP, port = `<port>`. Stop the proxy as soon as the user is done — leaving a phone proxied through the workstation is not a default state.
   - For a desktop app that respects `HTTP_PROXY` / `HTTPS_PROXY`: those env vars.
5. **Run the capture.** Either for the duration the user specified, or until they say "stop". Apply the host filter as flows arrive — discard non-matching flows so the file stays small.
6. **Stop mitmdump cleanly** (SIGTERM, wait for flush).
7. **Normalize.** Convert `capture.flows` into `endpoints.json` + `schemas/` using the same logic as `analyze-har`. Apply secret redaction (cookies, bearer tokens, common API-key header names).
8. **Produce `summary.md`** and hand off — offer `create-domain-map` and/or `generate-openapi`.

## Privacy

Same hard rule as the rest of the plugin: cookie values and bearer tokens are redacted from `endpoints.json` and `schemas/`. The raw `capture.flows` file may still contain them — if the user plans to share the run, tell them to share the normalized outputs, not the raw flow file, or to use `mitmproxy`'s built-in flow-stripping first.

## When to stop

- If the capture is producing noise from origins outside the target host, suggest tightening the host filter rather than letting the capture run longer.
- If the user mentions a site that has explicit API terms of service or a published API, route them to that instead — this skill is for cases where there's no published interface.
