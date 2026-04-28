# Browser Data Capture

A Claude Code plugin to streamline programmatic data ingestion against sites and apps that don't ship a documented API. Capture network traffic, map endpoints, infer schemas, and produce a draft OpenAPI spec you can build a stable client against.

> **White-hat use only.** This plugin is for building integrations against open data sources, public-interest data, your own systems, and sites where you have the right and intent to ingest. It does not bypass authentication, evade rate limits, or enumerate hidden surfaces. If a finding turns up incidentally that looks security-relevant, the `disclose-finding` skill helps you act as a good-faith reporter.

## Capture paths

Three independent ways to feed the plugin — pick whichever fits your situation:

| Path | Tool | Strength | Use when |
|------|------|----------|----------|
| **HAR** | Browser DevTools | Zero install, works in any browser | One-off captures, sharing with a teammate, no proxy setup |
| **mitmproxy** | `capture-via-proxy` | Captures any client, not just browsers | Desktop apps, mobile apps on your LAN, browsers across navigations |
| **claude-in-chrome** | `observe-tab` | Lowest friction if MCP server is installed | You're already driving a Chrome tab in this Claude session |

All three normalize to the same internal `endpoints.json` shape so the downstream skills (`generate-openapi`, `create-domain-map`) work transparently regardless of capture path.

## Skills

- **`setup`** — first-run setup; provisions the data folder, registers your private maps repo (optional), records your disclosure contact, checks dependencies.
- **`analyze-har`** — analyze a HAR exported from DevTools.
- **`capture-via-proxy`** — run `mitmdump` in the background and capture traffic from any client on the machine (or a phone proxied to it).
- **`observe-tab`** — observe a live Chrome tab via the `claude-in-chrome` MCP server (optional dependency).
- **`generate-openapi`** — turn a captured inventory into a draft OpenAPI 3.1 spec.
- **`create-domain-map`** — produce or update a versioned, human-readable map of a single target domain across multiple capture runs.
- **`init-private-repo`** — provision a private GitHub repo for version-controlling your domain maps.
- **`disclose-finding`** — draft a courteous, vendor-friendly disclosure email if a capture incidentally surfaces something that looks security-relevant.

## Typical workflows

**One-off integration scoping (zero install):**
1. In DevTools → Network → "Save all as HAR with content".
2. `analyze-har <path>`
3. `generate-openapi`

**Sustained mapping of a target you care about:**
1. `setup` — answer yes to the private repo prompt; `init-private-repo` runs and registers the path.
2. Capture (any path), as often as makes sense.
3. `create-domain-map example.com` — folds new captures into the persistent per-domain map in your private repo. Each map accumulates: endpoints, schemas, auth scheme, pagination, rate-limit signals, your hand-written ToS notes and integration notes, and a history log.
4. Commit and push when you want a checkpoint.

**Capture from a desktop or mobile app:**
1. `capture-via-proxy --host api.example.com` — starts mitmproxy in the background and walks you through CA trust + proxy pointing.
2. Drive the app normally.
3. Stop the proxy when done.
4. Continue with `create-domain-map` and/or `generate-openapi` as above.

## Storage

Plugin data lives at:

```
${CLAUDE_USER_DATA:-${XDG_DATA_HOME:-$HOME/.local/share}/claude-plugins}/browser-data-capture/
```

Per-run captures go under `data/<run-id>/`. If you've registered a private maps repo via `init-private-repo`, persistent domain maps live there instead — outside the plugin install directory, version-controlled, and yours.

## Privacy

By default the plugin redacts cookie values and bearer tokens before writing anything to disk — only the *presence* of an auth header is recorded, never the value. This default is enforced; turn it off only by hand-editing `config.json`.

Raw mitmproxy flow files and HARs may still contain unredacted auth values. The `init-private-repo` `.gitignore` ensures they never reach git by accident.

## What this is not

- Not a scraper. Output is a structured map of an API surface — a starting point for a documented client integration, not a one-off data pull.
- Not a pentest tool. The capture skills observe traffic the user is generating; they don't probe, enumerate, or fuzz.
- The OpenAPI specs produced are **drafts inferred from observed traffic** — useful as a starting point for documentation or a client SDK, but not authoritative. Endpoints not exercised in the capture won't appear; required-field inference is best-effort.

## Installation

```bash
claude plugins marketplace add danielrosehill/Claude-Code-Plugins
claude plugins install browser-data-capture@danielrosehill
```

## License

MIT — see [LICENSE](LICENSE).
