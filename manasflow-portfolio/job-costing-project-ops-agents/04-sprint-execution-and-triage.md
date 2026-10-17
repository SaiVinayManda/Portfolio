# Job-Costing / Project Ops Agents — Sprint Execution & Triage

[Portfolio index](../README.md) · [Phase 1](01-ideation-and-strategy.md) · [Phase 2](02-roadmapping-and-scoping.md) · [Phase 3](03-requirements-prd-frd.md) · [Phase 4](04-sprint-execution-and-triage.md) · [Phase 5](05-testing-qa-and-uat.md) · [Phase 6](06-launch-gtm-and-telemetry.md)

**Document control:** JC-P4, revision 1.0, authored 2026-10-17. Status: proposed portfolio design; implementation, customer research, tests and stakeholder approvals have not been performed.

**Source boundary:** Expansion capability, source p. 20: contractor job-costing sync and job-costing/project-ops agent offer. Detailed allocation controls are proposed. [Primary document](../../MatchClerk-Project-Implementation.pdf). Proposed architecture and assumptions are explicitly recorded in [the evidence register](../source-and-assumptions.md).

## [PM & BSA] Refinement and working agreement

Refinement inputs are the approved PRD/FRD revision, a synthetic fixture pack, connector capability evidence and the ranked scope in Phase 2. PM explains the outcome and protects the scope; BSA walks field mappings, state transitions and failure paths; engineers estimate after discovering dependencies. The following are fully specified proposed tickets, not historical sprint results.

### JC-ENG-101 — Valid allocation

| Ticket field | Groomed decision |
| --- | --- |
| Business linkage | JC-BR-01/02 → JC-US-3 → JC-FR-3 → JC-SIT-03 |
| Goal | Enable the project accountant to allocate a line to jobs and cost codes so they can keep job costs accurate. |
| Backend tasks | Implement PATCH /api/v1/cost-allocations/{id}/lines; validate tenant, role, source identity and expected_version; evaluate: Source bill posted/approved per connector contract; valid open job and cost code; allocations sum exactly to source amount; cost basis explicit; controller approves changes. |
| Frontend tasks | Render cost allocation evidence, calculated decision reasons and proposed result; show Closed job, unmapped code, allocation imbalance or stale source revision → NEEDS_REVIEW; no sync until resolved.; preserve edited values on validation errors and support keyboard-only review. |
| Data tasks | Migrate cost_allocation and its dependent evidence tables from the ERD; enforce tenant composite foreign keys; write work_record, audit_event and outbox_event atomically; add tenant/status/time index. |
| Estimation | 8 story points = 48 planning hours: backend 18h, frontend 12h, data/migrations 8h, integration/QA 10h. Planning ratio is six productive hours per point. |
| Dependencies | Requires stable job/cost-code master, approved bill events and a contracted project-cost adapter or reviewed CSV. Authentication, tenant RLS and source fixture must be available before sprint commitment. |
| Acceptance | Given source net cost is 1020.00 and two open jobs exist; When I split 60 percent and 40 percent; Then allocations are 612.00 and 408.00 and sum exactly to the source. Assert: allocation_line sum(amount)=1020.00; amounts 612.00 and 408.00. |
| Negative tests | Wrong tenant →404; disallowed role →403; malformed decimal →422; stale revision →409; replay preserves one business action. |
| Definition of ready | Field mapping reviewed, fixture expected values labeled, control rule agreed, API schema reviewed and integration dependency proven. |
| Definition of done | Code review, required SIT and role tests pass; migration rehearsal succeeds; audit/metrics present; accessible UI reviewed; PM accepts demonstrated behavior. |

### JC-ENG-102 — Approval and sync

