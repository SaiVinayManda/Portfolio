# Full AP Digital Employee — Sprint Execution & Triage

[Portfolio index](../README.md) · [Phase 1](01-ideation-and-strategy.md) · [Phase 2](02-roadmapping-and-scoping.md) · [Phase 3](03-requirements-prd-frd.md) · [Phase 4](04-sprint-execution-and-triage.md) · [Phase 5](05-testing-qa-and-uat.md) · [Phase 6](06-launch-gtm-and-telemetry.md)

**Document control:** FA-P4, revision 1.0, authored 2026-10-17. Status: proposed portfolio design; implementation, customer research, tests and stakeholder approvals have not been performed.

**Source boundary:** Umbrella upsell named on source p. 2, supported by the expansion path on p. 20. Proposed as an orchestration bundle, not a second invoice engine. [Primary document](../../MatchClerk-Project-Implementation.pdf). Proposed architecture and assumptions are explicitly recorded in [the evidence register](../source-and-assumptions.md).

## [PM & BSA] Refinement and working agreement

Refinement inputs are the approved PRD/FRD revision, a synthetic fixture pack, connector capability evidence and the ranked scope in Phase 2. PM explains the outcome and protects the scope; BSA walks field mappings, state transitions and failure paths; engineers estimate after discovering dependencies. The following are fully specified proposed tickets, not historical sprint results.

### FA-ENG-101 — Vendor verification

| Ticket field | Groomed decision |
| --- | --- |
| Business linkage | FA-BR-01/02 → FA-US-3 → FA-FR-3 → FA-SIT-03 |
| Goal | Enable the controller to verify onboarding evidence separately from intake so they can protect the vendor master. |
| Backend tasks | Implement POST /api/v1/ap-work-items/{id}/vendor-verifications; validate tenant, role, source identity and expected_version; evaluate: All required component tasks complete; vendor evidence verified; held exceptions cleared; controller independent of preparer; component versions still current. |
| Frontend tasks | Render AP work item evidence, calculated decision reasons and proposed result; show Any component hold, revoked vendor verification or stale approval → BLOCKED; no pack release while a linked task is unresolved.; preserve edited values on validation errors and support keyboard-only review. |
| Data tasks | Migrate ap_work_item and its dependent evidence tables from the ERD; enforce tenant composite foreign keys; write work_record, audit_event and outbox_event atomically; add tenant/status/time index. |
| Estimation | 8 story points = 48 planning hours: backend 18h, frontend 12h, data/migrations 8h, integration/QA 10h. Planning ratio is six productive hours per point. |
| Dependencies | Requires stable MatchClerk, AP Exception Agent and Vendor Onboarding Agent interfaces; receiving integration is optional by tenant. Authentication, tenant RLS and source fixture must be available before sprint commitment. |
| Acceptance | Given a clerk submitted vendor registration evidence; When a controller reviews the evidence; Then verification status becomes VERIFIED and no bank master update is sent. Assert: vendor_verification.status='VERIFIED'; external_write count=0. |
| Negative tests | Wrong tenant →404; disallowed role →403; malformed decimal →422; stale revision →409; replay preserves one business action. |
| Definition of ready | Field mapping reviewed, fixture expected values labeled, control rule agreed, API schema reviewed and integration dependency proven. |
| Definition of done | Code review, required SIT and role tests pass; migration rehearsal succeeds; audit/metrics present; accessible UI reviewed; PM accepts demonstrated behavior. |

### FA-ENG-102 — Separate approval

| Ticket field | Groomed decision |
| --- | --- |
| Business linkage | FA-BR-01/02/03 → FA-US-6 → FA-FR-6 → FA-SIT-06 |
| Goal | Enable the controller to approve a prepared AP cycle so they can retain independent accountability. |
| Backend tasks | Implement POST /api/v1/ap-work-items/{id}/approve; bind current payload/source revision to authorization; create durable operation; reconcile ambiguous outcomes before retry; enforce This bundle delegates to MatchClerk and AP Exception Agent APIs. It never independently writes a duplicate bill or executes a bank payment. |
| Frontend tasks | Show exact reviewed values, target system or consumer, authorization role and last evidence update; disable a stale action; show accepted/processing/succeeded/failed states separately with an operation reference. |
| Data tasks | Store approval payload hash and approved_version; add unique logical-action key; persist any authorized external_write plus outbox atomically; store returned artifact hash or provider reference only on confirmed success. |
| Estimation | 8 story points = 48 planning hours: backend/adapter 20h, frontend 10h, data/integrity 8h, failure-injection and QA 10h. |
| Dependencies | FA-ENG-101 evidence gate, identity/roles, sandbox/consumer fixture and operation reconciliation contract. No live provider credentials in fixtures. |
| Acceptance | Given the preparer and controller are different authenticated users; When the controller approves current version 4; Then one approval is stored and a pack handoff is queued without bill creation. Assert: approval count=1; outbox_event.event_type='ap.cycle_approved'. |
| Failure-path acceptance | Replayed commands return the existing operation; stale evidence invalidates authorization; 429 honors Retry-After; ambiguous timeout enters reconciliation; terminal failure has named recovery ownership. |
| Definition of ready | Target capability proof, deterministic fixture, payload mapping and review authority available; source data owner agrees to the semantics. |
| Definition of done | Contract and E2E sandbox/consumer checks pass; duplicate/replay and permissions verified; support can find and safely recover the operation; PM and BSA inspect evidence. |

