# Procurement Digital Employee — Testing, QA & UAT

[Portfolio index](../README.md) · [Phase 1](01-ideation-and-strategy.md) · [Phase 2](02-roadmapping-and-scoping.md) · [Phase 3](03-requirements-prd-frd.md) · [Phase 4](04-sprint-execution-and-triage.md) · [Phase 5](05-testing-qa-and-uat.md) · [Phase 6](06-launch-gtm-and-telemetry.md)

**Document control:** PD-P5, revision 1.0, authored 2026-10-17. Status: proposed portfolio design; implementation, customer research, tests and stakeholder approvals have not been performed.

**Source boundary:** Named expansion, source p. 20: PO drafting from approved requests, vendor scorecards and catalog price watch attachment. [Primary document](../../MatchClerk-Project-Implementation.pdf). Proposed architecture and assumptions are explicitly recorded in [the evidence register](../source-and-assumptions.md).

## [BSA] Test strategy and controlled fixtures

This is an executable test **plan** and expected-state specification; no application exists in this workspace, so no SIT, security, load or UAT result is claimed. Test evidence must contain build identifier, schema migration version, synthetic fixture revision, run timestamp, observed result and reviewer. Planned acceptance gates differ from documentation validation performed on these Markdown artifacts.

| Layer | Scope | Method / expected result |
| --- | --- | --- |
| Unit/property checks | Decimal arithmetic, decision boundaries, state transitions and allocation invariants | Reserve budget in the same transaction as final request approval; release on rejection or cancellation. Requests ≥5000.00 require both manager and controller. A lower threshold never permits requester self-approval. |
| System integration | API → domain transaction → outbox/queue → analysis → approved output | Use isolated PostgreSQL/queue/object-store staging fixtures; assert domain and control tables. |
| Connector contracts | Provider sandbox or deterministic consumer adapter | Test valid mapping, unknown fields, permission failure, 429, revoked auth and ambiguous timeout. |
| Tenant and authorization | Two tenants, four roles, object URLs and pooled DB sessions | Zero cross-tenant disclosure, zero unauthorized approval, no stale-session tenant leak. |
| End-to-end | Operator intake through verified business outcome | Use the five UAT scenarios below with human review and target evidence. |
| Recovery and performance | Declared load, worker loss, queue redelivery and restored backup | Meet Phase 2 latency/capacity and RPO/RTO; prove no duplicate external logical action. |

**Fixture baseline:** Tenant-A and Tenant-B have disjoint memberships; operator, controller and auditor roles exist. Source vendor is V-77. Ready-to-act records start at record_version 4. Intake records start at revision 1. Monetary fixture values use USD for concrete arithmetic; tenant configuration determines allowed production currencies. Original and correction data are retained separately; provider tokens are sandbox-only.

## [BSA] Requirements traceability matrix

Each FR row traces to the exact API and acceptance scenario in Phase 3; database expectations are assertions to verify, not observed states. The control tables must commit atomically with each accepted mutation.

| Business requirement | FRD / story | API endpoint | SIT test case | Expected database state |
| --- | --- | --- | --- | --- |
| PD-BR-01/02 | PD-FR-1 / US-1 | POST /api/v1/purchase-requests | PD-SIT-01 | purchase_request.amount=4800.00; purchase_order count=0 |
| PD-BR-01/02 | PD-FR-2 / US-2 | POST /api/v1/purchase-requests/{id}/reserve-budget | PD-SIT-02 | budget_reservation sum=4800.00; budget.available=1200.00 |
| PD-BR-01/02 | PD-FR-3 / US-3 | POST /api/v1/purchase-requests/{id}/submit | PD-SIT-03 | request_approval required count=2; purchase_request.status='NEEDS_REVIEW' |
| PD-BR-01/02 | PD-FR-4 / US-4 | POST /api/v1/purchase-requests/{id}/draft-po | PD-SIT-04 | po_line.quantity=48.0000; po_line.unit_price=100.0000 |
| PD-BR-01/02 | PD-FR-5 / US-5 | POST /api/v1/purchase-requests/{id}/approve | PD-SIT-05 | purchase_order.external_id IS NOT NULL; external_write count=1 |
| PD-BR-01/02 | PD-FR-6 / US-6 | POST /api/v1/purchase-orders/{id}/dispatch-approval | PD-SIT-06 | supplier_dispatch.approved_by IS NOT NULL; outbox_event dispatch count=1 |
| PD-BR-02/04 | PD-FR-7 / US-7 | GET /api/v1/vendors/{id}/scorecard | PD-SIT-07 | vendor_scorecard.on_time_rate=0.8000; sample_count=10 |
| PD-BR-03 | PD-FR-8 / US-8 | POST /api/v1/purchase-requests/{id}/revalidate | PD-SIT-08 | purchase_request.status='NEEDS_REVIEW'; purchase_order.external_id IS NULL |

