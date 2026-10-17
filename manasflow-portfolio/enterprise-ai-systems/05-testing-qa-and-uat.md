# Enterprise AI Systems — Testing, QA & UAT

[Portfolio index](../README.md) · [Phase 1](01-ideation-and-strategy.md) · [Phase 2](02-roadmapping-and-scoping.md) · [Phase 3](03-requirements-prd-frd.md) · [Phase 4](04-sprint-execution-and-triage.md) · [Phase 5](05-testing-qa-and-uat.md) · [Phase 6](06-launch-gtm-and-telemetry.md)

**Document control:** ES-P5, revision 1.0, authored 2026-10-17. Status: proposed portfolio design; implementation, customer research, tests and stakeholder approvals have not been performed.

**Source boundary:** Service/solution tier, source p. 20: cross-department AP/inventory/project hub, job-costing sync and custom multi-agent retainers. Reference architecture is proposed. [Primary document](../../MatchClerk-Project-Implementation.pdf). Proposed architecture and assumptions are explicitly recorded in [the evidence register](../source-and-assumptions.md).

## [BSA] Test strategy and controlled fixtures

This is an executable test **plan** and expected-state specification; no application exists in this workspace, so no SIT, security, load or UAT result is claimed. Test evidence must contain build identifier, schema migration version, synthetic fixture revision, run timestamp, observed result and reviewer. Planned acceptance gates differ from documentation validation performed on these Markdown artifacts.

| Layer | Scope | Method / expected result |
| --- | --- | --- |
| Unit/property checks | Decimal arithmetic, decision boundaries, state transitions and allocation invariants | Use a durable workflow engine for multi-day approval waits and compensation. Every tool action has a typed schema, tenant/entity scope, stable idempotency reference and immutable authorization evidence. Partial completion is visible; compensation is a separate authorized action. |
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
| ES-BR-01/02 | ES-FR-1 / US-1 | POST /api/v1/workflow-definitions | ES-SIT-01 | workflow_definition.version=3; workflow_definition.status='PUBLISHED' |
| ES-BR-01/02 | ES-FR-2 / US-2 | POST /api/v1/workflow-runs | ES-SIT-02 | workflow_run.correlation_key='FLOW-PO501-R1'; source_event count=1 |
| ES-BR-01/02 | ES-FR-3 / US-3 | POST /api/v1/workflow-runs/{id}/policy-evaluations | ES-SIT-03 | tool_action.status='DENIED'; external_write count=0 |
| ES-BR-01/02 | ES-FR-4 / US-4 | POST /api/v1/workflow-runs/{id}/approve | ES-SIT-04 | approval.payload_hash equals tool_action.payload_hash |
| ES-BR-01/02 | ES-FR-5 / US-5 | POST /api/v1/workflow-runs/{id}/budget-evaluations | ES-SIT-05 | workflow_run.cost_usd=0.49; tool_action.status='BUDGET_BLOCKED' |
| ES-BR-01/02 | ES-FR-6 / US-6 | POST /api/v1/workflow-runs/{id}/resume | ES-SIT-06 | workflow_step A remains SUCCEEDED; external_write for A count=1 |
| ES-BR-02/04 | ES-FR-7 / US-7 | GET /api/v1/workflow-runs/{id}/outcomes | ES-SIT-07 | workflow_run.status='COMPLETED'; outcome_check.passed=true |
| ES-BR-03 | ES-FR-8 / US-8 | POST /api/v1/workflow-runs/{id}/compensation-plans | ES-SIT-08 | workflow_run.status='PAUSED_REVIEW'; compensation_plan.status='PROPOSED' |

### SIT execution instructions

