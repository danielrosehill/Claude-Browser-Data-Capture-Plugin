---
name: generate-openapi
description: Generate a draft OpenAPI 3.1 spec from an endpoints inventory produced by analyze-har, capture-via-proxy, or observe-tab. Reads endpoints.json + inferred schemas and emits openapi.yaml plus openapi.json. Use when the user wants to build a stable client against a captured surface — "make an OpenAPI spec from this", "generate a spec so I can build a client", "turn the capture into something I can codegen against".
---

# generate-openapi

Promote a captured endpoint inventory into a draft OpenAPI 3.1 specification, suitable as a starting point for a client SDK or a documented integration contract.

## Inputs

- Path to a `browser-data-capture` run directory (containing `endpoints.json` + `schemas/`). If unspecified, list recent runs from `${CLAUDE_USER_DATA:-${XDG_DATA_HOME:-$HOME/.local/share}/claude-plugins}/browser-data-capture/data/` and ask which one.
- Optional: API title and version (default to host name + today's date).
- Optional: target output path (default: write into the same run directory).

## What to produce

- `openapi.yaml` — primary, hand-editable.
- `openapi.json` — same content, JSON form, for tooling that prefers JSON.
- `notes.md` — gaps the spec couldn't fill confidently: endpoints with too few samples to infer required fields, response variants that couldn't be reconciled, auth flows that need a human to describe.

## Method

1. Load `endpoints.json`. For each endpoint:
   - Build a `paths.<path-template>.<method>` entry.
   - `parameters`: derive `path` parameters from `{id}` / `{uuid}` placeholders, `query` parameters from observed query keys, `header` parameters from interesting headers (skip auth — handled in `securitySchemes`).
   - `requestBody`: if a request schema was inferred, reference `components.schemas`.
   - `responses`: one entry per observed status code, content schema referenced from `components.schemas`.
2. Build `components.securitySchemes` from the auth scheme detected in the inventory. Apply globally via `security:`.
3. Build `components.schemas` from the per-endpoint inferred schemas. Deduplicate by content hash where possible.
4. Set `info.title`, `info.version`, `servers[0].url` (most-contacted host).
5. Validate: minimally check it parses; if `openapi-spec-validator` is available, run it.
6. Write `notes.md` with anything that couldn't be expressed cleanly.

## What this is not

A **draft** spec inferred from observed traffic. Tell the user explicitly:

- Required fields are best-effort guesses from samples — review them.
- Endpoints not exercised in the capture won't appear.
- Error response shapes will be missing for any error path not triggered.
- Path templating may over- or under-merge — review `paths:` for collapses that should be split.

The spec is a starting point, not a finished artefact.
