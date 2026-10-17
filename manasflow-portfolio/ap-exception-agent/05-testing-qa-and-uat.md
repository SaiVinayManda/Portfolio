# AP Exception Agent — Testing, QA & UAT

[Portfolio index](../README.md) · [Phase 1](01-ideation-and-strategy.md) · [Phase 2](02-roadmapping-and-scoping.md) · [Phase 3](03-requirements-prd-frd.md) · [Phase 4](04-sprint-execution-and-triage.md) · [Phase 5](05-testing-qa-and-uat.md) · [Phase 6](06-launch-gtm-and-telemetry.md)

**Document control:** AE-P5, revision 1.0, authored 2026-10-17. Status: proposed portfolio design; implementation, customer research, tests and stakeholder approvals have not been performed.

**Source boundary:** Named expansion, source p. 20: statement reconciliation, duplicate/fraud flags and payment-run preparation. Remaining details are proposed. [Primary document](../../MatchClerk-Project-Implementation.pdf). Proposed architecture and assumptions are explicitly recorded in [the evidence register](../source-and-assumptions.md).

## [BSA] Test strategy and controlled fixtures

This is an executable test **plan** and expected-state specification; no application exists in this workspace, so no SIT, security, load or UAT result is claimed. Test evidence must contain build identifier, schema migration version, synthetic fixture revision, run timestamp, observed result and reviewer. Planned acceptance gates differ from documentation validation performed on these Markdown artifacts.

| Layer | Scope | Method / expected result |
| --- | --- | --- |
| Unit/property checks | Decimal arithmetic, decision boundaries, state transitions and allocation invariants | Reconcile signed statement balance = opening balance + invoices - credits - payments. A 0.01 currency-unit residual is permitted for rounding; higher residuals create a case. Exact duplicates are held; fuzzy similarity ≥0.90 only raises a review flag. |
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
| AE-BR-01/02 | AE-FR-1 / US-1 | POST /api/v1/exception-cases | AE-SIT-01 | statement.statement_number='STMT-2027-02'; statement.closing_balance=12500.00 |
| AE-BR-01/02 | AE-FR-2 / US-2 | POST /api/v1/exception-cases/{id}/reconcile | AE-SIT-02 | statement_line.external_bill_id IS NOT NULL; exception_case.residual=25.00 |
| AE-BR-01/02 | AE-FR-3 / US-3 | POST /api/v1/exception-cases/{id}/duplicate-check | AE-SIT-03 | risk_flag.flag_type='EXACT_DUPLICATE'; exception_case.status='HELD' |
| AE-BR-01/02 | AE-FR-4 / US-4 | POST /api/v1/exception-cases/{id}/risk-review | AE-SIT-04 | risk_flag.severity='HIGH'; external_write count=0 |
| AE-BR-01/02 | AE-FR-5 / US-5 | PATCH /api/v1/exception-cases/{id}/resolution | AE-SIT-05 | exception_case.residual=0.00; resolution_evidence count=1 |
| AE-BR-01/02 | AE-FR-6 / US-6 | POST /api/v1/payment-packs | AE-SIT-06 | payment_pack_line count=1; held case has no pack line |
| AE-BR-02/04 | AE-FR-7 / US-7 | POST /api/v1/payment-packs/{id}/approve | AE-SIT-07 | payment_pack.status='PREPARED'; approval count=0 |
| AE-BR-03 | AE-FR-8 / US-8 | GET /api/v1/payment-packs/{id}/download | AE-SIT-08 | payment_pack.artifact_sha256 unchanged; external_write count=0 |

### SIT execution instructions

