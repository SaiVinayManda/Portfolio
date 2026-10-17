# Job-Costing / Project Ops Agents — Testing, QA & UAT

[Portfolio index](../README.md) · [Phase 1](01-ideation-and-strategy.md) · [Phase 2](02-roadmapping-and-scoping.md) · [Phase 3](03-requirements-prd-frd.md) · [Phase 4](04-sprint-execution-and-triage.md) · [Phase 5](05-testing-qa-and-uat.md) · [Phase 6](06-launch-gtm-and-telemetry.md)

**Document control:** JC-P5, revision 1.0, authored 2026-10-17. Status: proposed portfolio design; implementation, customer research, tests and stakeholder approvals have not been performed.

**Source boundary:** Expansion capability, source p. 20: contractor job-costing sync and job-costing/project-ops agent offer. Detailed allocation controls are proposed. [Primary document](../../MatchClerk-Project-Implementation.pdf). Proposed architecture and assumptions are explicitly recorded in [the evidence register](../source-and-assumptions.md).

## [BSA] Test strategy and controlled fixtures

This is an executable test **plan** and expected-state specification; no application exists in this workspace, so no SIT, security, load or UAT result is claimed. Test evidence must contain build identifier, schema migration version, synthetic fixture revision, run timestamp, observed result and reviewer. Planned acceptance gates differ from documentation validation performed on these Markdown artifacts.

| Layer | Scope | Method / expected result |
| --- | --- | --- |
| Unit/property checks | Decimal arithmetic, decision boundaries, state transitions and allocation invariants | Allocate tax-exclusive net line cost to job budgets by default; distribute tax separately only when tenant policy capitalizes it. Percentage splits round in minor units using largest remainder with stable tie-breaking by allocation UUID. |
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
| JC-BR-01/02 | JC-FR-1 / US-1 | POST /api/v1/jobs/sync | JC-SIT-01 | job.job_code='JOB-42'; cost_code.code='MAT-01' |
| JC-BR-01/02 | JC-FR-2 / US-2 | POST /api/v1/cost-allocations | JC-SIT-02 | source_cost_line.external_bill_id='B-1042'; source_cost_line count=1 |
| JC-BR-01/02 | JC-FR-3 / US-3 | PATCH /api/v1/cost-allocations/{id}/lines | JC-SIT-03 | allocation_line sum(amount)=1020.00; amounts 612.00 and 408.00 |
| JC-BR-01/02 | JC-FR-4 / US-4 | POST /api/v1/cost-allocations/{id}/evaluate | JC-SIT-04 | cost_allocation.status='NEEDS_REVIEW'; external_write count=0 |
| JC-BR-01/02 | JC-FR-5 / US-5 | GET /api/v1/jobs/{id}/budget | JC-SIT-05 | job_budget.actual_cost=11000.00; variance=1000.00 |
| JC-BR-01/02 | JC-FR-6 / US-6 | POST /api/v1/cost-allocations/{id}/approve | JC-SIT-06 | approval count=1; export_batch.source_revision=4 |
| JC-BR-02/04 | JC-FR-7 / US-7 | GET /api/v1/cost-allocations/{id}/audit | JC-SIT-07 | audit_event.action='ALLOCATION_APPROVED'; export_batch.reconciliation_total=1020.00 |
| JC-BR-03 | JC-FR-8 / US-8 | POST /api/v1/cost-allocations/{id}/source-revisions | JC-SIT-08 | cost_adjustment.delta_amount=-20.00; original export unchanged |

### SIT execution instructions

