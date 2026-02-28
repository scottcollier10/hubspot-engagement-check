# HubSpot Engagement Intelligence Engine

> A suite of 5 coordinated n8n workflows that score, monitor, and alert on contact engagement — running automatically against your HubSpot CRM every night.

---

## What It Does

Most CRM systems tell you *who* is in your database. This system tells you *who's actually paying attention* — and who's quietly going dark before you've noticed.

The Engagement Intelligence Engine is not a single workflow. It's an architecture: five coordinated n8n workflows that together handle real-time event routing, nightly engagement scoring, on-demand health reporting, re-engagement alerting, and demo seeding. Each workflow has one job. Together they form a complete engagement monitoring system built directly on top of HubSpot's native data.

The system reads HubSpot's native email engagement signals — sends, opens, clicks, opt-outs — calculates a 0–100 score per contact, writes that score to canonical custom fields, and then makes those scores actionable through Slack. One command gives you a graded HTML report of your entire contact base. One webhook fires the moment a cold contact comes back to life.

---

## Architecture

```
                         ┌─────────────────────────────────┐
                         │     HubSpot Webhook Events       │
                         │  (contact property changes)      │
                         └────────────────┬────────────────┘
                                          │ POST /hubspot-router
                                          ▼
                         ┌─────────────────────────────────┐
                         │    1. WEBHOOK ROUTER v1.1        │
                         │                                  │
                         │  Ack HubSpot → 200 OK            │
                         │  Classify events by property     │
                         │  Forward to correct destination  │
                         └───────┬──────────────┬──────────┘
                                 │              │
              jobtitle, company, │              │ canon_engagement_status
              industry, etc.     │              │ (property changed)
                                 │              │
                                 ▼              ▼
                    ┌────────────────┐  ┌──────────────────────────┐
                    │  Normalization │  │  2. RE-ENGAGEMENT         │
                    │  Engine        │  │     ALERT v1.5            │
                    │  (separate     │  │                           │
                    │   repo)        │  │  Parse & filter event     │
                    └────────────────┘  │  Fetch contact from HS    │
                                        │  Build Slack alert        │
                                        │  Post to #marketing       │
                                        └──────────────────────────┘

  ┌──────────────────────────────────────────────────────────────────┐
  │                  3. NIGHTLY SCORER v1.0                          │
  │                                                                  │
  │  Schedule: 2:00am daily (cron: 0 2 * * *)                        │
  │                                                                  │
  │  Fetch all contacts with email                                   │
  │    → SplitInBatches (1 at a time)                                │
  │       → Read native HS fields (read-only)                        │
  │       → Calculate engagement score (0–100)                       │
  │       → Classify status (Active / At Risk / Cold / Dormant)      │
  │       → PATCH canonical fields back to HubSpot                   │
  │    → Summarize run                                               │
  │    → Post completion digest to Slack #marketing                  │
  └──────────────────────────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────────────────────┐
  │                 4. ENGAGEMENT CHECK v1.1                         │
  │                                                                  │
  │  Trigger: POST /engagement-check  (Slack slash command)          │
  │                                                                  │
  │  Ack Slack → "Running check..." (immediate)                      │
  │  Search HubSpot contacts by canon_engagement_status              │
  │  Read canonical fields, build segment counts                     │
  │  Generate graded HTML report (A–F grade, color-coded table)      │
  │  Post summary message to #marketing                              │
  │  Upload full HTML report as Slack file attachment                │
  └──────────────────────────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────────────────────┐
  │                    5. DEMO SEED v1.1                             │
  │                                                                  │
  │  Trigger: Manual                                                 │
  │                                                                  │
  │  Fetch first 20 contacts with email                              │
  │  Assign realistic engagement profiles                            │
  │  PATCH canon_ fields directly (bypasses read-only HS fields)     │
  │  Log distribution: Active / At Risk / Cold / Dormant             │
  └──────────────────────────────────────────────────────────────────┘
```

---

## Why Canonical Fields?

HubSpot's native email engagement fields (`hs_email_open`, `hs_email_click`, `hs_email_sends_since_last_engagement`) are **read-only**. HubSpot calculates them internally from its email send and tracking infrastructure. You can read them. You cannot write to them, seed them, or reset them.

This creates two problems for a scoring system:

1. **Demo and testing is impossible.** You can't put sample data into read-only fields, which means any demo environment shows zeros until real email campaigns run through it.
2. **The score can't be persisted.** Calculating a score at runtime is ephemeral — it's gone the moment the workflow ends. Segmenting, filtering, and reporting on scores requires them to live in writable HubSpot fields.

The solution is a canonical field layer: a set of custom HubSpot properties (`canon_*`) that we own completely. The Nightly Scorer reads native fields, calculates scores, and writes results to canonical fields. Everything downstream — the Engagement Check report, the Re-engagement Alert, the Demo Seed — reads only from canonical fields. This means:

- Scores persist between runs and are queryable from HubSpot itself
- Demo mode works without touching any read-only data
- The system degrades gracefully: if HubSpot native data is sparse, the score reflects that honestly
- Canonical fields are subscribable as webhook triggers (native fields like `hs_email_last_open_date` are not available as webhook triggers on HubSpot Starter)

---

## Workflow Reference

| Workflow | File | Trigger | Purpose | Key Output |
|---|---|---|---|---|
| Webhook Router | `hubspot-webhook-router-v1.1.json` | HubSpot webhook POST | Classify and forward all HubSpot property-change events to the correct downstream workflow | Forwards to Normalization Engine or Re-engagement Alert |
| Nightly Scorer | `hubspot-engagement-nightly-scorer-v1.0.json` | Cron `0 2 * * *` | Read native HubSpot email signals, calculate engagement score per contact, write to canonical fields | `canon_engagement_score`, `canon_engagement_status`, `canon_days_since_engagement`, `canon_engagement_last_calculated` |
| Engagement Check | `hubspot-engagement-check-v1.1.json` | Slack slash command (`/engagement-check`) | Query all contacts with canonical scores, generate graded HTML report, post to Slack | HTML report file + summary message in #marketing |
| Re-engagement Alert | `hubspot-reengagement-alert-v1.5.json` | HubSpot webhook (via Router) on `canon_engagement_status` change | Detect Cold/Dormant contacts that have become Active, post immediate Slack alert | Slack alert with contact name, score, silence duration, HubSpot link |
| Demo Seed | `hubspot-engagement-seed-v1.1.json` | Manual | Write realistic engagement profiles to canonical fields across 20 contacts for demo/testing | Canon fields updated; ready for `/engagement-check` |

---

## Prerequisites

**HubSpot:**
- A HubSpot account (Starter or above) with API access
- A private app token with scopes: `crm.objects.contacts.read`, `crm.objects.contacts.write`
- The following custom contact properties created (type: single-line text unless noted):
  - `canon_engagement_score`
  - `canon_engagement_status`
  - `canon_days_since_engagement`
  - `canon_engagement_last_calculated`
- A webhook subscription configured in HubSpot pointing to the Router URL, subscribed to:
  - `Contact property changed → canon_engagement_status`
  - Any normalization properties you want to route (jobtitle, company, industry, etc.)

**n8n:**
- Self-hosted n8n instance (tested on v2.7.4)
- Three credentials configured:
  - `HubSpot Data Normalization Engine Auth` — HTTP Header Auth with `Authorization: Bearer YOUR_PRIVATE_APP_TOKEN`
  - `Slack account` — Slack OAuth2 (for posting messages and files)
  - `Slack Key` — HTTP Header Auth with `Authorization: Bearer YOUR_SLACK_BOT_TOKEN` (for file upload API calls)

**Slack:**
- A bot with scopes: `chat:write`, `files:write`, `channels:read`
- The bot invited to your target channel (default: `#marketing`, channel ID `C0AEH1BS9DH`)
- Slash command `/engagement-check` configured to POST to your Engagement Check webhook URL

---

## Quick Start

Import workflows in this order — the Router must be active before HubSpot starts routing events, and the Scorer must run at least once before the Engagement Check has real data to report on.

**Step 1 — Import and activate the Webhook Router**
```
Import: hubspot-webhook-router-v1.1.json
```
After import, open the `Route Events` Code node and replace both placeholder URLs:
```javascript
const NORMALIZATION_WEBHOOK_URL = 'PASTE_NORMALIZATION_WEBHOOK_URL_HERE';
const REENGAGEMENT_WEBHOOK_URL  = 'PASTE_REENGAGEMENT_WEBHOOK_URL_HERE';
```
Activate the workflow, copy the webhook URL, and paste it as the Target URL in your HubSpot webhook subscription settings.

**Step 2 — Import the Re-engagement Alert**
```
Import: hubspot-reengagement-alert-v1.5.json
```
Copy the webhook URL from the `Webhook: Receive Event` node. Paste it into `REENGAGEMENT_WEBHOOK_URL` in the Router's `Route Events` node.

> ⚠️ **SplitInBatches wiring note:** n8n sometimes drops SplitInBatches connections on import. After importing any workflow that uses SplitInBatches, verify that:
> - The **loop output** (top) connects to the processing node (Calculate Score / Update Contact)
> - The **done output** (bottom) connects to the completion node (Summarize Run / Seed Complete)

**Step 3 — Import the Nightly Scorer**
```
Import: hubspot-engagement-nightly-scorer-v1.0.json
```
Activate the workflow. It will run automatically at 2:00am. To run it immediately for the first time, use the manual execution trigger in n8n.