| Case | Steps | Pass condition / evidence |
| --- | --- | --- |
| ES-SIT-01 | 1. Seed: AP-to-inventory workflow version 3 passed review. 2. Invoke the mapped endpoint: I publish its signed schema and action policy. 3. Wait for its terminal or deliberate review state. 4. Inspect SQL rows, operation history and output. | only version 3 is active for the selected tenant/entity scope. Persist query output proving workflow_definition.version=3; workflow_definition.status='PUBLISHED'. |
| ES-SIT-02 | 1. Seed: an event carries schema_version 2 and an allowed entity. 2. Invoke the mapped endpoint: its signature and schema are verified. 3. Wait for its terminal or deliberate review state. 4. Inspect SQL rows, operation history and output. | one workflow run is created with source hash and correlation key. Persist query output proving workflow_run.correlation_key='FLOW-PO501-R1'; source_event count=1. |
| ES-SIT-03 | 1. Seed: a planner proposes a bank transfer tool absent from its allowlist. 2. Invoke the mapped endpoint: the policy service evaluates the proposal. 3. Wait for its terminal or deliberate review state. 4. Inspect SQL rows, operation history and output. | the action is denied and the run pauses with no external call. Persist query output proving tool_action.status='DENIED'; external_write count=0. |
| ES-SIT-04 | 1. Seed: a valid inventory adjustment proposal references current source version 4. 2. Invoke the mapped endpoint: an authorized controller approves it. 3. Wait for its terminal or deliberate review state. 4. Inspect SQL rows, operation history and output. | the exact signed payload is authorized and later edits require a new approval. Persist query output proving approval.payload_hash equals tool_action.payload_hash. |
| ES-SIT-05 | 1. Seed: a run has a 0.50 USD model budget and accumulated cost 0.49. 2. Invoke the mapped endpoint: the next call is estimated at 0.05. 3. Wait for its terminal or deliberate review state. 4. Inspect SQL rows, operation history and output. | the call is blocked and the run enters PAUSED_REVIEW. Persist query output proving workflow_run.cost_usd=0.49; tool_action.status='BUDGET_BLOCKED'. |
| ES-SIT-06 | 1. Seed: action A completed and action B awaits approval. 2. Invoke the mapped endpoint: the worker restarts. 3. Wait for its terminal or deliberate review state. 4. Inspect SQL rows, operation history and output. | the workflow resumes from the persisted wait and action A is not repeated. Persist query output proving workflow_step A remains SUCCEEDED; external_write for A count=1. |
| ES-SIT-07 | 1. Seed: all steps report success with external references. 2. Invoke the mapped endpoint: the final reconciliation runs. 3. Wait for its terminal or deliberate review state. 4. Inspect SQL rows, operation history and output. | the run is COMPLETED only after target-state checks pass. Persist query output proving workflow_run.status='COMPLETED'; outcome_check.passed=true. |
| ES-SIT-08 | 1. Seed: inventory updated but project sync failed. 2. Invoke the mapped endpoint: the retry limit is exhausted. 3. Wait for its terminal or deliberate review state. 4. Inspect SQL rows, operation history and output. | the run pauses with a proposed compensation and no unapproved reversal. Persist query output proving workflow_run.status='PAUSED_REVIEW'; compensation_plan.status='PROPOSED'. |

| Control case | Failure injection / action | Expected result |
| --- | --- | --- |
| ES-SIT-09 | Repeat same action with same idempotency key, then change payload under that key | Same payload returns original result; changed payload returns 409 IDEMPOTENCY_CONFLICT; no second side effect. |
| ES-SIT-10 | Read/mutate Tenant-A record with Tenant-B session and a copied signed object URL | Record access returns 404; URL issuance denied; object route cannot expose another tenant object; no domain change. |
| ES-SIT-11 | Crash worker after target action but before local success persistence | Operation enters reconciliation; target is read by stable reference; ledger records one confirmed action. |
| ES-SIT-12 | Replay event and reorder source versions 4, 3, 5 | Consumer receipt prevents duplicate event execution; version 3 cannot overwrite 4; version 5 invalidates stale approvals. |
| ES-SIT-13 | Return malformed agent JSON, wrong units or invalid arithmetic | One schema repair attempt; unresolved values route to review; no invented missing business evidence. |
| ES-SIT-14 | Return 429 with Retry-After: 60 and revoke OAuth refresh | Affected connection waits ≥60 sec; revoked connection pauses with RECONNECT_REQUIRED; other tenants continue. |
| ES-SIT-15 | Run the declared 60-minute capacity profile with representative evidence sizes | Phase 2 API/agent/queue percentile and error targets pass; include failures/timeouts in the sample. |
| ES-SIT-16 | Restore a time-selected encrypted backup into isolated staging | RPO ≤15 min, RTO ≤4h; restored row counts, tenant policies, job/outbox lineage and evidence access reconcile. |

