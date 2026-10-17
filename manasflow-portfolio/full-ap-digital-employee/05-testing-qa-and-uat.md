# Full AP Digital Employee — Testing, QA & UAT

[Portfolio index](../README.md) · [Phase 1](01-ideation-and-strategy.md) · [Phase 2](02-roadmapping-and-scoping.md) · [Phase 3](03-requirements-prd-frd.md) · [Phase 4](04-sprint-execution-and-triage.md) · [Phase 5](05-testing-qa-and-uat.md) · [Phase 6](06-launch-gtm-and-telemetry.md)

**Document control:** FA-P5, revision 1.0, authored 2026-10-17. Status: proposed portfolio design; implementation, customer research, tests and stakeholder approvals have not been performed.

**Source boundary:** Umbrella upsell named on source p. 2, supported by the expansion path on p. 20. Proposed as an orchestration bundle, not a second invoice engine. [Primary document](../../MatchClerk-Project-Implementation.pdf). Proposed architecture and assumptions are explicitly recorded in [the evidence register](../source-and-assumptions.md).

## [BSA] Test strategy and controlled fixtures

This is an executable test **plan** and expected-state specification; no application exists in this workspace, so no SIT, security, load or UAT result is claimed. Test evidence must contain build identifier, schema migration version, synthetic fixture revision, run timestamp, observed result and reviewer. Planned acceptance gates differ from documentation validation performed on these Markdown artifacts.

| Layer | Scope | Method / expected result |
| --- | --- | --- |
| Unit/property checks | Decimal arithmetic, decision boundaries, state transitions and allocation invariants | State derives from component revisions, not just cached labels. A credit-note case reopening invalidates the parent release. Vendor onboarding validates evidence through an independent controller; bank details remain outside the product write scope. |
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
| FA-BR-01/02 | FA-FR-1 / US-1 | POST /api/v1/ap-work-items | FA-SIT-01 | component_task count=2; ap_work_item.workflow_reference='APCYCLE-2027-06' |
| FA-BR-01/02 | FA-FR-2 / US-2 | PATCH /api/v1/ap-work-items/{id}/assignment | FA-SIT-02 | component_task.assigned_to IS NOT NULL; due_at IS NOT NULL |
| FA-BR-01/02 | FA-FR-3 / US-3 | POST /api/v1/ap-work-items/{id}/vendor-verifications | FA-SIT-03 | vendor_verification.status='VERIFIED'; external_write count=0 |
| FA-BR-01/02 | FA-FR-4 / US-4 | POST /api/v1/ap-work-items/{id}/evaluate | FA-SIT-04 | ap_work_item.status='BLOCKED'; component_task.source_status='HELD' |
| FA-BR-01/02 | FA-FR-5 / US-5 | POST /api/v1/ap-work-items/{id}/prepare | FA-SIT-05 | control_snapshot.manifest_hash IS NOT NULL; ap_work_item.status='PREPARED' |
| FA-BR-01/02 | FA-FR-6 / US-6 | POST /api/v1/ap-work-items/{id}/approve | FA-SIT-06 | approval count=1; outbox_event.event_type='ap.cycle_approved' |
| FA-BR-02/04 | FA-FR-7 / US-7 | GET /api/v1/ap-work-items/{id}/audit | FA-SIT-07 | control_snapshot.component_count=3; audit_event append-only |
| FA-BR-03 | FA-FR-8 / US-8 | POST /api/v1/webhooks/components | FA-SIT-08 | ap_work_item.status='BLOCKED'; control_snapshot.invalidated_at IS NOT NULL |

### SIT execution instructions