| Case | Steps | Pass condition / evidence |
| --- | --- | --- |
| AE-SIT-01 | 1. Seed: a February statement and vendor V-77 are selected. 2. Invoke the mapped endpoint: I upload a statement totaling 12500.00 USD. 3. Wait for its terminal or deliberate review state. 4. Inspect SQL rows, operation history and output. | a statement hash and immutable opening-balance snapshot are recorded. Persist query output proving statement.statement_number='STMT-2027-02'; statement.closing_balance=12500.00. |
| AE-SIT-02 | 1. Seed: the statement has INV-1042 and the ledger contains its exact vendor and amount. 2. Invoke the mapped endpoint: reconciliation runs. 3. Wait for its terminal or deliberate review state. 4. Inspect SQL rows, operation history and output. | the line links to the external bill and a 25.00 unexplained residual opens a case. Persist query output proving statement_line.external_bill_id IS NOT NULL; exception_case.residual=25.00. |
| AE-SIT-03 | 1. Seed: two ledger records share vendor, normalized invoice number, currency and amount. 2. Invoke the mapped endpoint: the duplicate rule evaluates them. 3. Wait for its terminal or deliberate review state. 4. Inspect SQL rows, operation history and output. | one duplicate group is created and both candidates are excluded pending review. Persist query output proving risk_flag.flag_type='EXACT_DUPLICATE'; exception_case.status='HELD'. |
| AE-SIT-04 | 1. Seed: a statement text suggests new bank details. 2. Invoke the mapped endpoint: the classifier inspects the document. 3. Wait for its terminal or deliberate review state. 4. Inspect SQL rows, operation history and output. | a high-risk flag appears with source evidence and no account master is changed. Persist query output proving risk_flag.severity='HIGH'; external_write count=0. |
| AE-SIT-05 | 1. Seed: a 25.00 discrepancy has a valid credit note. 2. Invoke the mapped endpoint: I attach its accounting reference and resubmit. 3. Wait for its terminal or deliberate review state. 4. Inspect SQL rows, operation history and output. | the residual becomes zero and the case enters READY_FOR_APPROVAL. Persist query output proving exception_case.residual=0.00; resolution_evidence count=1. |
| AE-SIT-06 | 1. Seed: one cleared and one held case exist. 2. Invoke the mapped endpoint: I generate a payment preparation pack. 3. Wait for its terminal or deliberate review state. 4. Inspect SQL rows, operation history and output. | only the cleared case contributes an amount and the held case is listed as excluded. Persist query output proving payment_pack_line count=1; held case has no pack line. |
| AE-SIT-07 | 1. Seed: the pack preparer is the requesting user. 2. Invoke the mapped endpoint: that same user attempts approval. 3. Wait for its terminal or deliberate review state. 4. Inspect SQL rows, operation history and output. | 403 SOD_VIOLATION is returned and the pack remains PREPARED. Persist query output proving payment_pack.status='PREPARED'; approval count=0. |
| AE-SIT-08 | 1. Seed: an approved pack has content hash H1. 2. Invoke the mapped endpoint: the download request is repeated. 3. Wait for its terminal or deliberate review state. 4. Inspect SQL rows, operation history and output. | the same H1 artifact is returned and no payment transaction is initiated. Persist query output proving payment_pack.artifact_sha256 unchanged; external_write count=0. |

| Control case | Failure injection / action | Expected result |
| --- | --- | --- |
| AE-SIT-09 | Repeat same action with same idempotency key, then change payload under that key | Same payload returns original result; changed payload returns 409 IDEMPOTENCY_CONFLICT; no second side effect. |
| AE-SIT-10 | Read/mutate Tenant-A record with Tenant-B session and a copied signed object URL | Record access returns 404; URL issuance denied; object route cannot expose another tenant object; no domain change. |
| AE-SIT-11 | Crash worker after target action but before local success persistence | Operation enters reconciliation; target is read by stable reference; ledger records one confirmed action. |
| AE-SIT-12 | Replay event and reorder source versions 4, 3, 5 | Consumer receipt prevents duplicate event execution; version 3 cannot overwrite 4; version 5 invalidates stale approvals. |
| AE-SIT-13 | Return malformed agent JSON, wrong units or invalid arithmetic | One schema repair attempt; unresolved values route to review; no invented missing business evidence. |
| AE-SIT-14 | Return 429 with Retry-After: 60 and revoke OAuth refresh | Affected connection waits ≥60 sec; revoked connection pauses with RECONNECT_REQUIRED; other tenants continue. |
| AE-SIT-15 | Run the declared 60-minute capacity profile with representative evidence sizes | Phase 2 API/agent/queue percentile and error targets pass; include failures/timeouts in the sample. |
| AE-SIT-16 | Restore a time-selected encrypted backup into isolated staging | RPO ≤15 min, RTO ≤4h; restored row counts, tenant policies, job/outbox lineage and evidence access reconcile. |