## [PM & BSA] Two-week sprint plan

Reference Sprint 2 runs 2027-08-16–2027-08-27, ten working days. **Sprint goal:** demonstrate vendor verification through separate approval for one representative tenant, including the blocked path and recovery evidence. This slice assumes foundational work from Sprint 1.

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
| 3–5 | Vendor verification service, UI and rule demonstration | Full-stack + AI/integration engineer | ENG-101 remaining 5 points |
| 6–8 | Separate approval command, authorization and operation ledger | Integration + full-stack engineer | ENG-102 first 6 points |
| 9 | Failure/replay checks; complete UI and reconciliation evidence | BSA + engineers | ENG-102 remaining 2 + observability 2 points |
| 10 | Demonstration, targeted UAT rehearsal and retrospective | PM + BSA + customer process owner | Release evidence 2 points |

If a dependency fails, replace the blocked ticket only with a pre-refined item of equal or lower size that advances the sprint goal. PM records the scope decision; BSA records the failed technical assumption. No planned item is labeled delivered until acceptance evidence exists.

## [PM] Business and design triage log

These are simulated decisions for portfolio assessment. They are not reports of actual customer escalations.

| ID / severity | Roadblock | Resolution decision | Owner / success evidence |
| --- | --- | --- | --- |
| FA-PM-1 / High | Customer equates digital employee with replacing all AP staff | Sell supervised operating capacity with explicit human control responsibilities. | PM + controller; validate in design-partner walkthrough before GA |
| FA-PM-2 / Medium | Different teams disagree on cutoff | Adopt tenant business calendar and controller-owned weekly deadline. | PM + UX; validate in design-partner walkthrough before GA |
| FA-PM-3 / Medium | Bundle would double-charge component subscriptions | Bundle includes MatchClerk and exception processing allowances; one commercial meter. | PM + UX; validate in design-partner walkthrough before GA |

## [BSA] Data and API triage log

| ID / severity | Edge case | Exact technical resolution | Verification / owner |
| --- | --- | --- | --- |
| FA-BSA-1 / High | A source component is unavailable during cycle approval | Fail closed with 424 COMPONENT_UNAVAILABLE; cached completion does not authorize release. | Failure-injection fixture; verify source and target counts, state, audit and recovery behavior / integration lead |
| FA-BSA-2 / High | Events arrive out of order | Persist source_version; discard lower versions; re-fetch source on sequence gaps. | Failure-injection fixture; verify source and target counts, state, audit and recovery behavior / integration lead |
| FA-BSA-3 / High | Workflow retries attempt a second bill | Orchestrator can invoke component review and pack APIs only; no Bill write scope is assigned. | Failure-injection fixture; verify source and target counts, state, audit and recovery behavior / integration lead |

### Cross-cutting connector recovery

For a 429, suspend only the affected tenant/connection queue for Retry-After seconds; otherwise use 30 seconds if that header is absent. Read/provider 5xx retries use full-jitter delays capped at 5, 30, 120, 300 and 900 seconds; after five failed executions route to the dead-letter recovery queue. Unknown write outcomes bypass blind retries and enter RECONCILING. OAuth refresh is serialized per connection; a revoked grant moves it to RECONNECT_REQUIRED and pauses dependent work. These rules also apply when the output is a consumer webhook or approved export rather than an ERP mutation.

**Escalation and ownership:** PM resolves intended business behavior and customer impact; BSA owns field/state diagnosis; engineering owns the code repair and recovery; the controller decides whether financial evidence is sufficient. A severity-1 integrity or isolation issue pauses the affected action path immediately and follows Phase 6 incident rules.
