---
name: init-private-repo
description: Set up a private GitHub repository for version-controlling the user's domain maps and capture history. Creates the repo via gh, clones locally to a user-chosen path, scaffolds the standard layout (maps/, captures/, README, .gitignore), and registers the local path in the plugin's config so create-domain-map writes there. Use when the user says "set up a private repo for my maps", "I want to version-control my findings", or during setup when they accept the offer to track maps in git.
---

# init-private-repo

One-time provisioning of a private GitHub repository to hold the user's domain maps.

## Why a separate repo

Domain maps accumulate over time and benefit from: history, branches when exploring a target, the ability to share with a collaborator, and an off-machine backup. A separate repo also keeps this content out of the user's general workspace and makes the privacy boundary explicit — this repo is private by default and stays that way.

## Inputs

- Repo name (default: `Browser-Data-Capture-Maps`).
- Local clone path (default: `~/repos/github/my-repos/Browser-Data-Capture-Maps/`).
- Optional: a different GitHub user/org if the user has more than one account configured.

## Preconditions

- `gh` CLI authenticated.
- `git` configured.

## Method

1. **Confirm with the user** before creating anything: "I'll create a private GitHub repo `<owner>/<name>` and clone it to `<path>`. OK to proceed?" Wait for acknowledgement.
2. **Create the remote** with `gh repo create <owner>/<name> --private --description "Private maps of data sources captured with browser-data-capture"`.
3. **Clone locally** to the chosen path.
4. **Scaffold the layout:**
   ```
   <path>/
   ├── README.md          # explains the purpose and layout — see below
   ├── .gitignore         # ignore *.flows, *.har, anything secret-bearing
   ├── maps/              # one subfolder per domain (created on first use)
   ├── captures/          # optional: redacted normalized captures referenced by maps
   └── INTEGRATIONS.md    # user-maintained index of which maps drove which integrations
   ```
5. **Write `.gitignore`** with at minimum:
   ```
   *.flows
   *.har
   *.pcap
   .env
   *.cookies
   secrets/
   ```
   Raw flow files and HARs may contain unredacted auth values; the gitignore is a hard backstop in case the user accidentally drops one in.
6. **Write `README.md`** explaining:
   - This repo holds domain maps produced by `browser-data-capture`.
   - Each `maps/<domain>/` directory is a self-contained map (see `create-domain-map`).
   - The repo is private. Don't make it public — maps may name internal endpoints, sample IDs, and integration approaches the user doesn't want indexed.
   - All maps are built from redacted captures; raw `.flows` / `.har` files are gitignored.
7. **Initial commit and push.**
8. **Register the path** in the plugin's `config.json`:
   ```
   $DATA_ROOT/config.json → "private_maps_repo_path": "<path>"
   ```
9. **Report.** Tell the user the repo URL, the local path, and that `create-domain-map` will now write into the repo by default.

## Safety

- Never make this repo public from this skill, even if asked. If the user wants to publish a single map (e.g. for an open-data integration they want to share), tell them to copy that one map's directory out into a separate public repo by hand. Maintaining a private/public split is the user's call, not the plugin's.
- Do not stage raw capture files, HARs, or flow files. The `.gitignore` should catch them; if the user has manually added one, refuse and explain.
