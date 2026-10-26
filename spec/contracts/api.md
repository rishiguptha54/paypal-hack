# IntentLock — API Contract

> Source of truth for HTTP behavior. `docs/openapi.yaml` is **generated** from the shared Zod schemas and must match this file; CI fails on drift. Data shapes: [../data-model.md](../data-model.md). Flows: [../architecture.md §6](../architecture.md#6-key-runtime-flows).

## 1. Conventions

- Base path `/api`. JSON only: requests without `Content-Type: application/json` on non-GET routes ⇒ `415` (also blocks simple-form CSRF).
- Session: `il_sid` cookie — `HttpOnly; Secure; SameSite=Lax; Path=/`, signed. Created on first request. Every resource is scoped to it; foreign IDs ⇒ `404`.
- All request bodies are **strict** Zod objects: unknown keys ⇒ `400 UNEXPECTED_FIELD` (+ audit `REQUEST_REJECTED`).
- Money in responses: `{ "minor": "29800", "currency": "USD", "display": "$298.00" }`. `minor` is a string to avoid JS number precision issues.
- IDs are UUIDs; path params validated before any DB access.
- Every response carries `X-Request-Id`.

### Error envelope

```json
{ "error": { "code": "STATE_CONFLICT", "message": "Purchase is not awaiting approval.", "requestId": "0b6…", "details": [] } }
```

| HTTP | Codes |
|---|---|
| 400 | `VALIDATION_FAILED`, `UNEXPECTED_FIELD` |
| 401 | `SESSION_REQUIRED` |
| 404 | `NOT_FOUND` |
| 409 | `STATE_CONFLICT`, `POLICY_BLOCKED`, `CAPTURE_BLOCKED`, `MANDATE_HASH_MISMATCH`, `APPROVAL_EXPIRED` |
| 413 | `PAYLOAD_TOO_LARGE` (limit 16 KB) |
| 415 | `UNSUPPORTED_MEDIA_TYPE` |
| 422 | `NEEDS_CLARIFICATION` |
| 429 | `RATE_LIMITED` (+ `Retry-After`) |
| 502 / 504 | `UPSTREAM_ERROR` / `UPSTREAM_TIMEOUT` (`details[0].provider` = `ai`, `channel3`, `paypal`) |

## 2. Rate limits (per session, sliding 1 min; plus 300/min per IP global)

| Route group | Limit | Reason |
|---|---|---|
| `POST /mandates` | 10 | LLM cost |
| `POST /mandates/:id/search` | 20 | Channel3 quota |
| Payment routes | 20 | Abuse / accidental loops |
| `POST /studio-agent/turn` | 15 (max 8 turns per run) | LLM cost |
| Reads | 120 | |

## 3. Routes

| # | Method & path | Body | Success | Notes |
|---|---|---|---|---|
| 1 | `GET /health` | — | `200 {"status":"ok"}` | DB ping only |
| 2 | `GET /config` | — | `200 {paypalClientId, productSource, aiProvider, environment:"sandbox"}` | Public values only |
| 3 | `POST /mandates` | `{requestText}` | `201 Mandate` | Status `DRAFT` or `NEEDS_CLARIFICATION` |
| 4 | `GET /mandates/:id` | — | `200 Mandate` | |
| 5 | `POST /mandates/:id/confirm` | `MandateFields` | `200 Mandate` (CONFIRMED) | If called on a CONFIRMED mandate with different fields ⇒ new version, old ⇒ SUPERSEDED |
| 6 | `POST /mandates/:id/search` | `{}` | `200 {source, offers: EvaluatedOffer[]}` | Requires CONFIRMED |
| 7 | `POST /purchases` | `{mandateId, offerSnapshotId}` | `201 Purchase` (PENDING_APPROVAL) | Re-evaluates; `409 POLICY_BLOCKED` if not ALLOWED |
| 8 | `GET /purchases/:id` | — | `200 Purchase` | Reconciles `CAPTURING` > 60 s via PayPal GET |
| 9 | `POST /purchases/:id/approve` | `{mandateHash}` | `200 Purchase` (APPROVED) | Hash must equal server value (proves user saw this exact order) |
| 10 | `POST /purchases/:id/paypal-order` | `{}` | `200 {paypalOrderId}` | Idempotent: returns existing order if already created |
| 11 | `POST /purchases/:id/capture` | `{paypalOrderId}` | `200 Purchase` (CAPTURED / CAPTURE_PENDING) | Runs capture gate; `409 CAPTURE_BLOCKED` with `details` = failed checks |
| 12 | `POST /purchases/:id/cancel` | `{}` | `200 Purchase` (CANCELLED) | Allowed from PENDING_APPROVAL, APPROVED, ORDER_CREATED |
| 13 | `GET /audit` | query: `limit≤100`, `cursor`, `decision?`, `state?` | `200 {rows: LedgerRow[], nextCursor}` | Feeds AG Grid |
| 14 | `GET /purchases/:id/receipt` | — | `200 IntentReceipt` | |
| 15 | `GET /offers/:snapshotId/receipt` | — | `200 IntentReceipt` (no `purchase`) | Receipts for blocked / review offers |
| 16 | `GET /audit/verify` | — | `200 {status:"INTACT"\|"BROKEN", checked, firstBrokenSeq?}` | Recomputes the session chain |
| 17 | `POST /webhooks/paypal` | raw PayPal event | `200` always after storage | No session; signature verified; separate rate limit |
| 18 | `GET /analytics/dataset` | query: `days≤90` | `200 {rows: AnalyticsRow[]}` (≤ 5 000 rows) | Read-only; feeds AG Studio |
| 19 | `POST /studio-agent/turn` | `{threadId, messages[], tools[]}` (tools = Studio tool schemas, allow-listed by name) | `200 text/event-stream` of AG-UI events | Read-only; `404` when `STUDIO_AGENT_ENABLED=false` |
| 20 | `POST /paypal/client-token` | `{}` | `200 {clientToken, expiresIn}` | **Only if** Q6 shows that JS SDK v6 needs a client token; domain-bound, 15 min |

## 4. Schemas (shared Zod → TS)

```ts
MandateFields = strict({
  productQuery: string().trim().min(2).max(200),
  quantity: int().min(1).max(100),
  maxTotal: string().regex(/^\d{1,9}(\.\d{1,2})?$/),   // decimal string, converted to minor units server-side
  currency: literal("USD"),
  allowedMerchantDomains: array(domain()).min(1).max(20),
  recurringAllowed: boolean(),
  excludedTerms: array(string().trim().min(2).max(40)).max(20).default([]),
})

Mandate = MandateFields.partial() & {
  id, version, status: "DRAFT"|"NEEDS_CLARIFICATION"|"CONFIRMED"|"SUPERSEDED",
  requestText, clarificationQuestions: {field, question}[], parser, createdAt, confirmedAt?
}

EvaluatedOffer = {
  snapshotId, source: "channel3"|"fixture", title, brand?, merchantName?, merchantDomain?,
  offerUrl?, unitPrice?: Money, totalPrice?: Money, recurrenceSignal: "NONE"|"RECURRING",
  decision: "ALLOWED"|"BLOCKED"|"NEEDS_REVIEW", findings: Finding[], policyVersion, observedAt
}

Purchase = {
  id, mandateId, offerSnapshotId, state, quantity, unitPrice: Money, total: Money,
  mandateHash, approvedAt?, approvalExpiresAt?, paypalOrderId?, paypalCaptureId?,
  paypalStatus?, blockReasons?: Finding[], createdAt, updatedAt
}

LedgerRow = {
  id, time, kind: "OFFER"|"PURCHASE", product, merchant, source, decision, reasonCodes: string[],
  approved: boolean, paypalOrderId?, state?, total?: Money
}

IntentReceipt = {
  requestText, mandate, offer: EvaluatedOffer, purchase?, approval?, paypal?: {orderId, captureId?, status, debugId?},
  events: {seq, type, createdAt, hash}[], chain: "INTACT"|"BROKEN", sandboxNotice: string
}

AnalyticsRow = {
  time, day, productQuery, merchantDomain?, source, decision, primaryReason, reasonCodes: string[],
  offerTotalMinor?: string, approved: boolean, purchaseState?, capturedTotalMinor?: string,
  blockedAtCapture: boolean, approvalToCaptureSeconds?: number, policyVersion
}
```

`domain()` = lowercase, IDNA-normalized, `^[a-z0-9-]+(\.[a-z0-9-]+)+$`, ≤ 253 chars; leading `www.` stripped.

## 5. Policy reason codes

`packages/shared/src/reasonCodes.ts` is the single registry. Adding a code requires a test and a template message.

### 5.1 Offer evaluation

| Code | Severity | Fires when | Template |
|---|---|---|---|
| `ALL_RULES_PASSED` | INFO | No other finding | "All mandate rules passed." |
| `OVER_BUDGET` | BLOCKING | `unit × qty > maxTotal` | "Total {total} exceeds the {max} limit." |
| `VENDOR_NOT_APPROVED` | BLOCKING | domain not equal to, or subdomain of, an allowed domain | "{domain} is not an approved merchant." |
| `RECURRING_NOT_ALLOWED` | BLOCKING | `recurrenceSignal = RECURRING` and `!recurringAllowed` | "Offer includes recurring terms ('{evidence}') you did not authorize." |
| `CURRENCY_UNSUPPORTED` | BLOCKING | currency ≠ mandate currency | "Offer is priced in {currency}; mandate allows {mandateCurrency}." |
| `EXCLUDED_TERM_PRESENT` | BLOCKING | title/brand contains an excluded term (case-insensitive, word boundary) | "Offer matches excluded term '{term}'." |
| `PRICE_UNKNOWN` | REVIEW | price missing or unparseable | "Price could not be verified." |
| `MERCHANT_UNKNOWN` | REVIEW | merchant domain missing | "Merchant could not be verified." |

### 5.2 Capture gate (purchase → `BLOCKED_AT_CAPTURE`)

| Code | Check |
|---|---|
| `ORDER_MISMATCH` | Request `paypalOrderId` ≠ stored `paypal_order_id` |
| `PAYPAL_NOT_APPROVED` | PayPal order status ≠ `APPROVED` |
| `AMOUNT_MISMATCH` | PayPal amount/currency ≠ stored total |
| `CUSTOM_ID_MISMATCH` | PayPal `custom_id` ≠ purchase ID |
| `MANDATE_CHANGED` | Recomputed hash ≠ approved hash |
| `POLICY_NOW_BLOCKS` | Re-evaluation not `ALLOWED` (e.g. policy version or data changed) |
| `APPROVAL_EXPIRED` | `now ≥ approval_expires_at` |

## 6. PayPal outbound requests (server only)

**Create order** — `POST /v2/checkout/orders`, headers `PayPal-Request-Id: <stored uuid>`, `Prefer: return=representation` (VERIFY header support at Phase 1):

```json
{
  "intent": "CAPTURE",
  "purchase_units": [{
    "reference_id": "<purchaseId>",
    "custom_id": "<purchaseId>",
    "invoice_id": "<purchaseId>",
    "description": "IntentLock sandbox test order",
    "amount": { "currency_code": "USD", "value": "298.00",
      "breakdown": { "item_total": { "currency_code": "USD", "value": "298.00" } } },
    "items": [{ "name": "<title, truncated 127>", "quantity": "2", "category": "PHYSICAL_GOODS",
      "unit_amount": { "currency_code": "USD", "value": "149.00" } }]
  }]
}
```

**Get order** — `GET /v2/checkout/orders/{id}` (capture gate).
**Capture** — `POST /v2/checkout/orders/{id}/capture`, `PayPal-Request-Id: <stored uuid>`, empty body.

Map from capture response: `purchase_units[0].payments.captures[0].{id,status}`; record header `paypal-debug-id`.
