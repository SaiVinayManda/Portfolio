# Procurement Digital Employee — Sprint Execution & Triage

[Portfolio index](../README.md) · [Phase 1](01-ideation-and-strategy.md) · [Phase 2](02-roadmapping-and-scoping.md) · [Phase 3](03-requirements-prd-frd.md) · [Phase 4](04-sprint-execution-and-triage.md) · [Phase 5](05-testing-qa-and-uat.md) · [Phase 6](06-launch-gtm-and-telemetry.md)

**Document control:** PD-P4, revision 1.0, authored 2026-10-17. Status: proposed portfolio design; implementation, customer research, tests and stakeholder approvals have not been performed.

**Source boundary:** Named expansion, source p. 20: PO drafting from approved requests, vendor scorecards and catalog price watch attachment. [Primary document](../../MatchClerk-Project-Implementation.pdf). Proposed architecture and assumptions are explicitly recorded in [the evidence register](../source-and-assumptions.md).

## [PM & BSA] Refinement and working agreement

Refinement inputs are the approved PRD/FRD revision, a synthetic fixture pack, connector capability evidence and the ranked scope in Phase 2. PM explains the outcome and protects the scope; BSA walks field mappings, state transitions and failure paths; engineers estimate after discovering dependencies. The following are fully specified proposed tickets, not historical sprint results.

### PD-ENG-101 — Approval policy

| Ticket field | Groomed decision |
| --- | --- |
| Business linkage | PD-BR-01/02 → PD-US-3 → PD-FR-3 → PD-SIT-03 |
| Goal | Enable the procurement manager to route high-value requests through two reviewers so they can enforce commitment authority. |
| Backend tasks | Implement POST /api/v1/purchase-requests/{id}/submit; validate tenant, role, source identity and expected_version; evaluate: Requester cannot self-approve; budget reservation succeeds; supplier active; catalog valid; required manager/controller approval complete; PO connector supports line type. |
| Frontend tasks | Render purchase request evidence, calculated decision reasons and proposed result; show Budget overrun, inactive supplier, stale catalog or missing approval → NEEDS_REVIEW; no PO dispatch before explicit approval.; preserve edited values on validation errors and support keyboard-only review. |
| Data tasks | Migrate purchase_request and its dependent evidence tables from the ERD; enforce tenant composite foreign keys; write work_record, audit_event and outbox_event atomically; add tenant/status/time index. |
| Estimation | 8 story points = 48 planning hours: backend 18h, frontend 12h, data/migrations 8h, integration/QA 10h. Planning ratio is six productive hours per point. |
| Dependencies | Requires tenant budget owners, vendor master and a proven purchase-order connector; MarginGuard is an optional attachment. Authentication, tenant RLS and source fixture must be available before sprint commitment. |
| Acceptance | Given a request amount is 5200.00; When the requester submits it; Then manager and controller tasks are created and the requester cannot satisfy either task. Assert: request_approval required count=2; purchase_request.status='NEEDS_REVIEW'. |
| Negative tests | Wrong tenant →404; disallowed role →403; malformed decimal →422; stale revision →409; replay preserves one business action. |
| Definition of ready | Field mapping reviewed, fixture expected values labeled, control rule agreed, API schema reviewed and integration dependency proven. |
| Definition of done | Code review, required SIT and role tests pass; migration rehearsal succeeds; audit/metrics present; accessible UI reviewed; PM accepts demonstrated behavior. |

### PD-ENG-102 — Supplier dispatch

| Ticket field | Groomed decision |
| --- | --- |
| Business linkage | PD-BR-01/02/03 → PD-US-6 → PD-FR-6 → PD-SIT-06 |
| Goal | Enable the procurement manager to approve a supplier dispatch separately so they can control external communications. |
| Backend tasks | Implement POST /api/v1/purchase-orders/{id}/dispatch-approval; bind current payload/source revision to authorization; create durable operation; reconcile ambiguous outcomes before retry; enforce Agents draft POs and supplier messages. Humans authorize financial commitment and outbound dispatch; no inferred approval from an email sentiment. |
| Frontend tasks | Show exact reviewed values, target system or consumer, authorization role and last evidence update; disable a stale action; show accepted/processing/succeeded/failed states separately with an operation reference. |
| Data tasks | Store approval payload hash and approved_version; add unique logical-action key; persist any authorized external_write plus outbox atomically; store returned artifact hash or provider reference only on confirmed success. |
| Estimation | 8 story points = 48 planning hours: backend/adapter 20h, frontend 10h, data/integrity 8h, failure-injection and QA 10h. |
| Dependencies | PD-ENG-101 evidence gate, identity/roles, sandbox/consumer fixture and operation reconciliation contract. No live provider credentials in fixtures. |
| Acceptance | Given a published PO has a reviewed recipient and PDF; When I approve dispatch; Then one dispatch event is queued and delivery state is recorded independently of PO publication. Assert: supplier_dispatch.approved_by IS NOT NULL; outbox_event dispatch count=1. |
| Failure-path acceptance | Replayed commands return the existing operation; stale evidence invalidates authorization; 429 honors Retry-After; ambiguous timeout enters reconciliation; terminal failure has named recovery ownership. |
| Definition of ready | Target capability proof, deterministic fixture, payload mapping and review authority available; source data owner agrees to the semantics. |
| Definition of done | Contract and E2E sandbox/consumer checks pass; duplicate/replay and permissions verified; support can find and safely recover the operation; PM and BSA inspect evidence. |

