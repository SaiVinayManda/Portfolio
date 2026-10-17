# Enterprise AI Systems — Sprint Execution & Triage

[Portfolio index](../README.md) · [Phase 1](01-ideation-and-strategy.md) · [Phase 2](02-roadmapping-and-scoping.md) · [Phase 3](03-requirements-prd-frd.md) · [Phase 4](04-sprint-execution-and-triage.md) · [Phase 5](05-testing-qa-and-uat.md) · [Phase 6](06-launch-gtm-and-telemetry.md)

**Document control:** ES-P4, revision 1.0, authored 2026-10-17. Status: proposed portfolio design; implementation, customer research, tests and stakeholder approvals have not been performed.

**Source boundary:** Service/solution tier, source p. 20: cross-department AP/inventory/project hub, job-costing sync and custom multi-agent retainers. Reference architecture is proposed. [Primary document](../../MatchClerk-Project-Implementation.pdf). Proposed architecture and assumptions are explicitly recorded in [the evidence register](../source-and-assumptions.md).

## [PM & BSA] Refinement and working agreement

Refinement inputs are the approved PRD/FRD revision, a synthetic fixture pack, connector capability evidence and the ranked scope in Phase 2. PM explains the outcome and protects the scope; BSA walks field mappings, state transitions and failure paths; engineers estimate after discovering dependencies. The following are fully specified proposed tickets, not historical sprint results.

### ES-ENG-101 — Tool restrictions

| Ticket field | Groomed decision |
| --- | --- |
| Business linkage | ES-BR-01/02 → ES-US-3 → ES-FR-3 → ES-SIT-03 |
| Goal | Enable the finance process owner to deny tools outside the approved policy so they can prevent unintended system actions. |
| Backend tasks | Implement POST /api/v1/workflow-runs/{id}/policy-evaluations; validate tenant, role, source identity and expected_version; evaluate: Workflow version published; tenant and entity scopes valid; source schema verified; tool allowlist satisfied; cost budget available; human approval precedes every external mutation. |
| Frontend tasks | Render workflow run evidence, calculated decision reasons and proposed result; show Unsupported schema, denied tool, cost cap, stale approval or failed compensation → PAUSED_REVIEW; no autonomous widening of permissions.; preserve edited values on validation errors and support keyboard-only review. |
| Data tasks | Migrate workflow_run and its dependent evidence tables from the ERD; enforce tenant composite foreign keys; write work_record, audit_event and outbox_event atomically; add tenant/status/time index. |
| Estimation | 8 story points = 48 planning hours: backend 18h, frontend 12h, data/migrations 8h, integration/QA 10h. Planning ratio is six productive hours per point. |
| Dependencies | Requires an agreed statement of work, named data owners, connector sandboxes and signed workflow control policies. Authentication, tenant RLS and source fixture must be available before sprint commitment. |
| Acceptance | Given a planner proposes a bank transfer tool absent from its allowlist; When the policy service evaluates the proposal; Then the action is denied and the run pauses with no external call. Assert: tool_action.status='DENIED'; external_write count=0. |
| Negative tests | Wrong tenant →404; disallowed role →403; malformed decimal →422; stale revision →409; replay preserves one business action. |
| Definition of ready | Field mapping reviewed, fixture expected values labeled, control rule agreed, API schema reviewed and integration dependency proven. |
| Definition of done | Code review, required SIT and role tests pass; migration rehearsal succeeds; audit/metrics present; accessible UI reviewed; PM accepts demonstrated behavior. |

### ES-ENG-102 — Durable execution

| Ticket field | Groomed decision |
| --- | --- |
| Business linkage | ES-BR-01/02/03 → ES-US-6 → ES-FR-6 → ES-SIT-06 |
| Goal | Enable the operations transformation lead to resume an approved workflow after a worker restart so they can avoid duplicated business actions. |
| Backend tasks | Implement POST /api/v1/workflow-runs/{id}/resume; bind current payload/source revision to authorization; create durable operation; reconcile ambiguous outcomes before retry; enforce Custom engagement, not a claim of a shipped universal ERP product. Agents propose typed actions; a deterministic policy service authorizes execution. Banking tools are excluded. |
| Frontend tasks | Show exact reviewed values, target system or consumer, authorization role and last evidence update; disable a stale action; show accepted/processing/succeeded/failed states separately with an operation reference. |
| Data tasks | Store approval payload hash and approved_version; add unique logical-action key; persist any authorized external_write plus outbox atomically; store returned artifact hash or provider reference only on confirmed success. |
| Estimation | 8 story points = 48 planning hours: backend/adapter 20h, frontend 10h, data/integrity 8h, failure-injection and QA 10h. |
| Dependencies | ES-ENG-101 evidence gate, identity/roles, sandbox/consumer fixture and operation reconciliation contract. No live provider credentials in fixtures. |
| Acceptance | Given action A completed and action B awaits approval; When the worker restarts; Then the workflow resumes from the persisted wait and action A is not repeated. Assert: workflow_step A remains SUCCEEDED; external_write for A count=1. |
| Failure-path acceptance | Replayed commands return the existing operation; stale evidence invalidates authorization; 429 honors Retry-After; ambiguous timeout enters reconciliation; terminal failure has named recovery ownership. |
| Definition of ready | Target capability proof, deterministic fixture, payload mapping and review authority available; source data owner agrees to the semantics. |
| Definition of done | Contract and E2E sandbox/consumer checks pass; duplicate/replay and permissions verified; support can find and safely recover the operation; PM and BSA inspect evidence. |

