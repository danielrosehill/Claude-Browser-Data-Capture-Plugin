---
name: analyze-har
description: Analyze a HAR file (HTTP Archive — exported from Chrome/Firefox DevTools → Network tab → Save all as HAR with content) and produce a normalized inventory of the endpoints it captured, with inferred request/response schemas, auth scheme detection, and a human-readable summary. The zero-install capture path. Use when the user supplies a .har file, says "analyze this HAR", "what endpoints does this site call", or "I exported the network tab, now what".
---

# analyze-har

Turn a HAR file into a structured map of the API surface it captured. This is the zero-install entry point — any browser can produce a HAR.

## Inputs

- Path to a `.har` file. If the user pastes a URL or refers to a tab instead, redirect them to `observe-tab` (with claude-in-chrome) or `capture-via-proxy` (with mitmproxy).
- Optional: host filter — useful when the HAR contains noise from third-party trackers, fonts, analytics.

## What to produce

Write under `${CLAUDE_USER_DATA:-${XDG_DATA_HOME:-$HOME/.local/share}/claude-plugins}/browser-data-capture/data/<run-id>/`:

- `endpoints.json` — one entry per unique `(method, host, path-template)` tuple, with:
  - sample request URL, query params seen (with example values), request headers of interest (auth, content-type, custom `x-*`), request body shape if JSON
  - response status codes seen, response content-type, response body shape if JSON
  - count of times the endpoint appeared in the HAR
- `schemas/<endpoint-id>.request.json` and `.response.json` — inferred JSON Schema for bodies, when JSON
- `summary.md` — human-readable overview: hosts contacted, endpoint count per host, auth scheme(s) detected, notable patterns (REST vs GraphQL vs RPC, pagination style, ID formats)

## Method

1. Parse the HAR (`.log.entries[]`).
2. Group entries by host. Drop static-asset CDNs (fonts, images, analytics) unless the user asked to keep everything — list what was dropped in the summary.
3. For each remaining entry:
   - Extract `method`, `url`, parse out path and query.
   - Templatize the path: replace numeric IDs and UUID-like segments with `{id}` / `{uuid}` placeholders so `/users/123/posts` and `/users/456/posts` collapse to one endpoint.
   - Capture request headers of interest (`authorization` presence only, `content-type`, `x-*`).
   - If request body is JSON, infer schema (use `genson`).
   - Same for response body.
4. Detect auth scheme: Bearer token, cookie session, API key in header, API key in query, none.
5. Detect API style: GraphQL (POST to a single `/graphql` endpoint with `query` field), REST (resource-shaped paths), RPC (verb-shaped paths).
6. Write the outputs.

## Privacy

Never write raw cookie values, raw bearer tokens, or raw request/response bodies that contain auth secrets to the outputs. Replace with `<redacted>` and note presence. Users hand HAR files over expecting analysis, not a permanent copy of their session secrets.

## Hand-off

When done, point the user at `summary.md` and offer:
- `create-domain-map` — produce a per-domain map document for version control.
- `generate-openapi` — produce a draft OpenAPI 3.1 spec.
