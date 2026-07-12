# ZB SFvalidator — Email Validation & Discovery Pipeline

A production pipeline that validates CRM email deliverability at scale across
**Salesforce**, **n8n**, and **ZeroBounce** — and, for every record that comes
back invalid or empty, infers and verifies a working address. Nothing leaves the
workflow without a stamped verdict.

> **Live overview page:** open [`index.html`](index.html) in a browser (or host it
> with GitHub Pages) for a visual walkthrough of the system.

---

## Why this exists

A CRM full of addresses is not a CRM full of reachable people. Leads and Contacts
arrive with typos, dead domains, generic inboxes, or no email at all. Sending to
them burns sender reputation and buries the prospects worth contacting.

Flagging the bad addresses is only half the job. The harder, more valuable half is
**recovering a working address** for the records that fail — by learning how an
organisation names its mailboxes and verifying a single high-confidence guess.

---

## How it works

One user action in Salesforce kicks off a fully automated run:

| # | Stage | What happens |
|---|-------|--------------|
| 01 | **Trigger** | A list-view or record action fires a secured webhook. A shared-secret header is validated before any work begins. |
| 02 | **Fetch** | The selected Lead/Contact records are pulled by ID over SOQL, with the fields the pipeline needs. |
| 03 | **Validate** | Existing addresses are checked through ZeroBounce using the **batch endpoint** (up to 100 per request), keeping call volume low and avoiding rate limits. |
| 04 | **Classify** | Each result is sorted into one of six deliverability states. |
| 05 | **Discover** | For invalid/empty records: resolve the related organisation → read verified peer addresses → infer the naming pattern → verify one high-confidence guess. |
| 06 | **Stamp** | Status, detail, suggested email, and a validation timestamp are written back to **every** record (including still-invalid ones) via the Salesforce Composite API in batches of 200. |

---

## Architecture

Three systems, each with one job and explicit boundaries.

### Salesforce — system of record + UI
- Four custom fields on Lead & Contact: **validation status**, **validation detail**, **suggested email**, **last validated**
- Apex validation service and list controllers
- Lightning Web Component quick actions + Visualforce list-view buttons for mass validation
- **Named Credential** for the outbound callout to the workflow
- **Permission Set** scoping the feature to authorised users only

### n8n — orchestration (self-hosted)
- Webhook trigger with secret-header check
- SOQL fetch, batch assembler, and ZeroBounce response mapper
- Classifier node routing the six deliverability states
- Discovery branch: resolve account → fetch peer emails → infer pattern → generate + verify guess
- Salesforce **Composite API** write-back (PATCH, batches of 200)

### ZeroBounce — deliverability oracle
- Batch validation of existing addresses (`/v2/validatebatch`)
- Single-address verification of inferred guesses (`/v2/validate`)
- Catch-all and generic-domain signals
- Used strictly as a **filter**, never as a generator

---

## Classification states

Every record lands in exactly one state. The state is what downstream teams act on.

| State | Meaning |
|-------|---------|
| **Valid** | Deliverable. Safe to use as-is. |
| **Valid — Generic Domain** | Deliverable, but a free provider (gmail, yahoo, hotmail) — flagged for context. |
| **Valid — Domain Mismatch** | Deliverable, but the domain doesn't match the account's website. |
| **Catch-all — Unverified** | Domain accepts everything; the individual mailbox can't be confirmed. |
| **Invalid — No Email Found** | Undeliverable, and discovery couldn't recover an address. |
| **Invalid — Suggestion Failed** | Undeliverable; a candidate was inferred but didn't verify. |

---

## Key engineering decisions

- **Infer before you validate.** ZeroBounce is a filter, not a generator. Submitting
  one high-probability inferred candidate matches or beats a pile of brute-forced
  permutations — at a fraction of the API volume.
- **Batch, don't hammer.** Rapid sequential calls trip the provider's firewall
  (Cloudflare error 1020). The batch endpoint collapses call volume into a single
  request and keeps the run clean.
- **Cache the pattern per domain.** Once an organisation's naming convention is
  confirmed, it's reused for future records at the same domain — no repeat calls to
  relearn what's already known.
- **No silent drops.** Every record that enters the workflow exits with a stamped
  status, including the definitively invalid ones. Silent skips create invisible
  problem records — the hardest kind to find later.
- **Handle catch-all honestly.** When a domain accepts every address, mailbox-level
  confirmation is impossible. The record is labelled for what it is, and inferred
  pattern confidence becomes the only decision signal.

---

## Screenshots

From the live deployment, in [`screenshots/`](screenshots/):

| File | Shows |
|------|-------|
| `01_n8n_workflow.png` | The full **ZB SFvalidator** workflow — trigger, batch validation, classifier, discovery branch. |
| `02_actionable_list_menu.png` | Mass actions on a Salesforce list view. |
| `03_list_view.png` | Results stamped onto records: status, detail, suggested email, timestamp. |
| `04_record_detail.png` | A single record after discovery, with the audit trail in the detail field. |
| `05_list_actions.png` | Per-record quick actions on the Lead layout. |

---

## Tech stack

`Apex` · `Lightning Web Components` · `Visualforce` · `SFDX / Salesforce CLI` ·
`Composite API` · `Named Credentials` · `Permission Sets` · `n8n (self-hosted)` ·
`ZeroBounce API` · `SOQL`

---

## Repository structure

```
.
├── index.html          # Visual overview page (GitHub Pages ready)
├── screenshots/        # Redacted screenshots from production
├── README.md
├── LICENSE
└── .gitignore
```

---

## Preview the overview page locally

```bash
# any static server works; for example:
python3 -m http.server 8000
# then open http://localhost:8000
```

To publish it: enable **GitHub Pages** on this repo (Settings → Pages → deploy from
the default branch). Then update the `REPO` constant near the bottom of `index.html`
so the "View on GitHub" links point at your repository.

---

## A note on privacy

This repository documents a system built against a private production org. To share
it publicly:

- All contact email addresses in the screenshots are **synthetic examples** — no real
  Lead or Contact addresses appear anywhere.
- Personal names have been removed from record audit trails.
- Internal endpoint URLs in the workflow screenshot are **blurred**.

No credentials, secrets, org identifiers, or real customer data are included.

---

## License

Released under the MIT License. See [`LICENSE`](LICENSE).
