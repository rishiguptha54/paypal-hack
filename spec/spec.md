# IntentLock — Product Specification

> Defines **what** IntentLock must do. Implementation lives in [architecture.md](architecture.md). Every requirement has an ID that tests ([test-plan.md](test-plan.md)) and tasks ([tasks.md](tasks.md)) reference.

## 1. Problem

AI shopping agents can find products quickly, but a recommendation is not an authorization. Between the user's request and the payment, the order can silently drift: a higher total, an unapproved seller, a bundled subscription, a different quantity. Existing agentic-commerce rails (PayPal Agent Ready, ACP, UCP) are **merchant-side** integrations; the buyer still needs an independent, inspectable checkpoint that proves the final order matches what they agreed to.

## 2. Target user

**Primary:** a small-business owner or team lead who delegates routine procurement research (office supplies, equipment) but must keep spend inside explicit limits and show *why* a purchase was made.

**Secondary (judges/reviewers):** need to reproduce the flow from a clean clone and verify the guardrail is real, not scripted.

## 3. Value proposition

IntentLock converts a natural-language request into a **purchase mandate**, evaluates real product offers against it with deterministic rules, and only lets a PayPal Sandbox order be captured if the approved order still matches the mandate. Every step produces an audit record.

**Differentiator (stated as hypothesis, not as absolute novelty):** binding a user-confirmed mandate to a specific PayPal order via a hash, and re-verifying it against PayPal's own order record immediately before capture.

## 4. Glossary

| Term | Meaning |
|---|---|
| **Request** | The raw natural-language text the user types. |
| **Draft mandate** | AI-extracted, schema-validated constraints. Editable. Not yet authoritative. |
| **Mandate** | Draft confirmed by the user. Immutable after confirmation (edit = new mandate version). |
| **Offer snapshot** | Normalized, server-stored copy of the product/offer fields used for a decision. |
| **Evaluation** | Output of the pure policy function: `ALLOWED` / `BLOCKED` / `NEEDS_REVIEW` + findings. |
| **Purchase** | One selected offer + quantity + server-computed total, moving through the payment state machine. |
| **Mandate hash** | SHA-256 over canonical JSON of protected fields; locks the approval to the exact order. |
| **Intent Receipt** | Human-readable audit view linking request → mandate → evaluation → approval → PayPal result. |

## 5. User stories and acceptance criteria

Format: *Given / When / Then*. Priority: **P0** = required for submission, **P1** = should ship, **P2** = only if time allows.

### US-1 Describe a purchase (P0)
As a buyer, I type what I want in plain English so I don't have to fill a long form.
- **AC-1.1** Given a request containing item, quantity and maximum total, when I submit, then I see a draft mandate with each field populated and editable within 10 s (NFR-2).
- **AC-1.2** Given a request missing quantity or maximum total, then the draft is marked `NEEDS_CLARIFICATION` with a specific question per missing field; nothing is guessed.
- **AC-1.3** Given the request implies no recurring commitments ("no subscriptions"), then `recurringAllowed = false`. If not mentioned, default is `false` (fail closed).
- **AC-1.4** Given the AI provider fails or returns schema-invalid output, then I see a safe error and can fill the form manually; no partial data reaches policy.

### US-2 Confirm the mandate (P0)
As a buyer, I review and confirm the rules so that the agent is bound by what I approved, not by what the AI guessed.
- **AC-2.1** Confirm is disabled until all required fields are valid (quantity 1–100, max total > 0, currency `USD`, ≥1 approved merchant domain).
- **AC-2.2** After confirmation the mandate is immutable; editing creates a new version and invalidates previous evaluations.