| Case | Steps | Pass condition / evidence |
| --- | --- | --- |
| JC-SIT-01 | 1. Seed: JOB-42 is open and cost code MAT-01 is active. 2. Invoke the mapped endpoint: the job master refreshes. 3. Wait for its terminal or deliberate review state. 4. Inspect SQL rows, operation history and output. | both mappings appear with source revision and sync timestamp. Persist query output proving job.job_code='JOB-42'; cost_code.code='MAT-01'. |
| JC-SIT-02 | 1. Seed: bill B-1042 line 1 is approved for 1020.00. 2. Invoke the mapped endpoint: its event is replayed twice. 3. Wait for its terminal or deliberate review state. 4. Inspect SQL rows, operation history and output. | one source line exists and its provider, bill ID and line ID remain visible. Persist query output proving source_cost_line.external_bill_id='B-1042'; source_cost_line count=1. |
| JC-SIT-03 | 1. Seed: source net cost is 1020.00 and two open jobs exist. 2. Invoke the mapped endpoint: I split 60 percent and 40 percent. 3. Wait for its terminal or deliberate review state. 4. Inspect SQL rows, operation history and output. | allocations are 612.00 and 408.00 and sum exactly to the source. Persist query output proving allocation_line sum(amount)=1020.00; amounts 612.00 and 408.00. |
| JC-SIT-04 | 1. Seed: JOB-42 is closed. 2. Invoke the mapped endpoint: an allocation targets its MAT-01 code. 3. Wait for its terminal or deliberate review state. 4. Inspect SQL rows, operation history and output. | the allocation enters NEEDS_REVIEW and no project export is queued. Persist query output proving cost_allocation.status='NEEDS_REVIEW'; external_write count=0. |
| JC-SIT-05 | 1. Seed: approved budget is 10000.00 and actual net costs reach 11000.00. 2. Invoke the mapped endpoint: the dashboard refreshes. 3. Wait for its terminal or deliberate review state. 4. Inspect SQL rows, operation history and output. | a 1000.00 adverse variance and 10 percent overrun are displayed without mixing forecasts. Persist query output proving job_budget.actual_cost=11000.00; variance=1000.00. |
| JC-SIT-06 | 1. Seed: source revision 4 is current and allocations balance. 2. Invoke the mapped endpoint: I approve the allocation. 3. Wait for its terminal or deliberate review state. 4. Inspect SQL rows, operation history and output. | one export is queued using a stable source-line allocation reference. Persist query output proving approval count=1; export_batch.source_revision=4. |
| JC-SIT-07 | 1. Seed: an exported allocation totals 1020.00. 2. Invoke the mapped endpoint: I inspect lineage. 3. Wait for its terminal or deliberate review state. 4. Inspect SQL rows, operation history and output. | source bill, line, allocation split, tax policy and reviewer are visible. Persist query output proving audit_event.action='ALLOCATION_APPROVED'; export_batch.reconciliation_total=1020.00. |
| JC-SIT-08 | 1. Seed: 1020.00 was already exported and source changes to 1000.00. 2. Invoke the mapped endpoint: the revised event arrives. 3. Wait for its terminal or deliberate review state. 4. Inspect SQL rows, operation history and output. | a negative 20.00 adjustment proposal is created for review and the original export stays immutable. Persist query output proving cost_adjustment.delta_amount=-20.00; original export unchanged. |

| Control case | Failure injection / action | Expected result |
| --- | --- | --- |
| JC-SIT-09 | Repeat same action with same idempotency key, then change payload under that key | Same payload returns original result; changed payload returns 409 IDEMPOTENCY_CONFLICT; no second side effect. |
| JC-SIT-10 | Read/mutate Tenant-A record with Tenant-B session and a copied signed object URL | Record access returns 404; URL issuance denied; object route cannot expose another tenant object; no domain change. |
| JC-SIT-11 | Crash worker after target action but before local success persistence | Operation enters reconciliation; target is read by stable reference; ledger records one confirmed action. |
| JC-SIT-12 | Replay event and reorder source versions 4, 3, 5 | Consumer receipt prevents duplicate event execution; version 3 cannot overwrite 4; version 5 invalidates stale approvals. |
| JC-SIT-13 | Return malformed agent JSON, wrong units or invalid arithmetic | One schema repair attempt; unresolved values route to review; no invented missing business evidence. |
| JC-SIT-14 | Return 429 with Retry-After: 60 and revoke OAuth refresh | Affected connection waits ≥60 sec; revoked connection pauses with RECONNECT_REQUIRED; other tenants continue. |
| JC-SIT-15 | Run the declared 60-minute capacity profile with representative evidence sizes | Phase 2 API/agent/queue percentile and error targets pass; include failures/timeouts in the sample. |
| JC-SIT-16 | Restore a time-selected encrypted backup into isolated staging | RPO ≤15 min, RTO ≤4h; restored row counts, tenant policies, job/outbox lineage and evidence access reconcile. |