## [PM & BSA] Two-week sprint plan

Reference Sprint 2 runs 2027-10-18–2027-10-29, ten working days. **Sprint goal:** demonstrate approval policy through supplier dispatch for one representative tenant, including the blocked path and recovery evidence. This slice assumes foundational work from Sprint 1.

| Capacity / planning variable | Value | Interpretation |
| --- | --- | --- |
| Engineering allocation | 2 full-stack engineers + 0.5 AI/integration engineer | Matches the lean source team; half-time specialist is not a third full-time engineer |
| Gross capacity | 25 engineer-days × 6 productive hours = 150h | Ten working days; excludes design and PM point estimates |
| Support / interrupts reserve | 20% = 30h | Connector surprises and production interruptions are visible capacity |
| Net planned delivery | 120h = 20 story points | Planning conversion 6h/point; hours do not replace team-relative estimation |
| Velocity | 20 points assumed; no historical velocity exists | Replace with completed, accepted points after two sprints |
| Committed scope | ENG-101 8 + ENG-102 8 + observability 2 + release evidence 2 = 20 points | No automatic addition of Should/Could features |
| Design / QA / PM support | 0.25 designer; BSA owns SIT; PM 0.5 delivery/GTM allocation | Acceptance and evidence review scheduled, not implied engineering capacity |

| Working days | Activity and deliverable | Owner | Capacity commitment |
| --- | --- | --- | --- |
| 1–2 | Confirm mapping and control fixtures; migrate schema; review API and UX states | BSA + engineering lead | ENG-101 first 3 points |
| 3–5 | Approval policy service, UI and rule demonstration | Full-stack + AI/integration engineer | ENG-101 remaining 5 points |
| 6–8 | Supplier dispatch command, authorization and operation ledger | Integration + full-stack engineer | ENG-102 first 6 points |
| 9 | Failure/replay checks; complete UI and reconciliation evidence | BSA + engineers | ENG-102 remaining 2 + observability 2 points |
| 10 | Demonstration, targeted UAT rehearsal and retrospective | PM + BSA + customer process owner | Release evidence 2 points |

If a dependency fails, replace the blocked ticket only with a pre-refined item of equal or lower size that advances the sprint goal. PM records the scope decision; BSA records the failed technical assumption. No planned item is labeled delivered until acceptance evidence exists.

## [PM] Business and design triage log

These are simulated decisions for portfolio assessment. They are not reports of actual customer escalations.

| ID / severity | Roadblock | Resolution decision | Owner / success evidence |
| --- | --- | --- | --- |
| PD-PM-1 / High | Requesters want self-approval for urgent materials | Use an emergency alternate approver with recorded reason; preserve independent approval. | PM + controller; validate in design-partner walkthrough before GA |
| PD-PM-2 / Medium | Purchasing and finance disagree on commitment timing | Reserve budget at final approval and release on cancellation. | PM + UX; validate in design-partner walkthrough before GA |
| PD-PM-3 / Medium | Suppliers reject automatically sent POs | Require buyer-approved PDF and recipient; dispatch remains a separate action. | PM + UX; validate in design-partner walkthrough before GA |

## [BSA] Data and API triage log

| ID / severity | Edge case | Exact technical resolution | Verification / owner |
| --- | --- | --- | --- |
| PD-BSA-1 / High | Two approvals overdraw a budget | Lock budget row; conditional decrement where available ≥amount; write reservation atomically. | Failure-injection fixture; verify source and target counts, state, audit and recovery behavior / integration lead |
| PD-BSA-2 / High | Provider rejects an unsupported expense-only PO line | Run connector capability check; require mapped inventory/item reference or internal PDF-only controlled workflow. | Failure-injection fixture; verify source and target counts, state, audit and recovery behavior / integration lead |
| PD-BSA-3 / High | A vendor score uses incomplete deliveries | Use completed POs only, expose sample count and exclude cancelled lines from the denominator. | Failure-injection fixture; verify source and target counts, state, audit and recovery behavior / integration lead |

### Cross-cutting connector recovery

For a 429, suspend only the affected tenant/connection queue for Retry-After seconds; otherwise use 30 seconds if that header is absent. Read/provider 5xx retries use full-jitter delays capped at 5, 30, 120, 300 and 900 seconds; after five failed executions route to the dead-letter recovery queue. Unknown write outcomes bypass blind retries and enter RECONCILING. OAuth refresh is serialized per connection; a revoked grant moves it to RECONNECT_REQUIRED and pauses dependent work. These rules also apply when the output is a consumer webhook or approved export rather than an ERP mutation.

**Escalation and ownership:** PM resolves intended business behavior and customer impact; BSA owns field/state diagnosis; engineering owns the code repair and recovery; the controller decides whether financial evidence is sufficient. A severity-1 integrity or isolation issue pauses the affected action path immediately and follows Phase 6 incident rules.