## [PM] Structured UAT plan

**Participants:** one daily operator (Controller), one accountable process owner (AP specialist), one independent controller/approver and BSA as observer. PM owns acceptance decisions. Recruit two design-partner organizations and record each participant's role and consent. Run in staging/sandbox first; a consented production pilot uses parallel review and the same control boundaries.

**Entry criteria:** all Must stories demonstrated; SIT-01 through SIT-16 pass; no open severity-1/2 defect; pilot data consent and DPA confirmed; reviewers can access the source evidence; recovery path demonstrated. Budget two 90-minute sessions per organization. Each scenario gets two attempts after a defect fix; silently coaching through failures does not count as success.

| UAT ID / goal | Actor and setup | Step-by-step business exercise | Acceptance criteria |
| --- | --- | --- | --- |
| AE-UAT-1: Statement intake | AP specialist; a February statement and vendor V-77 are selected | 1. Sign in with the intended role. 2. Open the source evidence and record baseline/version. 3. I upload a statement totaling 12500.00 USD. 4. Inspect the result and decision reasons. 5. Open audit/output evidence and compare source values. 6. Record pass/fail and active handling time. | a statement hash and immutable opening-balance snapshot are recorded. statement.statement_number='STMT-2027-02'; statement.closing_balance=12500.00. No unexpected cross-tenant data or unapproved side effect. |
| AE-UAT-2: Duplicate hold | controller; two ledger records share vendor, normalized invoice number, currency and amount | 1. Sign in with the intended role. 2. Open the source evidence and record baseline/version. 3. The duplicate rule evaluates them. 4. Inspect the result and decision reasons. 5. Open audit/output evidence and compare source values. 6. Record pass/fail and active handling time. | one duplicate group is created and both candidates are excluded pending review. risk_flag.flag_type='EXACT_DUPLICATE'; exception_case.status='HELD'. No unexpected cross-tenant data or unapproved side effect. |
| AE-UAT-3: Evidence resolution | AP specialist; a 25.00 discrepancy has a valid credit note | 1. Sign in with the intended role. 2. Open the source evidence and record baseline/version. 3. I attach its accounting reference and resubmit. 4. Inspect the result and decision reasons. 5. Open audit/output evidence and compare source values. 6. Record pass/fail and active handling time. | the residual becomes zero and the case enters READY_FOR_APPROVAL. exception_case.residual=0.00; resolution_evidence count=1. No unexpected cross-tenant data or unapproved side effect. |
| AE-UAT-4: Pack preparation | AP specialist; one cleared and one held case exist | 1. Sign in with the intended role. 2. Open the source evidence and record baseline/version. 3. I generate a payment preparation pack. 4. Inspect the result and decision reasons. 5. Open audit/output evidence and compare source values. 6. Record pass/fail and active handling time. | only the cleared case contributes an amount and the held case is listed as excluded. payment_pack_line count=1; held case has no pack line. No unexpected cross-tenant data or unapproved side effect. |
| AE-UAT-5: Pack delivery recovery | controller; an approved pack has content hash H1 | 1. Sign in with the intended role. 2. Open the source evidence and record baseline/version. 3. The download request is repeated. 4. Inspect the result and decision reasons. 5. Open audit/output evidence and compare source values. 6. Record pass/fail and active handling time. | the same H1 artifact is returned and no payment transaction is initiated. payment_pack.artifact_sha256 unchanged; external_write count=0. No unexpected cross-tenant data or unapproved side effect. |

**Product outcome gate:** Statement line linkage precision ≥95% on labeled data; ≥50% lower median investigation time; zero held items in an approved pack. **Control gate:** Fraud outputs are review signals, never accusations. Packs contain bill references and amounts; no bank execution endpoint or bank account changes. **Measured usability gate:** at least 4/5 scenarios completed without coaching by each daily operator; no critical accessibility barrier; all financial/identity controls pass for every participant. A high usability rate cannot compensate for a control failure.

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
