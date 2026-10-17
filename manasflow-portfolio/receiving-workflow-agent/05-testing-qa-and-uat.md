# Receiving Workflow Agent — Testing, QA & UAT

[Portfolio index](../README.md) · [Phase 1](01-ideation-and-strategy.md) · [Phase 2](02-roadmapping-and-scoping.md) · [Phase 3](03-requirements-prd-frd.md) · [Phase 4](04-sprint-execution-and-triage.md) · [Phase 5](05-testing-qa-and-uat.md) · [Phase 6](06-launch-gtm-and-telemetry.md)

**Document control:** RW-P5, revision 1.0, authored 2026-10-17. Status: proposed portfolio design; implementation, customer research, tests and stakeholder approvals have not been performed.

**Source boundary:** Expansion offer, source p. 20: proposed when AWAITING_RECEIPT volume is high. Detailed receipt workflow is an architectural proposal. [Primary document](../../MatchClerk-Project-Implementation.pdf). Proposed architecture and assumptions are explicitly recorded in [the evidence register](../source-and-assumptions.md).

## [BSA] Test strategy and controlled fixtures

This is an executable test **plan** and expected-state specification; no application exists in this workspace, so no SIT, security, load or UAT result is claimed. Test evidence must contain build identifier, schema migration version, synthetic fixture revision, run timestamp, observed result and reviewer. Planned acceptance gates differ from documentation validation performed on these Markdown artifacts.

| Layer | Scope | Method / expected result |
| --- | --- | --- |
| Unit/property checks | Decimal arithmetic, decision boundaries, state transitions and allocation invariants | accepted_qty = delivered_qty - damaged_qty. Partial receipts accumulate per PO line. Returns are append-only negative adjustments linked to the original receipt; never erase a historical attestation. |
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
| RW-BR-01/02 | RW-FR-1 / US-1 | POST /api/v1/receipts | RW-SIT-01 | goods_receipt.purchase_order_id references PO-501 |
| RW-BR-01/02 | RW-FR-2 / US-2 | POST /api/v1/receipts/{id}/extract | RW-SIT-02 | goods_receipt.status='NEEDS_REVIEW'; receipt_line.delivered_qty=8.0000 |
| RW-BR-01/02 | RW-FR-3 / US-3 | POST /api/v1/receipts/{id}/attest | RW-SIT-03 | receipt_line.accepted_qty=7.0000; po_line.qty_received=7.0000 |
| RW-BR-01/02 | RW-FR-4 / US-4 | POST /api/v1/receipts/{id}/evaluate | RW-SIT-04 | goods_receipt.status='NEEDS_REVIEW'; outbox release count=0 |
| RW-BR-01/02 | RW-FR-5 / US-5 | POST /api/v1/receipts/{id}/adjustments | RW-SIT-05 | receipt_adjustment.quantity_delta=-1.0000; receipt_line.accepted_qty remains 7.0000 |
| RW-BR-01/02 | RW-FR-6 / US-6 | POST /api/v1/receipts/{id}/publish | RW-SIT-06 | outbox_event.event_type='receipt.attested'; consumption_receipt count=1 |
| RW-BR-02/04 | RW-FR-7 / US-7 | GET /api/v1/receipts/{id} | RW-SIT-07 | audit_event.action='ACCESS_DENIED'; goods_receipt unchanged |
| RW-BR-03 | RW-FR-8 / US-8 | POST /api/v1/receipts/offline-sync | RW-SIT-08 | UNIQUE(tenant_id,client_event_id) prevents a second goods_receipt |

### SIT execution instructions

