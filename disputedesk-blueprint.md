# DisputeDesk — PayPal AI Hackathon Blueprint

> **Status:** Final selected idea (replaces IntentLock).
> **Team:** 2 people · **Window:** 21 Oct → submit 11 Nov 2026 (official deadline 12 Nov 2026, 12:00 PM PT = 13 Nov 00:30 IST).
> **Research verified:** 21 Oct 2026 against official hackathon pages and PayPal API references (see §15).

---

## 1. Why this idea

No idea can guarantee a win: judging is subjective, and the hackathon has 17,861 registered participants. DisputeDesk gives the team the strongest position against the official judging criteria:

| Official criterion (equal weight; ties broken in this order) | Why DisputeDesk scores well |
|---|---|
| **Technological implementation** | Uses about 8 PayPal capabilities (Orders, Disputes, Webhooks, Refunds, Order Tracking, sandbox dispute simulation, Agent Toolkit/MCP). An agent uses tools, but its claims are checked by deterministic code. Not a thin wrapper around a chat model. |
| **Design** | A full seller workflow: alert → case file → review → submit → outcome, in an AG Grid console. |
| **Potential impact** | A specific, real audience — small online sellers — and a problem PayPal's own judges know well: disputes and chargebacks cost sellers money and time. |
| **Innovation** | Most teams will build shopping agents or invoicing tools, which are the examples on the official resources page. Few will build an agent that works on the **merchant's side of PayPal's dispute process**, and PayPal's dispute APIs are rarely used in hackathons. |
| **Presentation** | The complete dispute flow runs **live in the sandbox** in under 3 minutes, using sandbox-only PayPal endpoints that let us move a dispute through PayPal's review stages. |

**Prize targets:** Grand Prize · Best Use of PayPal + AI · Most Impactful, plus **one** sponsor prize: **AG Grid** (5 winners, largest sponsor prize). The rules allow at most one Grand Prize or one Honorable Mention, plus one Sponsor Prize.

---

## 2. Problem statement

> **Small online sellers lose disputes they should win.** When a PayPal buyer opens a dispute, the seller must answer within a deadline and provide evidence that matches the dispute reason. That evidence includes proof of shipment, delivery, the item listing, or customer communication. It is spread across the store database, shipping records, email and listing pages. Sellers without an operations team miss deadlines, send weak or wrong evidence, or refund by default — losing money even when they were in the right.

**Target user:** a solo or small-team online seller (1–10 people) taking PayPal payments.
**Persona for the demo:** "Maya runs *Tiny Candle Co.*, an online store with 2 staff. She gets a few disputes a month and handles each one manually, which takes hours."

**Do not invent statistics** (dispute rates, win rates, hours saved). Use only figures you measure yourselves (§10) or cite from a published source.

---

## 3. Solution in one sentence

**DisputeDesk is an AI agent that, when a PayPal dispute opens, gathers the evidence that matches the dispute reason, drafts a response with cited sources, flags missing evidence, and recommends whether to fight or refund. The seller approves, and DisputeDesk submits through PayPal's Disputes API.**

### Core loop

```
Buyer opens dispute ─► PayPal webhook ─► DisputeDesk
   1. Fetch dispute details (Disputes API) + linked order (Orders API)
   2. Look up the evidence playbook for the dispute reason
   3. Agent gathers evidence with tools: order, shipment/tracking, messages, listing snapshot
   4. Deterministic checker: required evidence present? Every claim in the draft cites real evidence?
   5. Fight-or-refund recommendation (rules + plain-English explanation)
   6. Seller reviews in console ─► Approve
   7. Submit: provide-evidence | accept-claim | make-offer | send-message
   8. Webhook/status update ─► outcome recorded ─► AG Grid ledger
```

### Agent and human boundaries

| The agent does automatically | Only the seller can approve |
|---|---|
| Classifies the dispute and opens a case | Submitting evidence to PayPal |
| Gathers and links evidence | Accepting a claim or refunding money |
| Drafts the response and explanation | Making an offer (partial refund, replacement) |
| Flags missing evidence and deadlines | Sending a message to the buyer |
| Suggests prevention steps, such as adding tracking | Any action that moves money or is irreversible |

---

## 4. Feature scope (two-person team)

