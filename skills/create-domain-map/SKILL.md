---
name: create-domain-map
description: Produce or update a versioned "domain map" document for a single host — a curated, human-readable summary of what's been learned about a target across one or many capture runs. Aggregates endpoints, auth scheme, pagination style, rate-limit signals, ToS notes, and integration notes the user has added by hand. Writes to the user's private maps repo (if registered via init-private-repo) or to the plugin data directory otherwise. Use when the user says "make a map for example.com", "update the example.com map with this capture", "give me a clean summary of what we've learned about this site".
---

# create-domain-map

Aggregate captures into a single, version-controllable document per target domain.

## Where the map is written

Resolve in this order:

1. If `config.json` has a `private_maps_repo_path` set, write into `<private_maps_repo_path>/maps/<domain>/`.
2. Otherwise, write into `${CLAUDE_USER_DATA:-${XDG_DATA_HOME:-$HOME/.local/share}/claude-plugins}/browser-data-capture/data/maps/<domain>/`.

Tell the user which root was used. Encourage routing through a private repo (via `init-private-repo`) if they haven't already — domain maps benefit from version control and a visible history of changes.

## Inputs

- Domain name (e.g. `example.com`).
- Optional: one or more capture run directories to fold into the map. If not supplied, list recent runs that contain entries for this domain and ask which to include.

## What to produce

Inside `<root>/maps/<domain>/`:

- `README.md` — the canonical map. Sections:
  - **Target** — domain, brief description (user-supplied or inferred).
  - **Status** — date of last capture, total endpoints catalogued, capture sources used.
  - **Auth** — observed auth scheme, where credentials are passed, any rotation/expiry signals.
  - **Endpoints** — table grouped by resource: method, path template, purpose (best-effort guess), response status codes seen, sample size.
  - **Schemas** — link to `schemas/` subfolder for per-endpoint JSON Schemas.
  - **Pagination** — detected style (offset, cursor, page-token, none).
  - **Rate-limit signals** — `Retry-After`, `X-RateLimit-*`, observed 429s.
  - **ToS / robots / API policy** — short note on what the user has confirmed about whether programmatic access is permitted. Empty by default; populated by hand.
  - **Integration notes** — free-form section the user maintains.
  - **History** — append-only log: date, what changed, which capture run contributed.
- `endpoints.json` — merged inventory across runs.
- `schemas/` — merged JSON Schemas.
- `openapi.yaml` — most-recent generated spec, if `generate-openapi` has been run.
- `captures/` — symlinks (or, in the private-repo case, references) to the source capture runs that fed the map.

## Method

1. Resolve `<root>` per the rules above.
2. If `<root>/maps/<domain>/` exists, load existing `endpoints.json` and merge new captures into it. Otherwise create fresh.
3. For merge:
   - Same `(method, host, path-template)` → union the query params, sample values, status codes, header observations. Re-infer schema from the union of body samples.
   - New endpoints → add.
4. Re-render `README.md` from the merged inventory + the user's hand-written sections (ToS, integration notes, target description). Preserve hand-written content — never overwrite the user's prose; only refresh the auto-generated tables.
5. Append to the History section: date, capture run IDs folded in, count of new endpoints / changed schemas.
6. If the map lives in a private repo (case 1), stage the changes but do **not** auto-commit unless the user asks — let them review the diff first.

## Privacy

The map is built from already-redacted `endpoints.json` files. Do not re-introduce raw secrets. If a hand-written note in `README.md` contains something that looks like a credential, flag it and ask before committing.
