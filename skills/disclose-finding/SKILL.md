---
name: disclose-finding
description: Help the user act as a good-faith reporter when a capture run incidentally surfaces something that looks like an unauthenticated endpoint exposing data that probably shouldn't be public, a leaked token in a response, or another security-relevant finding. Drafts a courteous, vendor-friendly disclosure email, identifies the right contact (security.txt, security@, abuse@, public bug-bounty programme), and writes the report into the maps repo so the user has a record. White-hat use only. Use when the user says "we found something we should report", "looks like an open endpoint", "draft a disclosure email for this finding".
---

# disclose-finding

A capture run is intended to map endpoints the user can build a legitimate integration against. Occasionally, it incidentally surfaces something that looks like a security issue — an endpoint that returns data without auth when it clearly should require it, a token or key embedded in a response body, an IDOR-shaped pattern. This skill helps the user behave well when that happens: stop, document, disclose, move on.

This skill is **not** a research or testing tool. It does not probe further, enumerate, or attempt to confirm impact. The premise is that the finding is already incidental — Claude has seen it once, in normal use, and the next responsible step is reporting.

## When to invoke

The user (or a previous skill in this plugin) flags one of:

- An endpoint returning what looks like personal or sensitive data without an auth header.
- A response body containing what looks like a credential, internal token, private key, or DB connection string.
- A response disclosing internal infrastructure details that shouldn't be public (stack traces with paths, internal hostnames, debug routes).
- An ID-shaped parameter that, when changed, returns another user's data — and the user noticed by accident, not by trying.

If the user is *trying* to find this kind of thing rather than incidentally surfacing it, this is the wrong plugin and the wrong skill. Tell them to use a proper bug-bounty or pentest workflow under explicit scope.

## Inputs

- A short description of what was observed and where (URL, method, what came back).
- The target domain.
- Optional: severity gut-check from the user (informational / low / medium / high / critical).

## Method

1. **Pause and confirm.** Before doing anything else, confirm with the user that they want to act on this finding. Some users prefer to ignore non-impactful informational findings; others want to report everything. Don't assume.

2. **Identify the right contact.** In order of preference:
   - `https://<domain>/.well-known/security.txt` — fetch it and use the `Contact:` field.
   - A public bug-bounty programme (HackerOne, Bugcrowd, Intigriti) — search and link if found.
   - `security@<domain>` — use only if security.txt is absent and no programme exists.
   - `abuse@<domain>` or generic `support@<domain>` — last resort, note that these may not reach the right team.
   - For government / regulated entities, the relevant national CERT (e.g. CERT-IL for Israel, CISA for US federal) may be more appropriate when the vendor is unresponsive.
   Do not invent a contact address. If you cannot find one, tell the user and stop.

3. **Draft the disclosure email.** Use this skeleton (adjust tone to match the severity):

   ```
   Subject: Possible security issue on <domain> — incidental finding

   Hi <team>,

   I'm <name> (<contact>), a developer working on a programmatic integration
   against <domain>. While capturing the network traffic of normal browsing in
   order to understand the data shape, I noticed what looks like an
   unintentionally exposed endpoint / leaked value:

       <one-paragraph factual description>
       Endpoint:   <method> <url>
       Observed:   <what came back, in neutral terms>
       Date/time:  <UTC timestamp>

   I have not probed further, have not attempted to enumerate, and have not
   shared this with anyone else. I'm flagging it so your team can review it.

   I'm happy to provide additional details (request/response samples) over a
   secure channel if useful. I don't expect a reward and I'm not requesting
   one — this is a courtesy disclosure.

   Best regards,
   <name>
   ```

   Tone: courteous, factual, under-claimed. Do **not** speculate about exploitability, impact, or affected users. Do **not** include working request samples in the initial email — offer them on follow-up over a channel the vendor specifies.

4. **Sanity-check the draft.** Before showing it to the user:
   - No raw secrets in the email body.
   - No claim of exploitation or impact beyond what was directly observed.
   - No demand for compensation, deadline, or threat to publish.
   - User's name and contact are filled in (from `config.json` `user_contact_for_disclosures` if set, otherwise prompt).

5. **Save a record.** Write the disclosure into the maps repo (or the data dir if no repo is registered):
   ```
   <maps-root>/maps/<domain>/disclosures/<YYYY-MM-DD>-<short-slug>.md
   ```
   With sections: Finding, Contact identified, Email sent (draft), Response received (initially empty, user appends later), Resolution.

6. **Hand off.** Show the draft to the user and let them send it themselves from their own mail client. Do **not** auto-send. The user owns the relationship.

## What this skill does not do

- It does not probe, enumerate, or test the suspected issue. The finding must already be incidental.
- It does not negotiate with the vendor. The user does that.
- It does not promise the vendor anonymity or anything else on the user's behalf.
- It does not make the finding public. If the user later wants to publish a writeup, that's their call and their timeline — typically after the issue is fixed and with vendor coordination.

## Framing reminder

The plugin's purpose is reliable programmatic data ingest. Security findings, when they appear, are an **incidental byproduct** of normal capture activity, not a goal. This skill exists so the user can be a good citizen when something does surface — not to encourage looking for them.
