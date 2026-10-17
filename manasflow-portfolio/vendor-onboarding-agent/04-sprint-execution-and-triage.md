# Vendor Onboarding Agent — Sprint Execution & Triage

[Portfolio index](../README.md) · [Phase 1](01-ideation-and-strategy.md) · [Phase 2](02-roadmapping-and-scoping.md) · [Phase 3](03-requirements-prd-frd.md) · [Phase 4](04-sprint-execution-and-triage.md) · [Phase 5](05-testing-qa-and-uat.md) · [Phase 6](06-launch-gtm-and-telemetry.md)

**Document control:** VO-P4, revision 1.0, authored 2026-10-17. Status: proposed portfolio design; implementation, customer research, tests and stakeholder approvals have not been performed.

**Source boundary:** Named parallel upsell, source p. 14. The source provides the name only; structured evidence collection and controlled vendor-master synchronization are proposed scope. [Primary document](../../MatchClerk-Project-Implementation.pdf). Proposed architecture and assumptions are explicitly recorded in [the evidence register](../source-and-assumptions.md).

## [PM & BSA] Refinement and working agreement

Refinement inputs are the approved PRD/FRD revision, a synthetic fixture pack, connector capability evidence and the ranked scope in Phase 2. PM explains the outcome and protects the scope; BSA walks field mappings, state transitions and failure paths; engineers estimate after discovering dependencies. The following are fully specified proposed tickets, not historical sprint results.

### VO-ENG-101 — Checklist validation

| Ticket field | Groomed decision |
| --- | --- |
| Business linkage | VO-BR-01/02 → VO-US-3 → VO-FR-3 → VO-SIT-03 |
| Goal | Enable the controller to verify mandatory supplier evidence so they can avoid approving incomplete registration. |
| Backend tasks | Implement POST /api/v1/vendor-applications/{id}/checks; validate tenant, role, source identity and expected_version; evaluate: Required legal-name/contact fields present; document evidence verified by a human; duplicate candidates resolved; intake actor differs from approver; external vendor mapping rechecked. |
| Frontend tasks | Render vendor application evidence, calculated decision reasons and proposed result; show Missing evidence, possible duplicate identity, conflicting vendor record or expired review → NEEDS_REVIEW; bank-detail changes are excluded.; preserve edited values on validation errors and support keyboard-only review. |
| Data tasks | Migrate vendor_application and its dependent evidence tables from the ERD; enforce tenant composite foreign keys; write work_record, audit_event and outbox_event atomically; add tenant/status/time index. |
| Estimation | 8 story points = 48 planning hours: backend 18h, frontend 12h, data/migrations 8h, integration/QA 10h. Planning ratio is six productive hours per point. |
| Dependencies | Requires vendor/contact read-write connector capability, an approved evidence checklist and independent controller reviewer. Authentication, tenant RLS and source fixture must be available before sprint commitment. |
| Acceptance | Given legal name is present but the contact address is missing; When I run the verification checklist; Then the application remains NEEDS_REVIEW with CONTACT_ADDRESS_MISSING and cannot be approved. Assert: verification_check.check_code='CONTACT_ADDRESS'; verification_check.passed=false. |
| Negative tests | Wrong tenant →404; disallowed role →403; malformed decimal →422; stale revision →409; replay preserves one business action. |
| Definition of ready | Field mapping reviewed, fixture expected values labeled, control rule agreed, API schema reviewed and integration dependency proven. |
| Definition of done | Code review, required SIT and role tests pass; migration rehearsal succeeds; audit/metrics present; accessible UI reviewed; PM accepts demonstrated behavior. |

### VO-ENG-102 — Vendor sync

| Ticket field | Groomed decision |
| --- | --- |
| Business linkage | VO-BR-01/02/03 → VO-US-6 → VO-FR-6 → VO-SIT-06 |
| Goal | Enable the controller to synchronize the approved vendor identity once so they can make the supplier available to invoice processing. |
| Backend tasks | Implement POST /api/v1/vendor-applications/{id}/sync; bind current payload/source revision to authorization; create durable operation; reconcile ambiguous outcomes before retry; enforce No bank account collection or update. A human verifies supplier evidence and approves the exact vendor-master payload; the agent cannot certify legal or tax compliance. |
| Frontend tasks | Show exact reviewed values, target system or consumer, authorization role and last evidence update; disable a stale action; show accepted/processing/succeeded/failed states separately with an operation reference. |
| Data tasks | Store approval payload hash and approved_version; add unique logical-action key; persist any authorized external_write plus outbox atomically; store returned artifact hash or provider reference only on confirmed success. |
| Estimation | 8 story points = 48 planning hours: backend/adapter 20h, frontend 10h, data/integrity 8h, failure-injection and QA 10h. |
| Dependencies | VO-ENG-101 evidence gate, identity/roles, sandbox/consumer fixture and operation reconciliation contract. No live provider credentials in fixtures. |
| Acceptance | Given the approved application has no bank fields and no unresolved duplicate; When the connector submits the authorized vendor/contact payload; Then one external vendor ID is stored and no financial transaction is created. Assert: vendor_mapping.external_vendor_id IS NOT NULL; sync_request.status='SUCCEEDED'. |
| Failure-path acceptance | Replayed commands return the existing operation; stale evidence invalidates authorization; 429 honors Retry-After; ambiguous timeout enters reconciliation; terminal failure has named recovery ownership. |
| Definition of ready | Target capability proof, deterministic fixture, payload mapping and review authority available; source data owner agrees to the semantics. |
| Definition of done | Contract and E2E sandbox/consumer checks pass; duplicate/replay and permissions verified; support can find and safely recover the operation; PM and BSA inspect evidence. |