### P0 — required for submission
1. **Demo shop** ("Tiny Candle Co.") using PayPal Checkout in the sandbox. This produces **real sandbox orders**, which become real disputes.
2. **Shipping records** in the shop. Tracking is pushed to PayPal with the Orders v2 tracking endpoint (verify the exact endpoint).
3. **Dispute ingestion:** a webhook receiver that verifies PayPal webhook signatures, plus a polling fallback using `GET /v1/customer/disputes`.
4. **Evidence playbooks** for two dispute reasons only:
   - `MERCHANDISE_OR_SERVICE_NOT_RECEIVED`
   - `MERCHANDISE_OR_SERVICE_NOT_AS_DESCRIBED`
5. **Case-building agent** with tool calls (§6) that outputs a schema-validated case file.
6. **Citation checker:** every fact in the draft (tracking number, date, amount, address) must match an evidence item exactly; otherwise the draft is rejected.
7. **Seller review screen:** case file, draft, missing-evidence warnings, and buttons for approve, edit or accept claim.
8. **Submission** through `provide-evidence` (multipart request with files) and `accept-claim`.
9. **AG Grid Dispute Console:** a queue with deadline countdowns, status badges, grouping by reason, expandable case-file rows, and an outcomes view.
10. **Full sandbox lifecycle in the demo:** move the dispute stage with `require-evidence`, submit evidence, then use `adjudicate` to simulate PayPal's decision.
11. **Render deployment** with Postgres, a demo login, README and license.

### P1 — add only after P0 works end to end
- **Fight-or-refund advisor:** compares evidence completeness, order amount and fees, then recommends making an offer (`make-offer`) or fighting.
- **Prevention mode:** flags orders that are shipped without tracking or delayed, and pushes tracking to PayPal before a dispute happens.
- **Listing snapshot with KERNEL:** a cloud browser captures the seller's product page as `LISTING_URL` / `ITEM_DESCRIPTION` evidence for not-as-described disputes.
- **Agent Toolkit / MCP:** expose DisputeDesk's case-building tools through MCP, and use the PayPal Agent Toolkit for PayPal calls where it supports them.

### P2 — only if time remains
- **Elastic** semantic search over the customer message history.
- **Zapier** import from a real mailbox.
- Support for more dispute reasons.

---

## 5. Architecture

```
┌─────────────── client (React + TS + Vite) ───────────────┐
│  /shop     Demo storefront (PayPal JS SDK checkout)      │
│  /console  AG Grid Dispute Console · Case review · Admin │
└───────────────▲──────────────────────────────────────────┘
                │ REST/JSON (session auth)
┌───────────────┴────────────── server (Node + Express + TS) ─────────────┐
│ routes/        shop · webhooks · disputes · cases · admin(sandbox-sim)  │
│ services/paypal/   oauth · orders · tracking · disputes · webhookVerify │
│ services/agent/    caseBuilder (LLM + tools) · draftWriter              │
│ services/policy/   playbooks · completenessCheck · citationVerifier ·   │
│                    fightOrRefund (deterministic)                        │
│ services/evidence/ store (private files) · listingSnapshot (KERNEL, P1) │
│ services/audit/    append-only events                                   │
│ persistence/       Postgres (orders, shipments, messages, disputes,     │
│                    cases, evidence, approvals, audit_events)            │
└──────┬───────────────────────┬─────────────────────────┬────────────────┘
       ▼                       ▼                         ▼
 PayPal Sandbox APIs      LLM provider (adapter)     KERNEL (P1) / Elastic (P2)
```

**Stack:** TypeScript monorepo · Express · Zod · Postgres · React 18 + Vite · AG Grid (Enterprise 2-month trial if master/detail and grouping are needed; otherwise Community) · Render (Web Service + Static Site + Postgres).

### PayPal capabilities used

| Capability | Endpoint/operation | Used for |
|---|---|---|
| Checkout / Orders v2 | create + capture order | Demo shop generates real sandbox payments |
| Order tracking | Orders v2 add-tracking (verify endpoint) | Proof of shipment, prevention mode |
| Webhooks | verify-webhook-signature + dispute events (verify event names, e.g. `CUSTOMER.DISPUTE.CREATED/UPDATED/RESOLVED`) | Real-time ingestion |
| Disputes — read | List disputes, Show dispute details | Case intake |
| Disputes — act | Provide evidence, Accept claim, Make offer, Send message, Provide supporting info | Seller actions (always human-approved) |
| Disputes — sandbox only | Update dispute status (`require-evidence`), Settle dispute (`adjudicate`) | Live end-to-end demo |
| Refunds | Refund captured payment (if not done through accept-claim) | Refund path |
| Agent Toolkit / MCP (P1) | `@paypal/agent-toolkit` (v1.11.0 at time of research) | Agent tool interface |