| Ticket field | Groomed decision |
| --- | --- |
| Business linkage | JC-BR-01/02/03 → JC-US-6 → JC-FR-6 → JC-SIT-06 |
| Goal | Enable the controller to approve balanced allocations for project export so they can control downstream cost changes. |
| Backend tasks | Implement POST /api/v1/cost-allocations/{id}/approve; bind current payload/source revision to authorization; create durable operation; reconcile ambiguous outcomes before retry; enforce AP remains the accounting system of record. Project entries use source-bill/line IDs to prevent a second financial expense; forecasts are separate from actuals. |
| Frontend tasks | Show exact reviewed values, target system or consumer, authorization role and last evidence update; disable a stale action; show accepted/processing/succeeded/failed states separately with an operation reference. |
| Data tasks | Store approval payload hash and approved_version; add unique logical-action key; persist any authorized external_write plus outbox atomically; store returned artifact hash or provider reference only on confirmed success. |
| Estimation | 8 story points = 48 planning hours: backend/adapter 20h, frontend 10h, data/integrity 8h, failure-injection and QA 10h. |
| Dependencies | JC-ENG-101 evidence gate, identity/roles, sandbox/consumer fixture and operation reconciliation contract. No live provider credentials in fixtures. |
| Acceptance | Given source revision 4 is current and allocations balance; When I approve the allocation; Then one export is queued using a stable source-line allocation reference. Assert: approval count=1; export_batch.source_revision=4. |
| Failure-path acceptance | Replayed commands return the existing operation; stale evidence invalidates authorization; 429 honors Retry-After; ambiguous timeout enters reconciliation; terminal failure has named recovery ownership. |
| Definition of ready | Target capability proof, deterministic fixture, payload mapping and review authority available; source data owner agrees to the semantics. |
| Definition of done | Contract and E2E sandbox/consumer checks pass; duplicate/replay and permissions verified; support can find and safely recover the operation; PM and BSA inspect evidence. |

## [PM & BSA] Two-week sprint plan

Reference Sprint 2 runs 2028-02-21–2028-03-03, ten working days. **Sprint goal:** demonstrate valid allocation through approval and sync for one representative tenant, including the blocked path and recovery evidence. This slice assumes foundational work from Sprint 1.

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
| 3–5 | Valid allocation service, UI and rule demonstration | Full-stack + AI/integration engineer | ENG-101 remaining 5 points |
| 6–8 | Approval and sync command, authorization and operation ledger | Integration + full-stack engineer | ENG-102 first 6 points |
| 9 | Failure/replay checks; complete UI and reconciliation evidence | BSA + engineers | ENG-102 remaining 2 + observability 2 points |
| 10 | Demonstration, targeted UAT rehearsal and retrospective | PM + BSA + customer process owner | Release evidence 2 points |

If a dependency fails, replace the blocked ticket only with a pre-refined item of equal or lower size that advances the sprint goal. PM records the scope decision; BSA records the failed technical assumption. No planned item is labeled delivered until acceptance evidence exists.

## [PM] Business and design triage log

These are simulated decisions for portfolio assessment. They are not reports of actual customer escalations.

| ID / severity | Roadblock | Resolution decision | Owner / success evidence |
| --- | --- | --- | --- |
| JC-PM-1 / High | Project managers include unapproved change orders in budget | Show pending changes separately; only approved budget drives variance. | PM + controller; validate in design-partner walkthrough before GA |
| JC-PM-2 / Medium | Finance and operations disagree on tax capitalization | Controller selects explicit tax policy per entity; version it on allocations. | PM + UX; validate in design-partner walkthrough before GA |
| JC-PM-3 / Medium | Customer expects payroll and resource planning | Limit scope to AP-derived project costs and reviewed forecasts. | PM + UX; validate in design-partner walkthrough before GA |

## [BSA] Data and API triage log

| ID / severity | Edge case | Exact technical resolution | Verification / owner |
| --- | --- | --- | --- |
| JC-BSA-1 / High | Rounding a three-way percentage split loses a cent | Use integer minor units and largest remainder; stable UUID ordering breaks ties. | Failure-injection fixture; verify source and target counts, state, audit and recovery behavior / integration lead |
| JC-BSA-2 / High | Source bill is edited after an export | Version every source line; generate delta adjustment and require review; never replay full amount. | Failure-injection fixture; verify source and target counts, state, audit and recovery behavior / integration lead |
| JC-BSA-3 / High | Same cost code label exists on different jobs | Key mapping by tenant, job and cost-code ID; reject cross-job code with 422 COST_CODE_JOB_MISMATCH. | Failure-injection fixture; verify source and target counts, state, audit and recovery behavior / integration lead |

### Cross-cutting connector recovery

For a 429, suspend only the affected tenant/connection queue for Retry-After seconds; otherwise use 30 seconds if that header is absent. Read/provider 5xx retries use full-jitter delays capped at 5, 30, 120, 300 and 900 seconds; after five failed executions route to the dead-letter recovery queue. Unknown write outcomes bypass blind retries and enter RECONCILING. OAuth refresh is serialized per connection; a revoked grant moves it to RECONNECT_REQUIRED and pauses dependent work. These rules also apply when the output is a consumer webhook or approved export rather than an ERP mutation.

**Escalation and ownership:** PM resolves intended business behavior and customer impact; BSA owns field/state diagnosis; engineering owns the code repair and recovery; the controller decides whether financial evidence is sufficient. A severity-1 integrity or isolation issue pauses the affected action path immediately and follows Phase 6 incident rules.
