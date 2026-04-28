---
name: setup
description: First-run setup for Browser-Data-Capture — provisions the user data folder under the standard $CLAUDE_USER_DATA pattern, optionally registers a path to the user's private maps repo for version-controlled storage, and verifies dependencies (mitmproxy, genson, pyyaml, optional openapi-spec-validator). Use when onboarding a new machine, when another skill reports a missing dependency, or when the user says "set up browser-data-capture" / "configure capture" / "where does this plugin store things".
---

# setup

One-time / re-check setup for Browser-Data-Capture.

## Steps

1. **Resolve the data root.**
   ```
   DATA_ROOT="${CLAUDE_USER_DATA:-${XDG_DATA_HOME:-$HOME/.local/share}/claude-plugins}/browser-data-capture"
   ```
   Create the standard layout if missing:
   ```
   $DATA_ROOT/
   ├── config.json     # small settings + pointer to private maps repo
   ├── data/           # per-capture run directories
   ├── cache/          # regenerable
   └── state/          # runtime state
   ```

2. **Write a default `config.json`** if absent:
   ```json
   {
     "default_host_filter": null,
     "drop_static_assets": true,
     "redact_secrets": true,
     "private_maps_repo_path": null,
     "user_contact_for_disclosures": null
   }
   ```
   - `redact_secrets` should never be flipped to `false` from this skill — flag as manual-edit only.
   - `private_maps_repo_path` is a pointer to a user-owned local clone of a private GitHub repo (set up by `init-private-repo`). When set, `create-domain-map` writes maps there instead of into `$DATA_ROOT/data/`.
   - `user_contact_for_disclosures` (name + email) is used by `disclose-finding` to populate the "from" line in vulnerability disclosure drafts. Ask the user once during setup; never autofill.

3. **Ask the user (interactive, one-time):**
   - "Do you want me to set up a private GitHub repo for version-controlling your discovered endpoint maps?" — if yes, hand off to `init-private-repo`. If no, leave `private_maps_repo_path` as `null` and proceed.
   - "What name and email should I use if you ever need to send a disclosure email to a site owner?" — store in `user_contact_for_disclosures`. Skip if the user prefers to be prompted each time.

4. **Check capture dependencies:**
   - **mitmproxy** (`mitmproxy --version`) — primary capture path used by `capture-via-proxy`. If missing, give the install command (`pip install --user mitmproxy` or distro package) but don't auto-install.
   - **HAR export** is dependency-free (any browser does it) — note this as the zero-install path.
   - **claude-in-chrome MCP server** — optional. Only needed if the user wants `observe-tab` for live tab observation. Note as optional, not required.

5. **Check inference dependencies:**
   - `genson` — JSON schema inference (`pip install --user genson`).
   - `pyyaml` — YAML emission. Almost always present.
   - `openapi-spec-validator` — recommended, used by `generate-openapi` to validate output.

6. **Report.** Print a status table:
   ```
   data root:                <path>      [ok / created]
   config.json:              <path>      [ok / created]
   private maps repo:        <path|null>
   disclosure contact:       <name|null>
   mitmproxy:                <version|missing>
   claude-in-chrome MCP:     <present|absent>  (optional)
   genson:                   <version|missing>
   openapi-spec-validator:   <version|missing>  (recommended)
   ```
   For anything missing, give the exact install command and stop short of running it without confirmation.