### SIT execution instructions

| Case | Steps | Pass condition / evidence |
| --- | --- | --- |
| PD-SIT-01 | 1. Seed: project JOB-42 and active vendor V-77 exist. 2. Invoke the mapped endpoint: I request 48 units at 100.00 USD. 3. Wait for its terminal or deliberate review state. 4. Inspect SQL rows, operation history and output. | a 4800.00 request with required-by date is recorded without creating a PO. Persist query output proving purchase_request.amount=4800.00; purchase_order count=0. |
| PD-SIT-02 | 1. Seed: budget remaining is 6000.00 and two requests each need 4800.00. 2. Invoke the mapped endpoint: both approval transactions run concurrently. 3. Wait for its terminal or deliberate review state. 4. Inspect SQL rows, operation history and output. | only one reserves 4800.00 and the other returns BUDGET_EXCEEDED. Persist query output proving budget_reservation sum=4800.00; budget.available=1200.00. |
| PD-SIT-03 | 1. Seed: a request amount is 5200.00. 2. Invoke the mapped endpoint: the requester submits it. 3. Wait for its terminal or deliberate review state. 4. Inspect SQL rows, operation history and output. | manager and controller tasks are created and the requester cannot satisfy either task. Persist query output proving request_approval required count=2; purchase_request.status='NEEDS_REVIEW'. |
| PD-SIT-04 | 1. Seed: REQ-2027-081 has approved quantity 48 and unit price 100.00. 2. Invoke the mapped endpoint: I generate a PO draft. 3. Wait for its terminal or deliberate review state. 4. Inspect SQL rows, operation history and output. | all source line references, quantities, account mappings and 4800.00 total match. Persist query output proving po_line.quantity=48.0000; po_line.unit_price=100.0000. |
| PD-SIT-05 | 1. Seed: an approved request has one internal PO and supported item references. 2. Invoke the mapped endpoint: I authorize publication with version 4. 3. Wait for its terminal or deliberate review state. 4. Inspect SQL rows, operation history and output. | one provider PO is created and its external ID is stored. Persist query output proving purchase_order.external_id IS NOT NULL; external_write count=1. |
| PD-SIT-06 | 1. Seed: a published PO has a reviewed recipient and PDF. 2. Invoke the mapped endpoint: I approve dispatch. 3. Wait for its terminal or deliberate review state. 4. Inspect SQL rows, operation history and output. | one dispatch event is queued and delivery state is recorded independently of PO publication. Persist query output proving supplier_dispatch.approved_by IS NOT NULL; outbox_event dispatch count=1. |
| PD-SIT-07 | 1. Seed: vendor V-77 has ten completed POs and eight on-time receipts. 2. Invoke the mapped endpoint: the scorecard refreshes. 3. Wait for its terminal or deliberate review state. 4. Inspect SQL rows, operation history and output. | on-time ratio is 0.80 with numerator eight, denominator ten and linked evidence. Persist query output proving vendor_scorecard.on_time_rate=0.8000; sample_count=10. |
| PD-SIT-08 | 1. Seed: a request was approved at 100.00 per unit. 2. Invoke the mapped endpoint: catalog price changes to 110.00 before publication. 3. Wait for its terminal or deliberate review state. 4. Inspect SQL rows, operation history and output. | the proposal is invalidated and requires revised budget reservation and approval. Persist query output proving purchase_request.status='NEEDS_REVIEW'; purchase_order.external_id IS NULL. |