| Case | Steps | Pass condition / evidence |
| --- | --- | --- |
| FA-SIT-01 | 1. Seed: MatchClerk emits one invoice hold and the exception agent emits one case. 2. Invoke the mapped endpoint: both signed events arrive. 3. Wait for its terminal or deliberate review state. 4. Inspect SQL rows, operation history and output. | two linked tasks appear under one AP cycle with original source IDs. Persist query output proving component_task count=2; ap_work_item.workflow_reference='APCYCLE-2027-06'. |
| FA-SIT-02 | 1. Seed: a cycle contains an unassigned task. 2. Invoke the mapped endpoint: I assign Leena and a due timestamp. 3. Wait for its terminal or deliberate review state. 4. Inspect SQL rows, operation history and output. | the task records one owner and overdue monitoring uses the tenant business calendar. Persist query output proving component_task.assigned_to IS NOT NULL; due_at IS NOT NULL. |
| FA-SIT-03 | 1. Seed: a clerk submitted vendor registration evidence. 2. Invoke the mapped endpoint: a controller reviews the evidence. 3. Wait for its terminal or deliberate review state. 4. Inspect SQL rows, operation history and output. | verification status becomes VERIFIED and no bank master update is sent. Persist query output proving vendor_verification.status='VERIFIED'; external_write count=0. |
| FA-SIT-04 | 1. Seed: one linked exception has risk HIGH and state HELD. 2. Invoke the mapped endpoint: a user requests cycle completion. 3. Wait for its terminal or deliberate review state. 4. Inspect SQL rows, operation history and output. | the cycle becomes BLOCKED and identifies the exact source case. Persist query output proving ap_work_item.status='BLOCKED'; component_task.source_status='HELD'. |
| FA-SIT-05 | 1. Seed: all component tasks are complete and evidence revisions match. 2. Invoke the mapped endpoint: I prepare the cycle. 3. Wait for its terminal or deliberate review state. 4. Inspect SQL rows, operation history and output. | a fixed control snapshot and component revision manifest are stored. Persist query output proving control_snapshot.manifest_hash IS NOT NULL; ap_work_item.status='PREPARED'. |
| FA-SIT-06 | 1. Seed: the preparer and controller are different authenticated users. 2. Invoke the mapped endpoint: the controller approves current version 4. 3. Wait for its terminal or deliberate review state. 4. Inspect SQL rows, operation history and output. | one approval is stored and a pack handoff is queued without bill creation. Persist query output proving approval count=1; outbox_event.event_type='ap.cycle_approved'. |
| FA-SIT-07 | 1. Seed: a completed cycle has three component revisions. 2. Invoke the mapped endpoint: I open the control snapshot. 3. Wait for its terminal or deliberate review state. 4. Inspect SQL rows, operation history and output. | all source references and approver identity are visible with read-only access. Persist query output proving control_snapshot.component_count=3; audit_event append-only. |
| FA-SIT-08 | 1. Seed: a cycle was prepared on case revision 3. 2. Invoke the mapped endpoint: case revision 4 reopens a hold. 3. Wait for its terminal or deliberate review state. 4. Inspect SQL rows, operation history and output. | the prior proposal becomes BLOCKED and cannot be approved. Persist query output proving ap_work_item.status='BLOCKED'; control_snapshot.invalidated_at IS NOT NULL. |

| Control case | Failure injection / action | Expected result |
| --- | --- | --- |
| FA-SIT-09 | Repeat same action with same idempotency key, then change payload under that key | Same payload returns original result; changed payload returns 409 IDEMPOTENCY_CONFLICT; no second side effect. |
| FA-SIT-10 | Read/mutate Tenant-A record with Tenant-B session and a copied signed object URL | Record access returns 404; URL issuance denied; object route cannot expose another tenant object; no domain change. |
| FA-SIT-11 | Crash worker after target action but before local success persistence | Operation enters reconciliation; target is read by stable reference; ledger records one confirmed action. |
| FA-SIT-12 | Replay event and reorder source versions 4, 3, 5 | Consumer receipt prevents duplicate event execution; version 3 cannot overwrite 4; version 5 invalidates stale approvals. |
| FA-SIT-13 | Return malformed agent JSON, wrong units or invalid arithmetic | One schema repair attempt; unresolved values route to review; no invented missing business evidence. |
| FA-SIT-14 | Return 429 with Retry-After: 60 and revoke OAuth refresh | Affected connection waits ≥60 sec; revoked connection pauses with RECONNECT_REQUIRED; other tenants continue. |
| FA-SIT-15 | Run the declared 60-minute capacity profile with representative evidence sizes | Phase 2 API/agent/queue percentile and error targets pass; include failures/timeouts in the sample. |
| FA-SIT-16 | Restore a time-selected encrypted backup into isolated staging | RPO ≤15 min, RTO ≤4h; restored row counts, tenant policies, job/outbox lineage and evidence access reconcile. |

