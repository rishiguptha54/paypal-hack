# IntentLock — Research Log and Decisions

> Facts checked against official sources on **26 Oct 2026** unless noted. Each fact maps to a design decision. Items marked **VERIFY** must be re-checked at implementation time because the source was unreachable or is likely to change.

## 1. Hackathon constraints (source: [official rules](https://paypalaihackathon.devpost.com/rules))

| Fact | Consequence for IntentLock |
|---|---|
| Submission period ends **Thu 12 Nov 2026, 12:00 PT**. | Internal freeze 10 Nov; submit 11 Nov ([plan.md](plan.md)). |
| Must integrate PayPal developer platform (Sandbox) **and** an AI tool/model; PayPal must be **central**. | PayPal Orders v2 create → buyer approve → verified capture is the spine of the product, built first. |
| Stage One is pass/fail on theme fit and use of required APIs. | P0 = PayPal + AI working end-to-end before anything else. |
| Five equally weighted criteria: Technological Implementation, Design, Potential Impact, Innovation, Presentation. | Each has an explicit artifact in [plan.md §6](plan.md#6-judging-criteria-coverage). |
| Functional demo required; a static prototype does **not** qualify. Hosted URL **or** complete run instructions. | Ship both: Render URL + `docker compose` path. |
| Public repo with OSI license visible in the About section. | MIT `LICENSE` in Phase 0. |
| Video < 3 min, public on YouTube, no unlicensed music/trademarks. | Script in [plan.md §7](plan.md#7-demo-video-script-under-3-minutes). |
| Must provide test credentials / sandbox logins needed to test. | README section "Judge access" with sandbox buyer account (sandbox only, never real credentials). |
| Max **one** Sponsor Prize + one Grand or Honorable Mention. | Single sponsor focus: **AG Grid via AG Studio**; fallback pitch **Channel3**. Render and Postman are used for product value, not prize count. |
| Sponsor tools listed: AG Grid, APIMatic, Astropods, Bryntum, Channel3, Elastic, KERNEL, Postman, Render, Zapier. | Use: Channel3 (data), AG Grid + AG Studio (ledger + governance dashboard), Render (hosting), Postman (judge reproducibility). Not used: others. |
| Honorable mentions include **Best Use of PayPal + AI** and **Best Use of Agentic Commerce**. | Positioning targets these two categories. |
| The overview page also requires a **"Tools" section**: name each AI model/service and explain how it is used. | Pre-written table in [submission.md §4.3](submission.md#43-tools-field-required). |
| Judging "may utilize … automated AI-driven analysis". Judges may score **only from the description, images and video**. | The README and Devpost text map each criterion to evidence explicitly; the video and gallery must stand alone. |
| Judge panel: PayPal developer advocates/engineers/PM plus one representative per sponsor (AG Grid, APIMatic, Bryntum, Channel3, Elastic, Postman, Render). | PayPal reviewers will look at current SDK use (JS SDK v6, Orders v2 best practice); the AG Grid reviewer will look at AG Studio depth. |
| Testing access must stay free and unrestricted **until the judging period ends (15 Dec)**. | Paid hosting + DB for December ([submission.md §7](submission.md#7-judging-period-runbook-115-dec)). |
| Resources page links **PayPal JS SDK v6**, PayPal AI docs, Agent Toolkit and MCP server. | Use JS SDK v6 (ADR-012). Agent Toolkit/MCP wrapping is a stretch item (S1), not MVP. |

## 2. PayPal

| Fact | Source | Decision |
|---|---|---|
| Orders v2 endpoints: `POST /v2/checkout/orders`, `GET /v2/checkout/orders/{id}`, `PATCH` (CREATED/APPROVED only), `POST …/authorize`, `POST …/capture`. Sandbox base `https://api-m.sandbox.paypal.com`; OAuth2 client credentials. | [Orders v2](https://developer.paypal.com/api/orders/v2/) | Use intent `CAPTURE`. Use `GET order` as pre-capture truth source. Never `PATCH` after approval (would let amounts change post-approval). |
| Standard Checkout: JS SDK Buttons call your server in `createOrder`, then `onApprove` calls your server to capture. Server uses the **PayPal Server SDK** (`ordersCreate`, `ordersCapture`). | [Standard checkout integrate](https://developer.paypal.com/studio/checkout/standard/integrate) | Frontend: **JS SDK v6** one-time payment session (see the JS SDK v6 row below); v5 Buttons only as a fallback if v6 blocks us. Backend: `@paypal/paypal-server-sdk` (VERIFY exact package/version at Phase 1). Raw REST fallback if the SDK lacks a header we need. |
| `PayPal-Request-Id` makes POSTs idempotent; same ID returns latest status instead of duplicating; omit it and PayPal duplicates. Must be unique per request **and** per call type; UUID recommended (≤ 38 chars). Simultaneous identical IDs: second may fail. | [Idempotency](https://developer.paypal.com/api/rest/reference/idempotency/) | Store one UUID per (purchase, operation) in `payment_operations`. Serialize capture with DB compare-and-set so we never send concurrent identical requests. |
| Funding failure returns `INSTRUMENT_DECLINED`; payer should be allowed to retry. | Standard checkout guide | Keep purchase in `ORDER_CREATED`, audit `PAYPAL_INSTRUMENT_DECLINED`, and let the buyer re-approve with the same order (no new order). |
| Webhooks: `CHECKOUT.ORDER.APPROVED`, `CHECKOUT.PAYMENT-APPROVAL.REVERSED`, `PAYMENT.CAPTURE.COMPLETED`, `PAYMENT.CAPTURE.PENDING`, `PAYMENT.CAPTURE.DECLINED`/`DENIED`, `PAYMENT.CAPTURE.REFUNDED`. | [Event names](https://developer.paypal.com/api/rest/webhooks/event-names/) | Subscribe to these six. Synchronous capture response stays primary; webhook reconciles `PENDING` and reversals. |
| Listener must be HTTPS:443, return 2xx; PayPal retries up to 25 times over 3 days. Self-verification: `transmissionId|timeStamp|webhookId|crc32(rawBody)` signed SHA256withRSA with cert from `paypal-cert-url`; raw body required. Postback alternative: `POST /v1/notifications/verify-webhook-signature` (not usable for simulator mock events). | [Integrate webhooks](https://developer.paypal.com/api/rest/webhooks/rest/) | Use `express.raw()` on the webhook route only. Self-verify; **pin cert URL host to `*.paypal.com`** (prevents attacker-supplied cert). Dedupe by event ID. Webhooks are P1. |
| **JS SDK v6**: script `https://www.sandbox.paypal.com/web-sdk/v6/core`; `window.paypal.createInstance({clientToken \| clientId, components:["paypal-payments"], pageType:"checkout"})`; `findEligibleMethods()`; `createPayPalOneTimePaymentSession({onApprove,onCancel,onError})`; `session.start({presentationMode:"auto"}, createOrder())` where `createOrder` resolves to `{ orderId }`. Docs' best practices: "Never pass up item total from browser", "Validate order details on your server before capture". Client token is server-generated, domain-bound, expires in 15 min. | [JS SDK v6 setup](https://developer.paypal.com/sdk/js/) | Use v6 (ADR-012). Docs say most integrations should use client ID, but the init example shows a client token: **VERIFY** in Phase 1 (Q6). PayPal's own best practices match our capture gate; cite them in the README. |
| PayPal AI resources: Agent Toolkit, MCP server, agent tools reference. | [Resources](https://paypalaihackathon.devpost.com/resources) | Not on the MVP critical path (the LLM must not hold payment tools). Stretch S1: an IntentLock MCP tool `purchase_within_mandate` so outside agents have to go through the gate. |
| Agent Ready (ACP via Braintree, UCP via Braintree/PayPal Orders v2) targets merchants on ChatGPT/Gemini surfaces; Agentic Commerce Services require onboarding. | [Agent Ready](https://developer.paypal.com/agent-ready/overview), [ACS](https://developer.paypal.com/agentic-commerce-services/about/) | IntentLock is positioned as a **buyer-side** policy gate complementary to these, not a replacement. No dependency on gated access. |

## 3. Channel3

| Fact | Source | Decision |
|---|---|---|
| Public developer page documents **Search** (natural language/image, filters, ranked results with offers) and **Lookup** (URL → structured product with offers). **Checkout is "Coming soon" / early access.** | [Channel3 developers](https://www.trychannel3.com/developers/) | Use Search for discovery only. Payment is the app's own PayPal Sandbox order, labelled as such. |
| Docs moved from `docs.trychannel3.com` to `channel3.ai/docs` (redirect observed). Detailed docs were not reachable from this network. | — | **VERIFY** endpoint, auth header and response fields in Phase 3 spike. Blueprint records `POST https://api.trychannel3.com/v1/search` with `x-api-key`. Adapter isolates any change. |
| Hackathon sponsor page: keys free for all participants; promo code `PAYPAL-HACKATHON-2026` = 20 000 credits (≈ 20 000 searches); top-ups via the `#channel3` Discord channel. Projects that only mention Channel3 without **calling** it don't qualify for the sponsor prize. Docs link given: `docs.trychannel3.com`. | [Channel3 sponsor page](https://paypalaihackathon.devpost.com/details/channel3) | Apply the promo code on 27 Oct. A 10-min search cache keeps usage low. Channel3 is the fallback sponsor-prize target. |
| An MCP server is offered. | Developer page | Not used: the LLM must not drive tools that influence payment. Plain REST from the server keeps the trust boundary simple. |
| Recurrence/subscription is **not** a documented first-class field. | Developer page | Derive `recurrenceSignal` deterministically from title/description keywords (ADR-006). Record the method as evidence. |

## 4. AG Grid

| Fact | Source | Decision |
|---|---|---|
| React grid installs as `ag-grid-react` + `ag-grid-community`; versions must match. Enterprise features need a license. Server-Side Row Model is Enterprise; Client-Side and Infinite Row Models are Community. | [Installation](https://www.ag-grid.com/react-data-grid/installation/) | Ledger uses Community, Client-Side Row Model. Custom cell renderers for decision chips, reason codes and chain status — **text only**, never HTML from product data. |
| **The "Best Use of AG Grid" prize is judged on AG Studio**: "what your dashboard looks like and how you build it … custom widgets, theming, and layout tools, not just AG Grid or AG Charts on their own", plus the **Studio Agent Framework** ("a custom agent talking to Studio's built-in agents, or a swapped-in harness"). Free **45-day trial license**; official PayPal + AG Studio boilerplate: `github.com/paypaldev/hackathon-paypal-ag-grid-boilerplate`. | [AG Grid sponsor page](https://paypalaihackathon.devpost.com/details/aggrid) | **A plain Community grid will not compete for this prize.** Add the Governance Studio (ADR-013). Start from the boilerplate. Activate the trial **on or after 1 Nov** so it lasts past 15 Dec. |
| Studio Agent Framework: agent = definition (`id`, `instructions()`, `tools()`, `maxTurns`) + runner. `directLlmRunner` (you supply a model adapter), **`clientToolRunner`** ("run: (input, ctx) => myServer.stream(input)"; Studio executes the unresolved tool calls in the browser), or a custom runner. Events follow the AG-UI protocol. Built-in agents can be delegated to via `delegate_to`. | [AG Studio agents](https://www.ag-grid.com/studio/react/ai-agents/) | Use `clientToolRunner` so the LLM key stays on the server (ADR-014). Our custom "Governance Analyst" agent delegates widget building to Studio's built-in agents. |

## 5. Render

| Fact | Source | Decision |
|---|---|---|
| Web Services must bind `0.0.0.0:$PORT`; env vars/secrets configured per service; Static Sites separate. Service filesystem is not durable storage. | [Web services](https://render.com/docs/web-services), [Env vars](https://render.com/docs/configure-environment-variables) | **One** Web Service serves API **and** built SPA (same origin ⇒ no CORS, simpler cookies/CSP) + Render Postgres. Blueprint `render.yaml` in repo. |
| Free tier: web service **spins down after 15 min idle, ~1 min to wake**; 750 free instance-hours/month; **free Postgres expires 30 days after creation** (14-day grace, then deleted); no backups; only one free Postgres per workspace. | [Render free tier](https://render.com/docs/free) | Use free tier during the build. Move to paid instances **before 30 Nov** using the hackathon's Render credit offer ($50 claim link on the [Render sponsor page](https://paypalaihackathon.devpost.com/details/render)). |

## 6. AI provider

| Consideration | Decision |
|---|---|
| Need strict JSON matching a schema; provider may change. | `IntentParser` interface with one production adapter (OpenAI-compatible structured outputs, model via `AI_MODEL` env) and one deterministic `RuleBasedParser` used in tests and as degraded mode. Output **always** re-validated by Zod. |
| Prompt injection via product data. | LLM sees **only the user's own request**. Product data never enters a prompt. Explanations are rendered from reason-code templates, not generated. |
| Cost/latency. | One LLM call per request (not per offer). |

## 7. Alternatives considered

| Option | Rejected because |
|---|---|
| Separate static site + API service | Adds CORS, cookie `SameSite=None`, two deploys. Same-origin single service is simpler and safer. |
| SQLite on Render disk | Not durable across redeploys without a paid disk; Postgres gives transactions + row-level CAS. |
| LLM decides allow/block | Non-deterministic, not auditable, injection-prone. |
| `intent=AUTHORIZE` then capture | Extra state and 29-day authorization handling with no MVP benefit; `CAPTURE` + pre-capture gate gives the same protection. |
| Webhook as primary success signal | Async and retried for days; synchronous capture response + `GET order` is authoritative for the demo. |
| Event sourcing for all state | Overkill; we use normal tables **plus** an append-only audit log. |

## 8. Open questions (owner → due)

| # | Question | Owner | Due |
|---|---|---|---|
| Q1 | Channel3 key issued? Exact search request/response schema and rate limit? | Backend | 28 Oct |
| Q2 | Does `@paypal/paypal-server-sdk` expose `PayPal-Request-Id` on `ordersCapture`? (Else raw `fetch`.) | Payments | 28 Oct |
| Q3 | Which AI provider/key does the team have budget for? | Lead | 27 Oct |
| Q4 | Render credits claimed and applied? Paid DB scheduled before 30 Nov? | DevOps | 30 Oct |
| Q5 | AG Studio trial: OK for a public open-source repo? Where must the key live (client-side by nature)? Ask on the AG Grid Discord channel. | Frontend | 28 Oct |
| Q6 | JS SDK v6: client ID or server client token for Orders-based one-time payments? | Payments | 28 Oct |
| Q7 | Employer policy / IP check done for every member ([submission.md §2](submission.md#employer--ip-check-do-this-first--27-oct)). | Each member | 27 Oct |