---

## 6. AI design

### Agent tools (server-side functions; the LLM never sees credentials)

| Tool | Returns |
|---|---|
| `get_dispute(disputeId)` | Reason, amount, buyer claim text, stage, deadline |
| `get_order(orderId)` | Items, amounts, shipping address, capture IDs |
| `get_shipments(orderId)` | Carrier, tracking number, ship date, delivery status |
| `search_messages(orderId, query)` | Customer messages (P0: Postgres full-text; P2: Elastic) |
| `get_listing_snapshot(sku)` | Stored listing text/images; P1 adds a KERNEL screenshot |
| `get_playbook(reason)` | Required and optional evidence types for this reason |
| `propose_case(caseFile)` | Saves the case after schema validation |

### Evidence playbooks (deterministic, version-controlled)
- Store in `server/src/services/policy/playbooks/*.json`, one file per dispute reason. Each file maps the reason to required and optional PayPal `evidence_type` values, such as `PROOF_OF_FULFILLMENT`, `ONLINE_TRACKING_INFORMATION`, `PROOF_OF_DELIVERY_SIGNATURE`, `ITEM_DESCRIPTION`, `LISTING_URL` or `PHOTOS_OF_SHIPPED_ITEM`.
- **Base the requirements on PayPal's published Seller Protection and dispute guidance, cite the source URL inside each playbook, and have a person review it.** The LLM must not invent evidence requirements.

### Guardrails
1. **Schema validation:** agent output must pass the Zod `CaseFile` schema or the agent retries, up to two retries.
2. **Citation checker:** extract every number, date, tracking ID and amount from the draft. Each must exactly match a field in a linked evidence item. Any unmatched fact blocks submission and is highlighted in the UI.
3. **Untrusted text:** treat buyer claims and customer messages as data, not instructions. Wrap them in delimiters, and never let them choose a tool action.
4. **Deterministic completeness score:** compute the score from the playbook, not from the LLM.
5. **Money-moving and irreversible actions** require an explicit seller approval tied to a **hash of the exact submission**, covering the files, text and action type. If anything changes, approval is required again.

### Example `CaseFile` (shape)
```ts
interface CaseFile {
  disputeId: string;
  reason: "MERCHANDISE_OR_SERVICE_NOT_RECEIVED" | "MERCHANDISE_OR_SERVICE_NOT_AS_DESCRIBED";
  evidence: { id: string; evidenceType: string; source: "order"|"shipment"|"message"|"listing"; summary: string; fileRef?: string; sha256: string }[];
  missingRequired: string[];          // computed by code, not LLM
  draftNote: string;                  // seller-facing response text
  citations: { claim: string; evidenceId: string }[];
  recommendation: { action: "FIGHT"|"OFFER"|"ACCEPT"; rationale: string; ruleCodes: string[] };
}
```

---

## 7. Case state machine

```
NEW → GATHERING → READY_FOR_REVIEW → APPROVED → SUBMITTED → AWAITING_PAYPAL → RESOLVED_WON | RESOLVED_LOST
                       │                                  
                       ├→ NEEDS_INFO (missing required evidence; seller uploads)
                       ├→ CLAIM_ACCEPTED (refund)
                       └→ OFFER_MADE → OFFER_ACCEPTED | OFFER_DENIED
```
- Only the backend can change the state, and each transition is validated.
- Every transition writes an append-only audit event.
- PayPal calls send a `PayPal-Request-Id` header for idempotency, and every webhook is de-duplicated by event ID.

---

## 8. Security and privacy (non-negotiable)

- Keep PayPal client ID/secret, LLM key, and KERNEL/Elastic keys **server-side only**. Commit only `.env.example`, with values set to `<REPLACE_ME>`.
- **Verify every webhook signature** with PayPal's verification API, and reject anything that fails.
- Store evidence files privately with access control. Never use public buckets.
- Send the LLM only the minimum buyer data needed. Redact emails and phone numbers where they are not required.
- The console requires a login. Sandbox simulation endpoints (`require-evidence`, `adjudicate`) are available only from an **admin-only** route that is disabled when `PAYPAL_ENV != sandbox`.
- Apply rate limits to public routes, return consistent JSON errors, and never return stack traces.
- **Tests for security controls:** reject a webhook with a bad signature; reject an unauthenticated console request; reject a submission whose approval hash does not match; reject a draft containing an uncited fact; reject customer messages containing injection text.

