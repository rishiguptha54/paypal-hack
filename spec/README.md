# IntentLock — Specification Pack

> **AI proposes. Code enforces. You approve.**
> A buyer-side policy gate that binds a user-confirmed purchase mandate to one exact PayPal Sandbox order and refuses capture if anything protected drifts.

| Field | Value |
|---|---|
| Status | Baseline v1.0 — approved for build |
| Spec date | 26 Oct 2026 |
| Hard deadline | **12 Nov 2026, 12:00 PT** (= 13 Nov 2026, 01:30 IST). Internal target: submit **11 Nov 2026**. |
| Source docs | [../paypal-ai-hackathon-blueprint.md](../paypal-ai-hackathon-blueprint.md), [../intentlock-prototype.html](../intentlock-prototype.html), [../IntentLock-Intent-Gate-Pitch.html](../IntentLock-Intent-Gate-Pitch.html) |

## Reading order

| # | File | Answers | Primary reader |
|---|---|---|---|
| 1 | [spec.md](spec.md) | *What* are we building and how do we know it works? (user stories, FR/NFR IDs, acceptance) | Everyone |
| 2 | [research.md](research.md) | What did we verify in official docs, and which decisions follow from it? | Tech lead, reviewers |
| 3 | [architecture.md](architecture.md) | *How* is it built? (containers, components, flows, state machine, ADRs) | Engineers |
| 4 | [data-model.md](data-model.md) | Tables, invariants, money rules, audit hash chain | Backend |
| 5 | [contracts/api.md](contracts/api.md) | Every HTTP route, request/response shape, error codes, policy reason codes | Frontend + backend |
| 6 | [security.md](security.md) | Threat model (STRIDE), controls, secrets, headers | Everyone touching payments |
| 7 | [test-plan.md](test-plan.md) | Test pyramid, mandatory scenarios, evidence for judges | QA / everyone |
| 8 | [plan.md](plan.md) | Calendar, phases, quality gates, owners, risk burndown | Team lead |
| 9 | [tasks.md](tasks.md) | Ordered, dependency-tagged task list for Copilot / devs | Engineers |
| 10 | [submission.md](submission.md) | Rules compliance matrix, prize strategy, Devpost content, judge access, video checklist, judging-period runbook | Team lead / submitter |

## Five rules that override everything else

1. **The LLM never authorizes money.** It produces a *draft* mandate only. Deterministic server code decides; a human approves; PayPal captures.
2. **The server owns every number.** Totals are recomputed from stored snapshots in integer minor units. Client-sent amounts are rejected, not ignored silently.
3. **Unknown fails closed.** Missing price, merchant or currency → `NEEDS_REVIEW` (not purchasable) — never `ALLOWED`.
4. **One approval binds one order.** Approval stores a `mandateHash`; any change to protected fields or an expired approval invalidates capture.
5. **Every decision is evidence.** Blocks, approvals, PayPal responses and rejections are written to an append-only, hash-chained audit log.

## Scope fence (do not build)

Real-money payments · retailer checkout via Channel3 (early-access only) · autonomous capture · multi-agent orchestration for payments · vector DB · Bryntum · Render Workflows · AG Grid Enterprise grid features (AG Studio trial is in scope for the governance dashboard only) · user accounts beyond an anonymous session.
