---
name: observe-tab
description: Optional convenience path — observe a live browser tab via the claude-in-chrome MCP server (read_network_requests) and build an endpoint inventory as the user interacts with the page. Useful when the user is already driving a tab and doesn't want to set up a proxy or export a HAR. Requires the claude-in-chrome MCP server. For most cases prefer analyze-har (zero install) or capture-via-proxy (works against any client). Trigger phrases: "observe this tab", "watch the network as I click around", "map this site as I use it".
---

# observe-tab

Convenience path that uses the `claude-in-chrome` MCP server to poll an open tab's network log instead of asking the user to export a HAR or run mitmproxy.

This is the lowest-friction option **when the MCP server is already installed**, but it has the narrowest reach (Chrome, current tab, no mobile, no desktop apps). The other two paths cover more ground.

## Prerequisites

- `claude-in-chrome` MCP server configured. Load its tools with `ToolSearch` first.
- A target tab the user is already driving, or a URL to open.

## Inputs

- Either a tab ID (use `tabs_context_mcp` to list current tabs) or a URL to open in a fresh tab.
- Optional: host filter, so traffic to unrelated origins is excluded.
- Optional: session label.

## What to produce

Write under `${CLAUDE_USER_DATA:-${XDG_DATA_HOME:-$HOME/.local/share}/claude-plugins}/browser-data-capture/data/<session-label-or-timestamp>/`:

- `endpoints.json`, `schemas/`, `summary.md` — same shape as `analyze-har` and `capture-via-proxy`.
- `timeline.jsonl` — append-only log of observed requests in order, useful for flow analysis.

## Method

1. If the user supplied a URL, open a new tab via `tabs_create_mcp`. Otherwise confirm the tab ID.
2. Establish a baseline with one `read_network_requests` call; discard.
3. Loop:
   - Prompt: *"Tell me when you've done something and want me to capture the new traffic, or say 'done' to finalise the map."*
   - Call `read_network_requests`. Diff against what's been recorded; process new entries only.
   - Templatize paths, infer schemas, merge into `endpoints.json`. Append to `timeline.jsonl`.
   - Report progress briefly.
4. On "done", produce `summary.md` and offer `create-domain-map` / `generate-openapi`.

## Filtering and privacy

Same defaults as `analyze-har` and `capture-via-proxy`: drop static-asset noise, redact auth values before writing.

## Limits

- WebSocket frames, server-sent events, and traffic over service workers may not surface cleanly — flag these in the summary if detected.
- Long sessions accumulate state. Beyond a few hundred unique endpoints, suggest narrowing the host filter or splitting into multiple sessions by user task.