## [PM] Structured UAT plan

**Participants:** one daily operator (IT integration owner), one accountable process owner (Finance process owner), one independent controller/approver and BSA as observer. PM owns acceptance decisions. Recruit two design-partner organizations and record each participant's role and consent. Run in staging/sandbox first; a consented production pilot uses parallel review and the same control boundaries.

**Entry criteria:** all Must stories demonstrated; SIT-01 through SIT-16 pass; no open severity-1/2 defect; pilot data consent and DPA confirmed; reviewers can access the source evidence; recovery path demonstrated. Budget two 90-minute sessions per organization. Each scenario gets two attempts after a defect fix; silently coaching through failures does not count as success.

| UAT ID / goal | Actor and setup | Step-by-step business exercise | Acceptance criteria |
| --- | --- | --- | --- |
| ES-UAT-1: Workflow registration | IT integration owner; AP-to-inventory workflow version 3 passed review | 1. Sign in with the intended role. 2. Open the source evidence and record baseline/version. 3. I publish its signed schema and action policy. 4. Inspect the result and decision reasons. 5. Open audit/output evidence and compare source values. 6. Record pass/fail and active handling time. | only version 3 is active for the selected tenant/entity scope. workflow_definition.version=3; workflow_definition.status='PUBLISHED'. No unexpected cross-tenant data or unapproved side effect. |
| ES-UAT-2: Tool restrictions | finance process owner; a planner proposes a bank transfer tool absent from its allowlist | 1. Sign in with the intended role. 2. Open the source evidence and record baseline/version. 3. The policy service evaluates the proposal. 4. Inspect the result and decision reasons. 5. Open audit/output evidence and compare source values. 6. Record pass/fail and active handling time. | the action is denied and the run pauses with no external call. tool_action.status='DENIED'; external_write count=0. No unexpected cross-tenant data or unapproved side effect. |
| ES-UAT-3: Budget guard | IT integration owner; a run has a 0.50 USD model budget and accumulated cost 0.49 | 1. Sign in with the intended role. 2. Open the source evidence and record baseline/version. 3. The next call is estimated at 0.05. 4. Inspect the result and decision reasons. 5. Open audit/output evidence and compare source values. 6. Record pass/fail and active handling time. | the call is blocked and the run enters PAUSED_REVIEW. workflow_run.cost_usd=0.49; tool_action.status='BUDGET_BLOCKED'. No unexpected cross-tenant data or unapproved side effect. |
| ES-UAT-4: Durable execution | operations transformation lead; action A completed and action B awaits approval | 1. Sign in with the intended role. 2. Open the source evidence and record baseline/version. 3. The worker restarts. 4. Inspect the result and decision reasons. 5. Open audit/output evidence and compare source values. 6. Record pass/fail and active handling time. | the workflow resumes from the persisted wait and action A is not repeated. workflow_step A remains SUCCEEDED; external_write for A count=1. No unexpected cross-tenant data or unapproved side effect. |
| ES-UAT-5: Compensation review | IT integration owner; inventory updated but project sync failed | 1. Sign in with the intended role. 2. Open the source evidence and record baseline/version. 3. The retry limit is exhausted. 4. Inspect the result and decision reasons. 5. Open audit/output evidence and compare source values. 6. Record pass/fail and active handling time. | the run pauses with a proposed compensation and no unapproved reversal. workflow_run.status='PAUSED_REVIEW'; compensation_plan.status='PROPOSED'. No unexpected cross-tenant data or unapproved side effect. |

**Product outcome gate:** ≥99% schema-valid intake; ≥95% successful business completion excluding human rejection; zero unapproved external mutations; ≥80% runs within approved cost budget. **Control gate:** Custom engagement, not a claim of a shipped universal ERP product. Agents propose typed actions; a deterministic policy service authorizes execution. Banking tools are excluded. **Measured usability gate:** at least 4/5 scenarios completed without coaching by each daily operator; no critical accessibility barrier; all financial/identity controls pass for every participant. A high usability rate cannot compensate for a control failure.

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