---

## 9. Sponsor tools — purpose of each

| Tool | Priority | Real purpose | Prize |
|---|---|---|---|
| **AG Grid** | P0 | Dispute console: deadline countdown cells, status badges, grouping by reason or stage, master/detail case files, outcomes view | **Primary sponsor target** |
| **Render** | P0 | Hosting (web service, static site, Postgres); cron for deadline reminders and the polling fallback | Eligible (credits) |
| **KERNEL** | P1 | Cloud browser captures the seller's listing page as dated, hashed evidence | No sponsor prize |
| **Elastic** | P2 | Semantic search over customer message history | No sponsor prize |
| **Zapier** | P2 | Import messages from a real mailbox | No sponsor prize |

Choose **one** sponsor prize to focus on: AG Grid. Add other tools only when they solve a real product need.

---

## 10. Measured evaluation (honest numbers only)

Run against **10 or more real sandbox disputes** created by the team, with a mix of complete-evidence and missing-evidence cases:

| Metric | How measured |
|---|---|
| Time from webhook receipt to a case ready for review | Server timestamps |
| Required-evidence completeness | Playbook check, before and after the agent runs |
| Uncited-fact rate in drafts | Citation checker results |
| Correct fight-or-refund recommendation against the team's labelled expected outcome | Labelled test set |

**Important:** sandbox `adjudicate` is a call **we make**, so a sandbox "win" does **not** prove that DisputeDesk improves real win rates. In the video and README, say: *"We simulate PayPal's decision in the sandbox."* Do not claim improved win rates.

---

## 11. 22-day plan (two people)

**Roles.** **A — Backend/PayPal/AI:** PayPal, webhooks, agent, policy and tests. **B — Frontend/Product/Demo:** shop, console, AG Grid, deployment, README and video.

| Dates | A (backend) | B (frontend/product) | Exit test |
|---|---|---|---|
| **Oct 21–22** ⛳ **Day-1 check** | Sandbox app with **Disputes enabled**; OAuth; buyer pays; **open a dispute from the sandbox buyer account**; list it; `require-evidence` → `provide-evidence` → `adjudicate` using raw API calls | Repository scaffold, LICENSE, `.env.example`, README skeleton, health endpoint | **Pass/fail gate:** a full dispute lifecycle works in the sandbox. If it fails by Oct 23, ask in the PayPal Discord about limited-release access to the create-dispute API; if it is still blocked, **switch to PayablesGuard** |
| Oct 23–26 | Postgres schema, webhook receiver + signature verification + polling fallback, dispute and order sync | Demo shop with PayPal checkout, shipment entry, tracking push | A real purchase becomes a dispute that appears in the database automatically |
| Oct 27–30 | Playbooks (2 reasons), agent tools, `CaseFile` schema, citation checker, unit tests | AG Grid console v1 (queue, deadlines, grouping) and case review screen | The agent builds a valid case file, and the console shows it |
| Oct 31–Nov 3 | Approval hash, `provide-evidence` (multipart), `accept-claim`, state machine, idempotency, audit log | Review/approve/edit flow, evidence viewer, outcomes view | **P0 end to end:** dispute → AI case → approve → submit → adjudicate → outcome in the grid |
| Nov 4–6 | P1: fight-or-refund advisor, `make-offer`, prevention mode | Deploy on Render; P1: KERNEL listing snapshot | Hosted URL works in a clean browser |
| **Nov 7–8** | Security tests and evaluation run (§10) | README by judging criteria, test credentials for judges | **Feature freeze on Nov 8** |
| Nov 9–10 | Bug fixes only | Record and edit video (<3 min), upload publicly to YouTube | Video reviewed by both people |
| **Nov 11** | — | **Submit on Devpost** (one day early) | Submission page, repository and video links verified |
| Nov 12 | Buffer | Buffer | Official deadline is 12:00 PM PT |

**If behind schedule, cut in this order:** P2 → KERNEL → prevention mode → fight-or-refund advisor. Never cut the citation checker, approval hash or full sandbox lifecycle.

---

## 12. Demo video script (under 3:00)