## [PM & BSA] Two-week sprint plan

Reference Sprint 2 runs 2027-06-21–2027-07-02, ten working days. **Sprint goal:** demonstrate checklist validation through vendor sync for one representative tenant, including the blocked path and recovery evidence. This slice assumes foundational work from Sprint 1.

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
| 3–5 | Checklist validation service, UI and rule demonstration | Full-stack + AI/integration engineer | ENG-101 remaining 5 points |
| 6–8 | Vendor sync command, authorization and operation ledger | Integration + full-stack engineer | ENG-102 first 6 points |
| 9 | Failure/replay checks; complete UI and reconciliation evidence | BSA + engineers | ENG-102 remaining 2 + observability 2 points |
| 10 | Demonstration, targeted UAT rehearsal and retrospective | PM + BSA + customer process owner | Release evidence 2 points |

If a dependency fails, replace the blocked ticket only with a pre-refined item of equal or lower size that advances the sprint goal. PM records the scope decision; BSA records the failed technical assumption. No planned item is labeled delivered until acceptance evidence exists.

## [PM] Business and design triage log

These are simulated decisions for portfolio assessment. They are not reports of actual customer escalations.

| ID / severity | Roadblock | Resolution decision | Owner / success evidence |
| --- | --- | --- | --- |
| VO-PM-1 / High | AP expects the agent to certify supplier compliance | Use an approved evidence checklist and independent human verification; no legal/tax certification claim. | PM + controller; validate in design-partner walkthrough before GA |
| VO-PM-2 / Medium | Supplier insists on sending bank details | Exclude bank fields from intake and redirect to the customer-controlled treasury process. | PM + UX; validate in design-partner walkthrough before GA |
| VO-PM-3 / Medium | Finance wants reused supplier data across entities | Require explicit entity scope, evidence review and separate external mapping before reuse. | PM + UX; validate in design-partner walkthrough before GA |

## [BSA] Data and API triage log

| ID / severity | Edge case | Exact technical resolution | Verification / owner |
| --- | --- | --- | --- |
| VO-BSA-1 / High | Similar legal names belong to different suppliers | Create a duplicate candidate only; use external ID and evidence to resolve; no fuzzy automatic merge. | Failure-injection fixture; verify source and target counts, state, audit and recovery behavior / integration lead |
| VO-BSA-2 / High | Identity changes after approval but before vendor create | Compare canonical payload hash and record_version at execution; block stale sync and require fresh approval. | Failure-injection fixture; verify source and target counts, state, audit and recovery behavior / integration lead |
| VO-BSA-3 / High | Provider created a vendor but response timed out | Enter RECONCILING; query the stable approved identity and operation evidence; require operator resolution if ambiguous rather than blind creation. | Failure-injection fixture; verify source and target counts, state, audit and recovery behavior / integration lead |

### Cross-cutting connector recovery

For a 429, suspend only the affected tenant/connection queue for Retry-After seconds; otherwise use 30 seconds if that header is absent. Read/provider 5xx retries use full-jitter delays capped at 5, 30, 120, 300 and 900 seconds; after five failed executions route to the dead-letter recovery queue. Unknown write outcomes bypass blind retries and enter RECONCILING. OAuth refresh is serialized per connection; a revoked grant moves it to RECONNECT_REQUIRED and pauses dependent work. These rules also apply when the output is a consumer webhook or approved export rather than an ERP mutation.

**Escalation and ownership:** PM resolves intended business behavior and customer impact; BSA owns field/state diagnosis; engineering owns the code repair and recovery; the controller decides whether financial evidence is sufficient. A severity-1 integrity or isolation issue pauses the affected action path immediately and follows Phase 6 incident rules.