## [PM] Structured UAT plan

**Participants:** one daily operator (Controller), one accountable process owner (AP team lead), one independent controller/approver and BSA as observer. PM owns acceptance decisions. Recruit two design-partner organizations and record each participant's role and consent. Run in staging/sandbox first; a consented production pilot uses parallel review and the same control boundaries.

**Entry criteria:** all Must stories demonstrated; SIT-01 through SIT-16 pass; no open severity-1/2 defect; pilot data consent and DPA confirmed; reviewers can access the source evidence; recovery path demonstrated. Budget two 90-minute sessions per organization. Each scenario gets two attempts after a defect fix; silently coaching through failures does not count as success.

| UAT ID / goal | Actor and setup | Step-by-step business exercise | Acceptance criteria |
| --- | --- | --- | --- |
| FA-UAT-1: Unified queue | AP team lead; MatchClerk emits one invoice hold and the exception agent emits one case | 1. Sign in with the intended role. 2. Open the source evidence and record baseline/version. 3. Both signed events arrive. 4. Inspect the result and decision reasons. 5. Open audit/output evidence and compare source values. 6. Record pass/fail and active handling time. | two linked tasks appear under one AP cycle with original source IDs. component_task count=2; ap_work_item.workflow_reference='APCYCLE-2027-06'. No unexpected cross-tenant data or unapproved side effect. |
| FA-UAT-2: Vendor verification | controller; a clerk submitted vendor registration evidence | 1. Sign in with the intended role. 2. Open the source evidence and record baseline/version. 3. A controller reviews the evidence. 4. Inspect the result and decision reasons. 5. Open audit/output evidence and compare source values. 6. Record pass/fail and active handling time. | verification status becomes VERIFIED and no bank master update is sent. vendor_verification.status='VERIFIED'; external_write count=0. No unexpected cross-tenant data or unapproved side effect. |
| FA-UAT-3: Cycle preparation | AP team lead; all component tasks are complete and evidence revisions match | 1. Sign in with the intended role. 2. Open the source evidence and record baseline/version. 3. I prepare the cycle. 4. Inspect the result and decision reasons. 5. Open audit/output evidence and compare source values. 6. Record pass/fail and active handling time. | a fixed control snapshot and component revision manifest are stored. control_snapshot.manifest_hash IS NOT NULL; ap_work_item.status='PREPARED'. No unexpected cross-tenant data or unapproved side effect. |
| FA-UAT-4: Separate approval | controller; the preparer and controller are different authenticated users | 1. Sign in with the intended role. 2. Open the source evidence and record baseline/version. 3. The controller approves current version 4. 4. Inspect the result and decision reasons. 5. Open audit/output evidence and compare source values. 6. Record pass/fail and active handling time. | one approval is stored and a pack handoff is queued without bill creation. approval count=1; outbox_event.event_type='ap.cycle_approved'. No unexpected cross-tenant data or unapproved side effect. |
| FA-UAT-5: Reopened case | controller; a cycle was prepared on case revision 3 | 1. Sign in with the intended role. 2. Open the source evidence and record baseline/version. 3. Case revision 4 reopens a hold. 4. Inspect the result and decision reasons. 5. Open audit/output evidence and compare source values. 6. Record pass/fail and active handling time. | the prior proposal becomes BLOCKED and cannot be approved. ap_work_item.status='BLOCKED'; control_snapshot.invalidated_at IS NOT NULL. No unexpected cross-tenant data or unapproved side effect. |

**Product outcome gate:** ≥30% fewer aged AP work items; ≥90% tasks have a named owner; zero approval bypass across component holds. **Control gate:** This bundle delegates to MatchClerk and AP Exception Agent APIs. It never independently writes a duplicate bill or executes a bank payment. **Measured usability gate:** at least 4/5 scenarios completed without coaching by each daily operator; no critical accessibility barrier; all financial/identity controls pass for every participant. A high usability rate cannot compensate for a control failure.

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
