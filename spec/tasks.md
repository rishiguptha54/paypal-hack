# IntentLock — Task Breakdown

> Ordered by dependency. `[P]` = can run in parallel with the previous task. Role letters from [plan.md §2](plan.md#2-roles-merge-if-the-team-is-smaller). "Verify" = test IDs from [test-plan.md](test-plan.md) or an explicit check. Give Copilot **one task at a time** with its row and the linked spec sections.

## Phase 0 — Foundation (26–27 Oct)

| ID | Task | Role | Depends | Verify |
|---|---|---|---|---|
| T000 | **Admin (day 1):** every member does the employer/IP check; all register on Devpost and join one team (pick the Representative); join the PayPal Discord; get a Channel3 key + promo `PAYPAL-HACKATHON-2026`; claim Render credits; choose an AI provider and set a hard spend cap; watch the PayPal and AG Grid webinars | All | — | [submission.md §2](submission.md#2-compliance-matrix-all-must-be--before-submitting) rows C15–C16 |
| T001 | Create public repo, MIT `LICENSE`, `.gitignore` (incl. `.env*` except `.env.example`), README skeleton | D | — | License shows in GitHub About |
| T002 | npm workspaces: `apps/web`, `apps/api`, `packages/shared`; strict `tsconfig` base; ESLint + Prettier | D | T001 | `npm run typecheck` passes |
| T003 | `apps/api`: Express 5 server, `config.ts` (Zod env, fails fast, `PAYPAL_ENV=sandbox` guard), pino with redaction, requestId, error envelope, helmet, JSON-only + 16 KB limit | B | T002 | UNIT-CFG-01, SEC-02, SEC-11, SEC-12 |
| T004 | `docker-compose.yml` (Postgres 16 + api), Drizzle setup, first migration (`sessions`), `/api/health` with DB ping | D | T003 | Health 200 on two machines |
| T005 | Signed anonymous session middleware (`il_sid`) | B | T004 | Cookie flags asserted |
| T006 | `apps/web`: Vite + React + TS shell, typed API client, `/api/config` fetch; Express serves built SPA in prod | C | T002 | Page loads from API origin |
| T007 | GitHub Actions: install, typecheck, lint, unit, integration (Postgres service), gitleaks, `npm audit --omit=dev` | D | T004 | CI green |
| T008 | `.env.example` with `<REPLACE_ME>` placeholders; `.github/copilot-instructions.md` from blueprint §12 | D | T003 | No real values in repo |

## Phase 1 — PayPal vertical slice (27–29 Oct) → **Gate 0**

| ID | Task | Role | Depends | Verify |
|---|---|---|---|---|
| T010 | Sandbox app + business/buyer accounts; record (privately) client ID/secret; answer Q2 (SDK request-id support) | A | — | Can obtain OAuth token |
| T011 | `adapters/paypal`: OAuth token cache, `createOrder`, `getOrder`, `captureOrder` with `PayPal-Request-Id`, 15 s timeout, `paypal-debug-id` capture, typed response mapping (Zod) | A | T003, T010 | Unit tests with mocked fetch |
| T012 | Migration: `purchases` (minimal), `payment_operations`; temporary route to create $1.00 purchase | A | T004 | — |
| T013 | Web: PayPal **JS SDK v6** (`createInstance` → `findEligibleMethods` → `createPayPalOneTimePaymentSession`; `createOrder` resolves `{orderId}` from `paypal-order` route; `onApprove` calls `capture`); answer Q6 (client ID vs token, add route 20 if needed); CSP allows `www.sandbox.paypal.com/web-sdk/v6/core` | C | T006, T011 | Popup opens; no CSP errors |
| T014 | Live sandbox capture end-to-end; cancel path | A+C | T011–T013 | **G0**: stored status `COMPLETED` = PayPal GET |

## Phase 2 — Pure core, test-first (27–30 Oct) [P with Phase 1]

| ID | Task | Role | Depends | Verify |
|---|---|---|---|---|
| T020 | `packages/shared`: Zod schemas (`MandateFields`, `OfferSnapshot`, `Evaluation`, `Purchase`, `Money`), `reasonCodes.ts` with templates | B | T002 | Types compile in web + api |
| T021 | `core/money` (bigint minor units, strict parse/format) | B | T020 | UNIT-MONEY-01 |
| T022 | `core/normalizeOffer` (domain, https URL, price, recurrence keyword_v1) | B | T021 | UNIT-NORM-01 |
| T023 | `core/policy` rule pipeline + decision fold + `POLICY_VERSION` | B | T021, T022 | UNIT-POL-01..06 |
| T024 | `core/mandateHash` canonical serializer + golden vector | B | T020 | UNIT-HASH-01 |
| T025 | `core/stateMachine` transition table (purchase + mandate) | B | T020 | UNIT-SM-01 |

## Phase 3 — Mandate + AI (30 Oct – 1 Nov)

| ID | Task | Role | Depends | Verify |
|---|---|---|---|---|
| T030 | Migration `mandates`; `MandateService` (create, confirm, version/supersede) | B | T005, T020 | INT (create/confirm) |
| T031 | `IntentParser` interface; `RuleBasedParser` (port prototype regexes, extend) | B | T030 | UNIT-PARSE-01..05 |
| T032 | `LlmIntentParser`: structured JSON output, system prompt (user text only), 10 s timeout, Zod re-validation, fallback to rule parser on failure | B | T031 | UNIT-PARSE-06, live smoke |
| T033 | Routes 3–5 (`/mandates*`) with rate limit 10/min | B | T030–T032 | SEC-01, SEC-06 |

## Phase 4 — Discovery + evaluation (31 Oct – 2 Nov)

| ID | Task | Role | Depends | Verify |
|---|---|---|---|---|
| T040 | `ProductSource` interface; `FixtureSource` with the 6 demo offers (allowed, over-budget, recurring, unknown price, unapproved vendor, look-alike domain), using **fictional `.example` merchants and product names** (safe for video trademarks) | B | T022 | Fixture contract test |
| T041 | Channel3 spike: confirm endpoint/auth/fields from current docs (Q1); record in `research.md` | B | — | Doc updated |
| T042 | `Channel3Source`: 5 s timeout, ≤ 20 results, LRU 10 min, Zod-validated mapping, drop malformed items | B | T040, T041 | INT-17 |
| T043 | Migrations `offer_snapshots`, `evaluations`; `OfferService.search` (one transaction) + route 6 | B | T023, T042 | INT search |
| T044 | **Gate 1** API-level run in fixture mode (script in `e2e/api-flow.http` or Postman) | B+A | T014, T043 | **G1** |

## Phase 5 — Integrity (2–4 Nov)

| ID | Task | Role | Depends | Verify |
|---|---|---|---|---|
| T050 | `PurchaseService`: create (re-evaluate, compute total + hash), approve (hash echo, TTL), cancel; CAS transitions; routes 7–9, 12 | A | T023–T025, T043 | INT-02, INT-03, INT-04 |
| T051 | `paypal-order` route: get-or-create `payment_operations(CREATE_ORDER)`; `custom_id`/`invoice_id` = purchaseId | A | T050, T011 | INT-01 (create part) |
| T052 | `CaptureGate` G1–G7 + capture with stable request ID; result mapping incl. `INSTRUMENT_DECLINED`, `PENDING` | A | T051 | INT-05..11, INT-13 |
| T053 | Reconcile-on-read for `CAPTURING` > 60 s (route 8) | A | T052 | INT-16 |
| T054 | Migration `audit_events` + trigger; `AuditService.append` inside caller tx; per-session hash chain; wire all event types from [data-model.md §4](data-model.md#4-audit-event-types) | B | T030 | UNIT-AUD-01, SEC-08 |
| T055 | Strict-schema rejection auditing (`REQUEST_REJECTED`) | B | T054 | SEC-01, E2E-10 |
| T056 | Routes 13–16: ledger (cursor), receipts, chain verify | B | T054 | INT ledger |

## Phase 6 — UI (2–6 Nov)

| ID | Task | Role | Depends | Verify |
|---|---|---|---|---|
| T060 | Step header + request screen + mandate editor (clarifications, confirm disabled until valid) | C | T033 | E2E-06 |
| T061 | Offers grouped by decision; reason chips; source badge; only ALLOWED selectable | C | T043 | E2E-02..05 |
| T062 | Approval screen (server values only, TTL countdown) → JS SDK v6 payment session → verified status screen | C | T050–T052 | E2E-01, E2E-08, E2E-09, E2E-11 |
| T063 | AG Grid Community ledger: typed `colDefs`, `getRowId`, sort/filter, decision chip + reason renderers (text only), quick filter | C | T056 | E2E-12, SEC-05 |
| T064 | Intent Receipt panel + "Verify chain" button | C | T056 | E2E-12 |
| T065 | Tamper demo control (sends extra `total` field, shows server rejection) — clearly labelled as demo | C | T055 | E2E-10 |
| T066 | A11y: labels, focus order, `aria-live` status; axe in Playwright | C | T060–T064 | E2E-13 |

## Phase 6c — Governance Studio, AG Studio (sponsor track, 2–8 Nov)

| ID | Task | Role | Depends | Verify |
|---|---|---|---|---|
| T067 | Activate the AG Studio 45-day trial **on or after 1 Nov** (record the expiry); clone the official PayPal + AG Studio boilerplate as a reference; answer Q5 | C | T000 | Expiry ≥ 16 Dec |
| T068 | `GET /analytics/dataset` (route 18) from evaluations + purchases + audit | B | T056 | INT-18 |
| T069 | AG Studio page: IntentLock theme tokens, default saved layout (§17 widgets), data bound to route 18 | C | T067, T068 | Renders real session data |
| T06A | Custom **Mandate Integrity** widget (chain status from `/audit/verify`, gate-block counts G1–G7) | C | T069 | E2E-14 |
| T06B | `POST /studio-agent/turn` (route 19): SSE AG-UI events, tool-name allow-list, 8-turn cap, rate limit, kill switch | B | T032 | INT-19, INT-20 |
| T06C | Governance Analyst agent via `clientToolRunner`, delegating to Studio built-in agents; 3 tested example prompts | C | T06B, T069 | E2E-14 (LLM mocked) + live check |

## Delivery track (3–8 Nov)

| ID | Task | Role | Depends | Verify |
|---|---|---|---|---|
| T070 | Multi-stage `Dockerfile` (non-root), `render.yaml` (web service + Postgres, pre-deploy migrate, health path) | D | T004 | Deploy succeeds |
| T071 | Seed script for a demo session (optional reset button disabled in prod) | D | T054 | — |
| T072 | Playwright suite E2E-01..13 in fixture mode with MSW PayPal mock; run in CI | D | T060–T066 | **G2** |
| T073 | Live smoke on hosted URL (test-plan §6), record evidence; **record the backup video cut** | D+A | T070, T072 | **G2** |
| T074 | Webhook route (raw body), self-verification, cert host pin, dedupe, transitions; subscribe 6 events | A | T052 | INT-14, INT-15, SEC-07 (P1) |
| T075 | OpenAPI generated from Zod (`zod-to-openapi`) + CI drift check; Postman collection export | D | T056 | FR-18 (P1) |
| T076 | Security sweep: SEC-03, 04, 09, 10, 13 (fuzz) | D | T072 | SEC-* |

## Phase 8 — Release (9–11 Nov)

| ID | Task | Role | Depends | Verify |
|---|---|---|---|---|
| T080 | Feature freeze; triage only P0 bugs | All | — | — |
| T081 | README final (setup ≤ 5 commands, **Judge access** with known-good scenario + expected outcomes, criteria→evidence map, limitations, sponsor usage, AG Studio trial note), `TESTING.md` from last CI run, `THIRD_PARTY_NOTICES.md` | D | T072 | — |
| T082 | Clean-clone dry run by a non-author + judge-eye run in a private window | D | T081 | **G3** |
| T083 | Record + edit video per [plan.md §7](plan.md#7-demo-video-script-under-3-minutes) and [submission.md §6](submission.md#6-video-production-checklist); captions; frame-check for secrets/trademarks; upload **Public** | All | T073 | < 3:00, plays logged-out |
| T084 | Devpost: story, **Tools** table, built-with tags, 5–7 gallery images, links; all team members attached; tag `v1.0.0-submission` and link the tag | Lead | T082, T083 | Submitted by 11 Nov 18:00 IST |

## After submission (13 Nov – 21 Dec)

| ID | Task | Role | Depends | Verify |
|---|---|---|---|---|
| T090 | Freeze `main` at the submission tag; further work on a branch only | Lead | T084 | Branch protection on |
| T091 | By 30 Nov: Render web + Postgres on paid plans (credits); uptime monitor with 2 alert recipients | D | T070 | No cold start; DB not expiring |
| T092 | Live smoke on 1 Dec and ~8 Dec; daily health glance until 15 Dec | D | T091 | Logged results |
| T093 | After 21 Dec: rotate all keys and the judge sandbox password; downgrade paid plans | D | — | — |

## Copilot prompt template for a task

```text
Implement task <ID> from spec/tasks.md only.
Read: spec/architecture.md §<n>, spec/data-model.md §<n>, spec/contracts/api.md §<n>.
Constraints: follow .github/copilot-instructions.md; money as bigint minor units; strict Zod schemas;
no client-trusted amounts; no new dependencies without stating why.
Deliver: code + tests <test IDs>. Run typecheck and tests; report exact commands and results.
If an external API detail is unverified, stop and list the question instead of guessing.
```