| Control case | Failure injection / action | Expected result |
| --- | --- | --- |
| PD-SIT-09 | Repeat same action with same idempotency key, then change payload under that key | Same payload returns original result; changed payload returns 409 IDEMPOTENCY_CONFLICT; no second side effect. |
| PD-SIT-10 | Read/mutate Tenant-A record with Tenant-B session and a copied signed object URL | Record access returns 404; URL issuance denied; object route cannot expose another tenant object; no domain change. |
| PD-SIT-11 | Crash worker after target action but before local success persistence | Operation enters reconciliation; target is read by stable reference; ledger records one confirmed action. |
| PD-SIT-12 | Replay event and reorder source versions 4, 3, 5 | Consumer receipt prevents duplicate event execution; version 3 cannot overwrite 4; version 5 invalidates stale approvals. |
| PD-SIT-13 | Return malformed agent JSON, wrong units or invalid arithmetic | One schema repair attempt; unresolved values route to review; no invented missing business evidence. |
| PD-SIT-14 | Return 429 with Retry-After: 60 and revoke OAuth refresh | Affected connection waits ≥60 sec; revoked connection pauses with RECONNECT_REQUIRED; other tenants continue. |
| PD-SIT-15 | Run the declared 60-minute capacity profile with representative evidence sizes | Phase 2 API/agent/queue percentile and error targets pass; include failures/timeouts in the sample. |
| PD-SIT-16 | Restore a time-selected encrypted backup into isolated staging | RPO ≤15 min, RTO ≤4h; restored row counts, tenant policies, job/outbox lineage and evidence access reconcile. |

## [PM] Structured UAT plan

**Participants:** one daily operator (Operations requester), one accountable process owner (Procurement manager), one independent controller/approver and BSA as observer. PM owns acceptance decisions. Recruit two design-partner organizations and record each participant's role and consent. Run in staging/sandbox first; a consented production pilot uses parallel review and the same control boundaries.

**Entry criteria:** all Must stories demonstrated; SIT-01 through SIT-16 pass; no open severity-1/2 defect; pilot data consent and DPA confirmed; reviewers can access the source evidence; recovery path demonstrated. Budget two 90-minute sessions per organization. Each scenario gets two attempts after a defect fix; silently coaching through failures does not count as success.

| UAT ID / goal | Actor and setup | Step-by-step business exercise | Acceptance criteria |
| --- | --- | --- | --- |
| PD-UAT-1: Request intake | operations requester; project JOB-42 and active vendor V-77 exist | 1. Sign in with the intended role. 2. Open the source evidence and record baseline/version. 3. I request 48 units at 100.00 USD. 4. Inspect the result and decision reasons. 5. Open audit/output evidence and compare source values. 6. Record pass/fail and active handling time. | a 4800.00 request with required-by date is recorded without creating a PO. purchase_request.amount=4800.00; purchase_order count=0. No unexpected cross-tenant data or unapproved side effect. |
| PD-UAT-2: Approval policy | procurement manager; a request amount is 5200.00 | 1. Sign in with the intended role. 2. Open the source evidence and record baseline/version. 3. The requester submits it. 4. Inspect the result and decision reasons. 5. Open audit/output evidence and compare source values. 6. Record pass/fail and active handling time. | manager and controller tasks are created and the requester cannot satisfy either task. request_approval required count=2; purchase_request.status='NEEDS_REVIEW'. No unexpected cross-tenant data or unapproved side effect. |
| PD-UAT-3: Authorized write | procurement manager; an approved request has one internal PO and supported item references | 1. Sign in with the intended role. 2. Open the source evidence and record baseline/version. 3. I authorize publication with version 4. 4. Inspect the result and decision reasons. 5. Open audit/output evidence and compare source values. 6. Record pass/fail and active handling time. | one provider PO is created and its external ID is stored. purchase_order.external_id IS NOT NULL; external_write count=1. No unexpected cross-tenant data or unapproved side effect. |
| PD-UAT-4: Supplier dispatch | procurement manager; a published PO has a reviewed recipient and PDF | 1. Sign in with the intended role. 2. Open the source evidence and record baseline/version. 3. I approve dispatch. 4. Inspect the result and decision reasons. 5. Open audit/output evidence and compare source values. 6. Record pass/fail and active handling time. | one dispatch event is queued and delivery state is recorded independently of PO publication. supplier_dispatch.approved_by IS NOT NULL; outbox_event dispatch count=1. No unexpected cross-tenant data or unapproved side effect. |
| PD-UAT-5: Catalog drift recovery | controller; a request was approved at 100.00 per unit | 1. Sign in with the intended role. 2. Open the source evidence and record baseline/version. 3. Catalog price changes to 110.00 before publication. 4. Inspect the result and decision reasons. 5. Open audit/output evidence and compare source values. 6. Record pass/fail and active handling time. | the proposal is invalidated and requires revised budget reservation and approval. purchase_request.status='NEEDS_REVIEW'; purchase_order.external_id IS NULL. No unexpected cross-tenant data or unapproved side effect. |