| Time | Scene |
|---|---|
| 0:00–0:20 | **Hook:** "Maya just got a PayPal dispute. She has a deadline, and the evidence is spread across five places." Show the dispute notification. |
| 0:20–0:35 | Who it is for and why it matters (one sentence each). |
| 0:35–1:20 | **Live:** a dispute arrives in the AG Grid console → the agent builds the case: order, tracking, delivery, listing snapshot and customer message. Citations appear next to each claim. |
| 1:20–1:45 | **Guardrail moment:** a second case is missing tracking, so DisputeDesk **blocks submission** and asks for it. A draft with an uncited claim is rejected. |
| 1:45–2:15 | Maya approves, evidence is submitted through PayPal's Disputes API, and the **sandbox decision is simulated**: resolved in the seller's favour. The grid updates. |
| 2:15–2:35 | Fight-or-refund advisor recommends a partial-refund offer on a weak case; prevention mode flags an order shipped without tracking. |
| 2:35–3:00 | Architecture in one slide: PayPal APIs used, AI with guardrails, measured results from §10. Close: "Small sellers keep money they are owed." |

---

## 13. Submission checklist (from official rules)

- [ ] Public GitHub repository with an **open-source LICENSE visible in the About section**
- [ ] All source code, assets and complete setup/run instructions; hosted URL on Render
- [ ] Test credentials for judges (demo console login, plus sandbox buyer account instructions), with no secrets committed
- [ ] Text description with features, plus a **Tools section explaining which AI model/service is used and how**
- [ ] Video **under 3 minutes**, public on YouTube, showing the working app, with no copyrighted music or third-party trademarks without permission
- [ ] README sections mirror the 5 judging criteria (reviewers, possibly including AI-assisted review, can map content quickly)
- [ ] Sandbox clearly labelled everywhere; sandbox adjudication described as simulated

---

## 14. Risks and mitigations

| Risk | Mitigation |
|---|---|
| Opening disputes from the sandbox buyer account doesn't work (the create-dispute API is limited release) | Day-1 pass/fail gate; ask in the PayPal Discord; fall back to PayablesGuard by Oct 23 |
| `provide-evidence` multipart format details | Build a working raw API call on Day 1, then wrap it |
| Webhook delivery to localhost | Use the polling fallback; use the Render URL for webhooks |
| AG Grid Enterprise features | Use the 2-month trial; note the licence status in the README |
| Evidence requirements wrong or invented | Base playbooks on PayPal's published guidance, cite it, and have a person review them |
| LLM hallucination | Citation checker + schema validation + human approval |
| Scope creep (team of 2) | Strict P0/P1/P2 tiers; feature freeze on Nov 8 |

### Claims to avoid
- "Wins more disputes" or any win-rate figure. Sandbox decisions are simulated.
- "Fully autonomous." The seller approves every submission.
- Any invented statistic about disputes, losses or time saved.

---

## 15. Verified sources (checked 21 Oct 2026)

- Hackathon rules, prizes, judging: https://paypalaihackathon.devpost.com/rules
- Hackathon overview and requirements: https://paypalaihackathon.devpost.com/
- Hackathon resources (PayPal APIs, AI Toolkit, MCP): https://paypalaihackathon.devpost.com/resources
- PayPal Disputes API v1 (operations, sandbox-only `adjudicate` and `require-evidence`, "merchants can create disputes in the sandbox"; create-dispute method is limited release): https://developer.paypal.com/docs/api/customer-disputes/v1/
- PayPal Disputes overview: https://developer.paypal.com/disputes/overview
- PayPal authorize and capture (29-day authorization, 3-day honor period): https://developer.paypal.com/docs/checkout/standard/customize/authorization/
- PayPal Payouts API: https://developer.paypal.com/docs/api/payments.payouts-batch/v1/
- PayPal AI Toolkit (sandbox MCP: orders, invoices, subscriptions, disputes, shipment, reporting): https://github.com/paypal/AI-Toolkit
- KERNEL (cloud browsers for agents): https://www.kernel.sh/
- AG Grid (Community + 2-month Enterprise trial): https://www.ag-grid.com/react-data-grid/getting-started/
- Channel3 docs: https://docs.trychannel3.com/
- Astropods (agent governance; reason IntentLock was dropped): https://astropods.com/

**Still to verify during build:** exact webhook event names for disputes; Orders v2 tracking endpoint; `provide-evidence` file size and type limits; whether the sandbox Resolution Center can open disputes on the team's accounts.