**Step 4 — Import the Engagement Check**
```
Import: hubspot-engagement-check-v1.1.json
```
Copy the webhook URL and configure your Slack slash command to POST to it. Test by typing `/engagement-check` in your Slack channel.

**Step 5 (Optional) — Import the Demo Seed**
```
Import: hubspot-engagement-seed-v1.1.json
```
For demo environments where you want data before real email campaigns have run, execute this manually. It writes realistic profiles across your first 20 contacts with email addresses, then `/engagement-check` will immediately show a populated, graded report.

---

## Canonical Fields

These are the custom HubSpot contact properties the system writes. All must be created in HubSpot before the Scorer or Seed can run.

| Field Name | Type | Written By | Read By | Description |
|---|---|---|---|---|
| `canon_engagement_score` | Text (0–100) | Nightly Scorer, Demo Seed | Engagement Check, Re-engagement Alert | Calculated engagement score. Higher = more engaged. |
| `canon_engagement_status` | Text | Nightly Scorer, Demo Seed | Engagement Check, Re-engagement Alert, Webhook Router | Classified status: Active / At Risk / Cold / Dormant / Opted Out |
| `canon_days_since_engagement` | Text (integer) | Nightly Scorer, Demo Seed | Engagement Check, Re-engagement Alert | Days since the contact last opened or clicked an email |
| `canon_engagement_last_calculated` | Text (ISO timestamp) | Nightly Scorer, Demo Seed | Engagement Check | When the score was last written. Helps identify stale records. |

> These fields complement but do not replace canonical fields written by the Data Normalization Engine (`canon_seniority`, `canon_persona`, `canon_industry_tier`, `canon_data_confidence`). The Engagement Check report surfaces normalization fields alongside engagement data for a complete contact picture.

---

## Scoring Algorithm

The Nightly Scorer reads six native HubSpot fields per contact and produces a 0–100 engagement score. All native fields are **read-only** — the scorer never writes to them.

**Native fields read:**

| Field | Description | Role in Scoring |
|---|---|---|
| `hs_email_sends_since_last_engagement` | Emails sent since the contact last opened or clicked | Primary decay signal |
| `hs_email_open` | Total historical opens | Historical depth bonus |
| `hs_email_click` | Total historical clicks | Historical depth bonus |
| `hs_email_optout` | Boolean: contact has opted out | Hard penalty |
| `hs_email_last_open_date` | Date of most recent open | Used to calculate `days_since_engagement` |
| `hs_email_last_click_date` | Date of most recent click | Used to calculate `days_since_engagement` (preferred over open date) |

**Scoring formula:**

```
score = 100
score -= min(sends_since_last * 12, 80)    // primary decay: -12 per send, capped at -80
score += min(total_opens  *  3, 25)         // historical depth bonus, capped at +25
score += min(total_clicks *  5, 25)         // historical depth bonus, capped at +25
score -= 50  (if opted out)                 // hard opt-out penalty
score = clamp(score, 0, 100)
```

**Status classification** (based on `sends_since_last_engagement`):

| Status | Threshold | Meaning |
|---|---|---|
| `Active` | < 3 sends | Engaging normally |
| `At Risk` | 3–4 sends | Starting to go quiet |
| `Cold` | 5–7 sends | Clear non-engagement pattern |
| `Dormant` | ≥ 8 sends | Essentially unresponsive |
| `Opted Out` | `hs_email_optout = true` | Unsubscribed — hard stop |

**Rationale for `sends_since_last_engagement` as the primary signal:**
This field is HubSpot's own decay counter. It increments on every email send and resets to zero when the contact opens or clicks. It captures *decay rate* rather than just recency, which means a contact who opens every 3rd email looks healthier than one who last opened 15 emails ago but has never clicked. Historical open/click bonuses prevent penalizing contacts who have a strong historical track record but are in a natural quiet period.

**Report grading:**

| Grade | Condition |
|---|---|
| A — Healthy | Avg score ≥ 80 and < 10% flagged |
| B — Good | Avg score ≥ 65 and < 25% flagged |
| C — Needs Attention | Avg score ≥ 50 and < 40% flagged |
| D — At Risk | Avg score ≥ 35 |
| F — Critical | Avg score < 35 |

---

## Re-engagement Alert Logic

The Re-engagement Alert fires when the Webhook Router forwards a `canon_engagement_status` change event. Because the system runs on HubSpot Starter, `previousPropertyValue` is not reliably sent.

**HubSpot Starter behavior (no previousPropertyValue):**
Alert fires on any transition **to `Active`**. This accepts a small false-positive rate (new contacts entering Active for the first time) in exchange for never missing a real re-engagement event.