## [PM & BSA] Two-week sprint plan

Reference Sprint 2 runs 2028-04-24–2028-05-05, ten working days. **Sprint goal:** demonstrate tool restrictions through durable execution for one representative tenant, including the blocked path and recovery evidence. This slice assumes foundational work from Sprint 1.

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
| 3–5 | Tool restrictions service, UI and rule demonstration | Full-stack + AI/integration engineer | ENG-101 remaining 5 points |
| 6–8 | Durable execution command, authorization and operation ledger | Integration + full-stack engineer | ENG-102 first 6 points |
| 9 | Failure/replay checks; complete UI and reconciliation evidence | BSA + engineers | ENG-102 remaining 2 + observability 2 points |
| 10 | Demonstration, targeted UAT rehearsal and retrospective | PM + BSA + customer process owner | Release evidence 2 points |

If a dependency fails, replace the blocked ticket only with a pre-refined item of equal or lower size that advances the sprint goal. PM records the scope decision; BSA records the failed technical assumption. No planned item is labeled delivered until acceptance evidence exists.

## [PM] Business and design triage log

These are simulated decisions for portfolio assessment. They are not reports of actual customer escalations.

| ID / severity | Roadblock | Resolution decision | Owner / success evidence |
| --- | --- | --- | --- |
| ES-PM-1 / High | Sponsor asks for a universal autonomous ERP layer | Contract two named workflows and explicit connector capabilities per statement of work. | PM + controller; validate in design-partner walkthrough before GA |
| ES-PM-2 / Medium | Departments dispute ownership of cross-system data | Assign one authoritative owner per field and an escalation owner per workflow. | PM + UX; validate in design-partner walkthrough before GA |
| ES-PM-3 / Medium | IT wants every process in the first pilot | Select AP receipt-to-project handoff; expand only after outcome and control gates pass. | PM + UX; validate in design-partner walkthrough before GA |

## [BSA] Data and API triage log

| ID / severity | Edge case | Exact technical resolution | Verification / owner |
| --- | --- | --- | --- |
| ES-BSA-1 / High | External schema changes without version notice | Validate schema at ingress; quarantine unknown versions; pin adapter contract and notify integration owner. | Failure-injection fixture; verify source and target counts, state, audit and recovery behavior / integration lead |
| ES-BSA-2 / High | A retry arrives after partial cross-system success | Use durable workflow history, per-action ledger and target reconciliation; never claim distributed exactly-once execution. | Failure-injection fixture; verify source and target counts, state, audit and recovery behavior / integration lead |
| ES-BSA-3 / High | Agent invents a tool or modifies an approved payload | Reject tool name outside allowlist; compare canonical payload hash with signed approval; return 403 POLICY_DENIED. | Failure-injection fixture; verify source and target counts, state, audit and recovery behavior / integration lead |

### Cross-cutting connector recovery

For a 429, suspend only the affected tenant/connection queue for Retry-After seconds; otherwise use 30 seconds if that header is absent. Read/provider 5xx retries use full-jitter delays capped at 5, 30, 120, 300 and 900 seconds; after five failed executions route to the dead-letter recovery queue. Unknown write outcomes bypass blind retries and enter RECONCILING. OAuth refresh is serialized per connection; a revoked grant moves it to RECONNECT_REQUIRED and pauses dependent work. These rules also apply when the output is a consumer webhook or approved export rather than an ERP mutation.

**Escalation and ownership:** PM resolves intended business behavior and customer impact; BSA owns field/state diagnosis; engineering owns the code repair and recovery; the controller decides whether financial evidence is sufficient. A severity-1 integrity or isolation issue pauses the affected action path immediately and follows Phase 6 incident rules.