## [PM] Structured UAT plan

**Participants:** one daily operator (Project accountant), one accountable process owner (Project manager), one independent controller/approver and BSA as observer. PM owns acceptance decisions. Recruit two design-partner organizations and record each participant's role and consent. Run in staging/sandbox first; a consented production pilot uses parallel review and the same control boundaries.

**Entry criteria:** all Must stories demonstrated; SIT-01 through SIT-16 pass; no open severity-1/2 defect; pilot data consent and DPA confirmed; reviewers can access the source evidence; recovery path demonstrated. Budget two 90-minute sessions per organization. Each scenario gets two attempts after a defect fix; silently coaching through failures does not count as success.

| UAT ID / goal | Actor and setup | Step-by-step business exercise | Acceptance criteria |
| --- | --- | --- | --- |
| JC-UAT-1: Master sync | project accountant; JOB-42 is open and cost code MAT-01 is active | 1. Sign in with the intended role. 2. Open the source evidence and record baseline/version. 3. The job master refreshes. 4. Inspect the result and decision reasons. 5. Open audit/output evidence and compare source values. 6. Record pass/fail and active handling time. | both mappings appear with source revision and sync timestamp. job.job_code='JOB-42'; cost_code.code='MAT-01'. No unexpected cross-tenant data or unapproved side effect. |
| JC-UAT-2: Valid allocation | project accountant; source net cost is 1020.00 and two open jobs exist | 1. Sign in with the intended role. 2. Open the source evidence and record baseline/version. 3. I split 60 percent and 40 percent. 4. Inspect the result and decision reasons. 5. Open audit/output evidence and compare source values. 6. Record pass/fail and active handling time. | allocations are 612.00 and 408.00 and sum exactly to the source. allocation_line sum(amount)=1020.00; amounts 612.00 and 408.00. No unexpected cross-tenant data or unapproved side effect. |
| JC-UAT-3: Budget variance | project manager; approved budget is 10000.00 and actual net costs reach 11000.00 | 1. Sign in with the intended role. 2. Open the source evidence and record baseline/version. 3. The dashboard refreshes. 4. Inspect the result and decision reasons. 5. Open audit/output evidence and compare source values. 6. Record pass/fail and active handling time. | a 1000.00 adverse variance and 10 percent overrun are displayed without mixing forecasts. job_budget.actual_cost=11000.00; variance=1000.00. No unexpected cross-tenant data or unapproved side effect. |
| JC-UAT-4: Approval and sync | controller; source revision 4 is current and allocations balance | 1. Sign in with the intended role. 2. Open the source evidence and record baseline/version. 3. I approve the allocation. 4. Inspect the result and decision reasons. 5. Open audit/output evidence and compare source values. 6. Record pass/fail and active handling time. | one export is queued using a stable source-line allocation reference. approval count=1; export_batch.source_revision=4. No unexpected cross-tenant data or unapproved side effect. |
| JC-UAT-5: Bill revision recovery | project accountant; 1020.00 was already exported and source changes to 1000.00 | 1. Sign in with the intended role. 2. Open the source evidence and record baseline/version. 3. The revised event arrives. 4. Inspect the result and decision reasons. 5. Open audit/output evidence and compare source values. 6. Record pass/fail and active handling time. | a negative 20.00 adjustment proposal is created for review and the original export stays immutable. cost_adjustment.delta_amount=-20.00; original export unchanged. No unexpected cross-tenant data or unapproved side effect. |

**Product outcome gate:** Allocation conservation 100%; ≥95% valid job/code mappings; project sync freshness p95 <15 minutes; zero duplicated source cost. **Control gate:** AP remains the accounting system of record. Project entries use source-bill/line IDs to prevent a second financial expense; forecasts are separate from actuals. **Measured usability gate:** at least 4/5 scenarios completed without coaching by each daily operator; no critical accessibility barrier; all financial/identity controls pass for every participant. A high usability rate cannot compensate for a control failure.

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