### US-3 Discover and evaluate offers (P0)
As a buyer, I see real product offers sorted into Allowed / Blocked / Needs review with exact reasons.
- **AC-3.1** Offers come from Channel3 when `PRODUCT_SOURCE=channel3`; otherwise from fixtures. The UI shows a visible **source badge** (`Live: Channel3` / `Fixture`).
- **AC-3.2** Each offer shows decision + reason codes + one-line human explanation (codes in [contracts/api.md §5](contracts/api.md#5-policy-reason-codes)).
- **AC-3.3** Only `ALLOWED` offers are selectable. `NEEDS_REVIEW` and `BLOCKED` are not purchasable in the MVP.
- **AC-3.4** Every evaluation (including blocks) is written to the audit log.

### US-4 Approve a specific order (P0)
As a buyer, I approve one exact order (item, seller, quantity, total, currency) before any payment is created.
- **AC-4.1** The approval screen shows **server-computed** values only.
- **AC-4.2** Approval records `mandateHash`, `approvedAt`, `approvalExpiresAt` (default 15 min).
- **AC-4.3** A request that includes any client-supplied amount/total/decision field is rejected with `400 UNEXPECTED_FIELD` and audited (tamper demo).

### US-5 Pay with PayPal Sandbox (P0)
As a buyer, I complete payment in PayPal's own checkout.
- **AC-5.1** The PayPal order is created server-side with the stored total, `custom_id = purchaseId`, `invoice_id = purchaseId`.
- **AC-5.2** Before capture the server: re-reads the PayPal order, verifies status `APPROVED`, amount, currency and `custom_id`; recomputes the mandate hash; re-runs policy; checks expiry. Any mismatch → no capture, state `BLOCKED_AT_CAPTURE` with reason.
- **AC-5.3** Capture is idempotent: double-click, retry, or replay yields at most one capture.
- **AC-5.4** Success is shown only when PayPal returns capture status `COMPLETED`. `PENDING` is shown as pending. A redirect alone never means success.
- **AC-5.5** Cancel in PayPal → state `CANCELLED`, no capture.

### US-6 Inspect the audit ledger (P0 for AG Grid sponsor focus)
As a buyer/reviewer, I can sort, filter and open any decision to see its full evidence.
- **AC-6.1** AG Grid Community ledger with columns: time, product, merchant, decision, reason codes, approval, PayPal order ID, capture status, source.
- **AC-6.2** Clicking a row opens the Intent Receipt (request, mandate, snapshot, findings, approval, PayPal IDs, hash-chain verification result).
- **AC-6.3** Records persist across server restart/redeploy.
- **AC-6.4** A "Verify chain" action recomputes the audit hash chain and reports `INTACT` or the first broken event.

### US-7 Reconcile asynchronous PayPal results (P1)
- **AC-7.1** A signature-verified webhook for `PAYMENT.CAPTURE.COMPLETED` / `DENIED` / `CHECKOUT.PAYMENT-APPROVAL.REVERSED` updates a matching purchase; unverifiable or unknown events are stored and ignored.

### US-8 Reproduce as a judge (P0)
- **AC-8.1** From a clean clone: `docker compose up` + `.env` from `.env.example` + documented sandbox credentials → full flow works.
- **AC-8.2** A hosted demo URL is available (Render) with the same behavior **from submission until 15 Dec 2026** (end of judging).
- **AC-8.3** README has a "Judge access" section with a known-good scenario and its expected outcomes ([submission.md §5](submission.md#5-judge-access-goes-in-readme-and-devpost-testing-instructions)).

### US-9 See governance at a glance (P1 — required for the AG Grid sponsor prize)
As a business owner, I want a dashboard and an assistant that show whether my agent stays inside policy.
- **AC-9.1** An AG Studio dashboard with an IntentLock theme loads a default layout built from my session's real audit data ([architecture.md §17](architecture.md#17-governance-studio-ag-studio)).
- **AC-9.2** A custom **Mandate Integrity** widget shows the chain status and capture-gate block counts by check.
- **AC-9.3** I can ask the Governance Analyst agent a question in plain English, and it adds or changes widgets. It cannot change purchases or payments.
- **AC-9.4** All figures come from stored events; no projected or invented metrics.

## 6. Functional requirements

| ID | Requirement | Story | Pri |
|---|---|---|---|
| FR-01 | Accept request text 1–1 000 chars; reject control characters except newline. | US-1 | P0 |
| FR-02 | Extract draft mandate via AI adapter returning JSON validated by the shared Zod schema. | US-1 | P0 |
| FR-03 | Produce `clarificationQuestions[]` for missing required fields; never default quantity or budget. | US-1 | P0 |
| FR-04 | Allow manual edit and explicit confirmation; confirmed mandate is immutable and versioned. | US-2 | P0 |
| FR-05 | Search products via server-side Channel3 adapter (timeout, bounded results, cache); fixture adapter behind the same interface. | US-3 | P0 |
| FR-06 | Normalize offers: merchant domain (lowercase, strip `www.`), price → integer minor units, currency, recurrence signal. | US-3 | P0 |
| FR-07 | Evaluate each offer with a pure, versioned policy function; persist findings and `policyVersion`. | US-3 | P0 |
| FR-08 | Create purchase from (mandate, offer) with server-computed total; compute `mandateHash`. | US-4 | P0 |
| FR-09 | Record explicit human approval with TTL; reject strict-schema violations. | US-4 | P0 |
| FR-10 | Create PayPal order (intent `CAPTURE`) only from an `APPROVED` purchase with a stable `PayPal-Request-Id`. | US-5 | P0 |
| FR-11 | Pre-capture verification gate (AC-5.2) then capture with stable `PayPal-Request-Id`; persist capture ID and status. | US-5 | P0 |
| FR-12 | Enforce state transitions server-side using compare-and-set; reject illegal transitions with `409`. | US-4/5 | P0 |
| FR-13 | Append-only, hash-chained audit events for every decision and state change. | US-6 | P0 |
| FR-14 | Audit list API with cursor pagination and filters; receipt API. | US-6 | P0 |
| FR-15 | Webhook listener with PayPal signature verification and event-ID deduplication. | US-7 | P1 |
| FR-16 | Anonymous session isolation: each browser session only sees its own records. | All | P0 |
| FR-17 | Health endpoint without internal details. | US-8 | P0 |
| FR-18 | OpenAPI document generated/maintained for all routes; Postman collection exported. | US-8 | P1 |
| FR-19 | Read-only, session-scoped analytics dataset endpoint for AG Studio. | US-9 | P1 |
| FR-20 | AG Studio agent turns proxied through the server (`clientToolRunner`); tool allow-list; kill switch. | US-9 | P1 |

## 7. Non-functional requirements

| ID | Category | Requirement | How verified |
|---|---|---|---|
| NFR-1 | Correctness | Policy function is deterministic: same inputs + `policyVersion` ⇒ same output. | Property-based test |
| NFR-2 | Latency | Draft mandate p95 ≤ 10 s; search+evaluate p95 ≤ 5 s (cache warm ≤ 1 s); capture round-trip bounded by 15 s timeout. | Timed log fields in demo run |
| NFR-3 | Money | No floating-point arithmetic on amounts; integer minor units; USD only in MVP. | Lint rule + unit tests |
| NFR-4 | Idempotency | Duplicate create/capture/webhook ⇒ at most one side effect. | Integration tests (concurrent) |
| NFR-5 | Security | No secrets in client bundle, repo, logs; OWASP headers; rate limits; strict input schemas. See [security.md](security.md). | CI secret scan + header test |
| NFR-6 | Durability | Audit and purchase records survive restart/redeploy (managed Postgres). | Manual redeploy check |
| NFR-7 | Accessibility | Keyboard-operable flow; labelled inputs; status announced via `aria-live`. | Playwright + axe check |
| NFR-8 | Observability | Structured JSON logs with `requestId`, `purchaseId`, `paypalDebugId`; secret redaction. | Log snapshot test |
| NFR-9 | Honesty | All sandbox/fixture data labelled in UI and receipts. | E2E assertion on badges |
| NFR-10 | Reproducibility | Clean clone → running app with ≤ 5 documented commands. | Teammate dry-run (Gate 3) |
| NFR-11 | Availability | Hosted demo reachable without cold start from submission until 15 Dec 2026. | Uptime monitor history |

## 8. Success criteria (what we report — measured, not projected)

| Criterion | Evidence produced |
|---|---|
| Policy correctness | N/N policy unit + property tests pass (report actual counts). |
| Integrity | All tamper/drift/expiry/replay scenarios in [test-plan.md §4](test-plan.md#4-mandatory-scenarios) rejected. |
| Payment truth | For every captured purchase, stored status equals PayPal `GET order` status. |
| Reproducibility | Second teammate completes AC-8.1 from README without help. |

No market size, savings, accuracy or conversion figures are claimed.

## 9. Out of scope

See [README.md](README.md#scope-fence-do-not-build). Any addition requires an explicit change to this file.