**HubSpot Pro+ behavior (previousPropertyValue present):**
Alert fires only when the previous status was `Cold` or `Dormant` AND the new status has lower severity:

```
Severity: Active (1) < At Risk (2) < Cold (3) < Dormant (4) < Opted Out (5)

Fires:   Dormant → Active  ✅  🔥 High signal
         Dormant → At Risk ✅  👀 Partial recovery
         Dormant → Cold    ✅  🧊 Trending up
         Cold    → Active  ✅  ⚡ Full re-engagement
         Cold    → At Risk ✅  📈 Improving
Does not fire: anything → worse status, or Active/At Risk → Active/At Risk
```

**Alert payload in Slack includes:**
- Contact name (linked to HubSpot record)
- Job title and company
- Previous and new status
- Engagement score and silence duration
- Seniority, persona, and ICP tier (from Normalization Engine canonical fields)
- Urgency note keyed to the specific transition type

---

## Demo Mode

The Demo Seed workflow exists because HubSpot's native email engagement fields are read-only. You cannot inject test data into `hs_email_sends_since_last_engagement` or `hs_email_open` — HubSpot controls those entirely. This means a fresh HubSpot portal will show all zeros in any report that reads native fields.

The seed writes directly to canonical fields (`canon_engagement_score`, `canon_engagement_status`, etc.), which the system owns and which are fully writable. Since the Engagement Check reads only canonical fields, the demo report looks exactly the same as a production report — the underlying data source is just seeded rather than scored.

**Seed distribution across 20 contacts:**

| Status | Count | Score Range | Days Since Engagement |
|---|---|---|---|
| Active | 6 | 76–95 | 1–12 days |
| At Risk | 5 | 47–61 | 22–42 days |
| Cold | 5 | 25–41 | 58–90 days |
| Dormant | 4 | 5–18 | 145–290 days |

**To run a demo:**
1. Execute `DEMO SEED — Engagement Data Reset v1.1` manually
2. Wait ~15 seconds for all 20 contacts to update
3. Type `/engagement-check` in Slack
4. The full graded report appears in `#marketing`

To reset, run the seed again — it overwrites canonical fields on the same 20 contacts.

---

## Extending the System

**Add a new webhook route:**
1. Add the HubSpot property name to the appropriate Set in the Router's `Classify Events` node
2. Add a forward block in the `Route Events` node following the existing pattern
3. No HubSpot changes required (assuming the property is already subscribed)

**Adjust scoring thresholds:**
All thresholds are defined as named constants at the top of the `Calculate Score` node in the Nightly Scorer. Change `THRESHOLDS.AT_RISK`, `THRESHOLDS.COLD`, and `THRESHOLDS.DORMANT` without touching any other logic.

**Add pagination for large portals:**
The Nightly Scorer fetches up to 100 contacts per run. For portals with more contacts, add an `after` cursor loop using HubSpot's `paging.next.after` response field. This is documented in v1.1 as the next planned enhancement.

**Adjust the scoring weights:**
The `WEIGHTS` object at the top of `Calculate Score` controls all bonus and penalty values. Each weight is named descriptively. Changes take effect on the next nightly run.

**Connect additional downstream workflows:**
The Router's `Classify Events` node uses a simple Set-based routing table. Adding a new destination is three lines: a new Set, a new `if` check, and a new group in the reducer.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Workflow runtime | n8n (self-hosted, v2.7.4+) |
| CRM | HubSpot (Starter tier and above) |
| Alerting & reporting | Slack (OAuth2 + Bot Token) |
| HubSpot auth | Private App Token via HTTP Header Auth |
| Scoring language | JavaScript (n8n Code nodes) |
| Report format | Self-contained HTML (generated in-workflow, uploaded as Slack file) |
| Scheduling | n8n Schedule Trigger (cron `0 2 * * *`) |
| Event routing | HubSpot webhook subscriptions → n8n Webhook nodes |

---

## Related Projects

This project is part of the HubSpot Automation Suite:

- [`hubspot-data-normalization`](https://github.com/scottcollier10/hubspot-data-normalization) — Normalizes raw HubSpot contact fields into canonical forms with confidence scoring. Writes the `canon_seniority`, `canon_persona`, `canon_industry_tier`, and `canon_data_confidence` fields that the Engagement Check report surfaces alongside engagement data.
- [`content-generator`](https://github.com/scottcollier10/content-generator) — AI-powered email content generation via Slack slash command. Uses HubSpot template cloning + Claude to produce three brand-aware variations in under 20 seconds.
- [`analytics-agent`](https://github.com/scottcollier10/analytics-agent) — Autonomous ops reporting agent built with the Anthropic SDK. Decides which tools to call, in what order, and surfaces cross-system anomalies.

---

*Built by [Scott Collier](https://intelligentdesigns.io) · Intelligent Designs · Round Rock, TX*