| Case | Steps | Pass condition / evidence |
| --- | --- | --- |
| RW-SIT-01 | 1. Seed: PO-501 has ten units ordered. 2. Invoke the mapped endpoint: I select vendor V-77 and PO-501. 3. Wait for its terminal or deliberate review state. 4. Inspect SQL rows, operation history and output. | only that tenant and site open lines appear with remaining quantity ten. Persist query output proving goods_receipt.purchase_order_id references PO-501. |
| RW-SIT-02 | 1. Seed: a clean slip shows eight units delivered. 2. Invoke the mapped endpoint: the extraction agent reads the image. 3. Wait for its terminal or deliberate review state. 4. Inspect SQL rows, operation history and output. | eight units appear as a proposal with confidence and require my attestation. Persist query output proving goods_receipt.status='NEEDS_REVIEW'; receipt_line.delivered_qty=8.0000. |
| RW-SIT-03 | 1. Seed: ten units are ordered and eight arrived with one damaged. 2. Invoke the mapped endpoint: I attest delivered eight and damaged one. 3. Wait for its terminal or deliberate review state. 4. Inspect SQL rows, operation history and output. | accepted quantity is seven and the remaining unreceived quantity is three. Persist query output proving receipt_line.accepted_qty=7.0000; po_line.qty_received=7.0000. |
| RW-SIT-04 | 1. Seed: PO remaining quantity is three. 2. Invoke the mapped endpoint: a receiver submits four accepted units with zero tolerance. 3. Wait for its terminal or deliberate review state. 4. Inspect SQL rows, operation history and output. | the receipt enters NEEDS_REVIEW with a one-unit excess and no rematch release. Persist query output proving goods_receipt.status='NEEDS_REVIEW'; outbox release count=0. |
| RW-SIT-05 | 1. Seed: receipt R1 incorrectly attested seven accepted units. 2. Invoke the mapped endpoint: I add a negative one-unit adjustment with reason. 3. Wait for its terminal or deliberate review state. 4. Inspect SQL rows, operation history and output. | the original stays immutable and net accepted quantity becomes six. Persist query output proving receipt_adjustment.quantity_delta=-1.0000; receipt_line.accepted_qty remains 7.0000. |
| RW-SIT-06 | 1. Seed: invoice INV-1042 bills seven units and accepted quantity is seven. 2. Invoke the mapped endpoint: the attestation event commits. 3. Wait for its terminal or deliberate review state. 4. Inspect SQL rows, operation history and output. | one rematch event is consumed and invoice matching re-evaluates quantity coverage. Persist query output proving outbox_event.event_type='receipt.attested'; consumption_receipt count=1. |
| RW-SIT-07 | 1. Seed: a receiver belongs only to Site-A. 2. Invoke the mapped endpoint: they request a Site-B receipt. 3. Wait for its terminal or deliberate review state. 4. Inspect SQL rows, operation history and output. | the API returns 404 without exposing the supplier or PO. Persist query output proving audit_event.action='ACCESS_DENIED'; goods_receipt unchanged. |
| RW-SIT-08 | 1. Seed: a local draft has client_event_id DEL-8801-A. 2. Invoke the mapped endpoint: the device submits it twice. 3. Wait for its terminal or deliberate review state. 4. Inspect SQL rows, operation history and output. | one receipt is committed and both responses identify the same receipt. Persist query output proving UNIQUE(tenant_id,client_event_id) prevents a second goods_receipt. |

| Control case | Failure injection / action | Expected result |
| --- | --- | --- |
| RW-SIT-09 | Repeat same action with same idempotency key, then change payload under that key | Same payload returns original result; changed payload returns 409 IDEMPOTENCY_CONFLICT; no second side effect. |
| RW-SIT-10 | Read/mutate Tenant-A record with Tenant-B session and a copied signed object URL | Record access returns 404; URL issuance denied; object route cannot expose another tenant object; no domain change. |
| RW-SIT-11 | Crash worker after target action but before local success persistence | Operation enters reconciliation; target is read by stable reference; ledger records one confirmed action. |
| RW-SIT-12 | Replay event and reorder source versions 4, 3, 5 | Consumer receipt prevents duplicate event execution; version 3 cannot overwrite 4; version 5 invalidates stale approvals. |
| RW-SIT-13 | Return malformed agent JSON, wrong units or invalid arithmetic | One schema repair attempt; unresolved values route to review; no invented missing business evidence. |
| RW-SIT-14 | Return 429 with Retry-After: 60 and revoke OAuth refresh | Affected connection waits ≥60 sec; revoked connection pauses with RECONNECT_REQUIRED; other tenants continue. |
| RW-SIT-15 | Run the declared 60-minute capacity profile with representative evidence sizes | Phase 2 API/agent/queue percentile and error targets pass; include failures/timeouts in the sample. |
| RW-SIT-16 | Restore a time-selected encrypted backup into isolated staging | RPO ≤15 min, RTO ≤4h; restored row counts, tenant policies, job/outbox lineage and evidence access reconcile. |