**Product outcome gate:** Request-to-PO median <4 business hours; ≥95% PO line fidelity; ≥80% vendor scorecards have receipt/invoice-backed evidence. **Control gate:** Agents draft POs and supplier messages. Humans authorize financial commitment and outbound dispatch; no inferred approval from an email sentiment. **Measured usability gate:** at least 4/5 scenarios completed without coaching by each daily operator; no critical accessibility barrier; all financial/identity controls pass for every participant. A high usability rate cannot compensate for a control failure.

### Defect management and retest

| Severity | Example | Release rule / response |
| --- | --- | --- |
| S1 critical | Tenant leak, unauthorized action, duplicate financial commitment or corrupted allocation | Immediate stop; isolate path; architect and controller review; no release with open S1. |
| S2 high | Core flow blocked without safe workaround or wrong material decision | No release; fix and rerun impacted RTM rows plus authorization/replay controls. |
| S3 medium | Noncritical workflow defect with documented safe workaround | PM and process owner may accept dated remediation owner; BSA confirms control unaffected. |
| S4 low | Minor visual issue without accessibility or decision impact | Track in backlog; does not bypass required critical-flow accessibility. |

Capture record ID, request ID, fixture, actual/expected behavior, redacted screenshot, control impact and reproducible steps in each issue. BSA links defects to FR/SIT IDs; PM links them to the affected business scenario. Reopen an accepted story when a mandatory acceptance criterion fails.

## [PM & BSA] Formal Go/No-Go checklist

| Readiness owner | Required release evidence | Pass threshold | Current sign-off state |
| --- | --- | --- | --- |
| BSA technical | RTM and domain rules | SIT-01…16 pass; expected DB state and operation side effects verified | Not executed; release approval withheld |
| BSA technical | Integration semantics and replay | Sandbox/consumer contract proven; no unknown write outcome or blind duplicate create | Not executed; release approval withheld |
| BSA technical | Security, tenant and role controls | Zero open S1/S2, zero tenant disclosure, MFA/RLS/object access checks pass | Not executed; release approval withheld |
| BSA technical | Performance and recovery | Phase 2 targets pass; restore drill within 14 days of release | Not executed; release approval withheld |
| BSA technical | Observability and rollback | Dashboard, paging route, feature switch and queue pause/reconciliation drill proven | Not executed; release approval withheld |
| PM business | UAT and value evidence | All five scenarios accepted; product outcome gate evaluated against labeled cohort | Not executed; release approval withheld |
| PM business | Commercial and adoption readiness | Named ICP, signed pilot terms, reviewed pricing and support ownership | Not executed; release approval withheld |
| PM business | Customer-facing readiness | Onboarding, accessible approval flow, training and escalation instructions reviewed | Not executed; release approval withheld |
| PM + controller | Control boundary | Human action ownership and excluded capabilities accepted in writing | Not executed; release approval withheld |

**Go rule:** PM business approval and BSA technical approval are both required against the same build/artifact revision, with customer controller approval for financial controls. Either role can issue No-Go. An actual sign-off stores name, role, UTC timestamp and evidence links. All statuses above accurately indicate that this portfolio is a design and test plan, not a launched service.