## [PM] Structured UAT plan

**Participants:** one daily operator (Receiving lead), one accountable process owner (Operations manager), one independent controller/approver and BSA as observer. PM owns acceptance decisions. Recruit two design-partner organizations and record each participant's role and consent. Run in staging/sandbox first; a consented production pilot uses parallel review and the same control boundaries.

**Entry criteria:** all Must stories demonstrated; SIT-01 through SIT-16 pass; no open severity-1/2 defect; pilot data consent and DPA confirmed; reviewers can access the source evidence; recovery path demonstrated. Budget two 90-minute sessions per organization. Each scenario gets two attempts after a defect fix; silently coaching through failures does not count as success.

| UAT ID / goal | Actor and setup | Step-by-step business exercise | Acceptance criteria |
| --- | --- | --- | --- |
| RW-UAT-1: PO lookup | receiving lead; PO-501 has ten units ordered | 1. Sign in with the intended role. 2. Open the source evidence and record baseline/version. 3. I select vendor V-77 and PO-501. 4. Inspect the result and decision reasons. 5. Open audit/output evidence and compare source values. 6. Record pass/fail and active handling time. | only that tenant and site open lines appear with remaining quantity ten. goods_receipt.purchase_order_id references PO-501. No unexpected cross-tenant data or unapproved side effect. |
| RW-UAT-2: Partial attestation | receiving lead; ten units are ordered and eight arrived with one damaged | 1. Sign in with the intended role. 2. Open the source evidence and record baseline/version. 3. I attest delivered eight and damaged one. 4. Inspect the result and decision reasons. 5. Open audit/output evidence and compare source values. 6. Record pass/fail and active handling time. | accepted quantity is seven and the remaining unreceived quantity is three. receipt_line.accepted_qty=7.0000; po_line.qty_received=7.0000. No unexpected cross-tenant data or unapproved side effect. |
| RW-UAT-3: Evidence correction | receiving lead; receipt R1 incorrectly attested seven accepted units | 1. Sign in with the intended role. 2. Open the source evidence and record baseline/version. 3. I add a negative one-unit adjustment with reason. 4. Inspect the result and decision reasons. 5. Open audit/output evidence and compare source values. 6. Record pass/fail and active handling time. | the original stays immutable and net accepted quantity becomes six. receipt_adjustment.quantity_delta=-1.0000; receipt_line.accepted_qty remains 7.0000. No unexpected cross-tenant data or unapproved side effect. |
| RW-UAT-4: Unblock matching | AP clerk; invoice INV-1042 bills seven units and accepted quantity is seven | 1. Sign in with the intended role. 2. Open the source evidence and record baseline/version. 3. The attestation event commits. 4. Inspect the result and decision reasons. 5. Open audit/output evidence and compare source values. 6. Record pass/fail and active handling time. | one rematch event is consumed and invoice matching re-evaluates quantity coverage. outbox_event.event_type='receipt.attested'; consumption_receipt count=1. No unexpected cross-tenant data or unapproved side effect. |
| RW-UAT-5: Offline replay | receiving lead; a local draft has client_event_id DEL-8801-A | 1. Sign in with the intended role. 2. Open the source evidence and record baseline/version. 3. The device submits it twice. 4. Inspect the result and decision reasons. 5. Open audit/output evidence and compare source values. 6. Record pass/fail and active handling time. | one receipt is committed and both responses identify the same receipt. UNIQUE(tenant_id,client_event_id) prevents a second goods_receipt. No unexpected cross-tenant data or unapproved side effect. |

**Product outcome gate:** ≥95% attested receipts link to correct PO line; receipt evidence latency p95 <1 business day; zero billed allocation above accepted quantity. **Control gate:** OCR proposes quantities; staff attest physical receipt. A receipt does not create an accounting liability or payment. **Measured usability gate:** at least 4/5 scenarios completed without coaching by each daily operator; no critical accessibility barrier; all financial/identity controls pass for every participant. A high usability rate cannot compensate for a control failure.

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
