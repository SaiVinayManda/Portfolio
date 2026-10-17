# MatchClerk — Testing, QA & UAT

[Portfolio index](../README.md) · [Phase 1](01-ideation-and-strategy.md) · [Phase 2](02-roadmapping-and-scoping.md) · [Phase 3](03-requirements-prd-frd.md) · [Phase 4](04-sprint-execution-and-triage.md) · [Phase 5](05-testing-qa-and-uat.md) · [Phase 6](06-launch-gtm-and-telemetry.md)

**Document control:** MC-P5, revision 1.0, authored 2026-10-17. Status: proposed portfolio design; implementation, customer research, tests and stakeholder approvals have not been performed.

**Source boundary:** Fully specified product; source pp. 1–19 and 21–22, F1–F18, NF1–NF8. [Primary document](../../MatchClerk-Project-Implementation.pdf). Proposed architecture and assumptions are explicitly recorded in [the evidence register](../source-and-assumptions.md).

## [BSA] Test strategy and controlled fixtures

This is an executable test **plan** and expected-state specification; no application exists in this workspace, so no SIT, security, load or UAT result is claimed. Test evidence must contain build identifier, schema migration version, synthetic fixture revision, run timestamp, observed result and reviewer. Planned acceptance gates differ from documentation validation performed on these Markdown artifacts.

| Layer | Scope | Method / expected result |
| --- | --- | --- |
| Unit/property checks | Decimal arithmetic, decision boundaries, state transitions and allocation invariants | Default allowed price variance is max(1% of comparable PO line net amount, 2.00 invoice-currency units), or min when REQUIRE_BOTH is enabled. Quantity tolerance is zero. Compare tax separately; do not allow an aggregate total to hide a line overcharge. |
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
| MC-BR-01/02 | MC-FR-1 / US-1 | POST /api/v1/invoices/upload | MC-SIT-01 | invoice.status='RECEIVED'; processing_job count=1 |
| MC-BR-01/02 | MC-FR-2 / US-2 | POST /api/v1/invoices/{id}/extract | MC-SIT-02 | invoice.total=1020.00; invoice_line.quantity=10.0000 |
| MC-BR-01/02 | MC-FR-3 / US-3 | POST /api/v1/invoices/{id}/match | MC-SIT-03 | invoice.status='AWAITING_RECEIPT'; external_write count=0 |
| MC-BR-01/02 | MC-FR-4 / US-4 | PATCH /api/v1/settings/matching | MC-SIT-04 | match_result.price_delta=12.00; invoice.status='NEEDS_REVIEW' |
| MC-BR-01/02 | MC-FR-5 / US-5 | POST /api/v1/invoices/{id}/approve | MC-SIT-05 | approval count=1; external_write count=1; external_bill_id IS NOT NULL after success |
| MC-BR-01/02 | MC-FR-6 / US-6 | POST /api/v1/invoices/{id}/approve | MC-SIT-06 | external_write.provider='XERO'; invoice.external_bill_id IS NOT NULL |
| MC-BR-02/04 | MC-FR-7 / US-7 | GET /api/v1/invoices/{id}/audit | MC-SIT-07 | audit_event.action='FIELD_CORRECTED'; record_version advanced once |
| MC-BR-03 | MC-FR-8 / US-8 | POST /api/v1/operations/{operation_id}/reconcile | MC-SIT-08 | external_write.status='SUCCEEDED'; unique external reference count=1 |

### SIT execution instructions

| Case | Steps | Pass condition / evidence |
| --- | --- | --- |
| MC-SIT-01 | 1. Seed: a signed-in clerk and a clean 2-page PDF of 2 MB. 2. Invoke the mapped endpoint: the upload completes. 3. Wait for its terminal or deliberate review state. 4. Inspect SQL rows, operation history and output. | the invoice appears as RECEIVED within 5 seconds and one processing job exists. Persist query output proving invoice.status='RECEIVED'; processing_job count=1. |
| MC-SIT-02 | 1. Seed: INV-1042 has ten units at 100.00 and tax 20.00. 2. Invoke the mapped endpoint: the extractor returns total 1020.00 with confidence 0.91. 3. Wait for its terminal or deliberate review state. 4. Inspect SQL rows, operation history and output. | ten units, subtotal 1000.00 and tax 20.00 are persisted as decimal strings and source-page provenance is visible. Persist query output proving invoice.total=1020.00; invoice_line.quantity=10.0000. |
| MC-SIT-03 | 1. Seed: PO-501 has ten units ordered but eight attested as received. 2. Invoke the mapped endpoint: ten units are invoiced in THREE_WAY mode. 3. Wait for its terminal or deliberate review state. 4. Inspect SQL rows, operation history and output. | the case becomes AWAITING_RECEIPT and no accounting write is queued. Persist query output proving invoice.status='AWAITING_RECEIPT'; external_write count=0. |
| MC-SIT-04 | 1. Seed: PO net line is 1000.00 and policy is 1 percent or 2.00 maximum allowance. 2. Invoke the mapped endpoint: an invoice line is 1012.00. 3. Wait for its terminal or deliberate review state. 4. Inspect SQL rows, operation history and output. | the 12.00 delta exceeds the 10.00 allowance and NEEDS_REVIEW records the calculation. Persist query output proving match_result.price_delta=12.00; invoice.status='NEEDS_REVIEW'. |
| MC-SIT-05 | 1. Seed: all mandatory evidence is valid and invoice version is 4. 2. Invoke the mapped endpoint: two approve requests carry the same idempotency key and expected_version 4. 3. Wait for its terminal or deliberate review state. 4. Inspect SQL rows, operation history and output. | one approval and one QBO Bill write occur and both clients receive the same operation ID. Persist query output proving approval count=1; external_write count=1; external_bill_id IS NOT NULL after success. |
| MC-SIT-06 | 1. Seed: an approved invoice has valid ContactID and AccountCode mappings. 2. Invoke the mapped endpoint: the Xero adapter submits it. 3. Wait for its terminal or deliberate review state. 4. Inspect SQL rows, operation history and output. | the payload has Type ACCPAY and Status DRAFT and the returned InvoiceID is recorded. Persist query output proving external_write.provider='XERO'; invoice.external_bill_id IS NOT NULL. |
| MC-SIT-07 | 1. Seed: a clerk corrected GL code 5000 to 5100. 2. Invoke the mapped endpoint: an auditor opens the audit view. 3. Wait for its terminal or deliberate review state. 4. Inspect SQL rows, operation history and output. | before and after values and actor are visible while edit and approve controls remain forbidden. Persist query output proving audit_event.action='FIELD_CORRECTED'; record_version advanced once. |
| MC-SIT-08 | 1. Seed: a provider created a bill but its response was lost. 2. Invoke the mapped endpoint: the connector worker retries after reconciling its stable reference. 3. Wait for its terminal or deliberate review state. 4. Inspect SQL rows, operation history and output. | it records the existing bill and does not issue a second blind create. Persist query output proving external_write.status='SUCCEEDED'; unique external reference count=1. |

| Control case | Failure injection / action | Expected result |
| --- | --- | --- |
| MC-SIT-09 | Repeat same action with same idempotency key, then change payload under that key | Same payload returns original result; changed payload returns 409 IDEMPOTENCY_CONFLICT; no second side effect. |
| MC-SIT-10 | Read/mutate Tenant-A record with Tenant-B session and a copied signed object URL | Record access returns 404; URL issuance denied; object route cannot expose another tenant object; no domain change. |
| MC-SIT-11 | Crash worker after target action but before local success persistence | Operation enters reconciliation; target is read by stable reference; ledger records one confirmed action. |
| MC-SIT-12 | Replay event and reorder source versions 4, 3, 5 | Consumer receipt prevents duplicate event execution; version 3 cannot overwrite 4; version 5 invalidates stale approvals. |
| MC-SIT-13 | Return malformed agent JSON, wrong units or invalid arithmetic | One schema repair attempt; unresolved values route to review; no invented missing business evidence. |
| MC-SIT-14 | Return 429 with Retry-After: 60 and revoke OAuth refresh | Affected connection waits ≥60 sec; revoked connection pauses with RECONNECT_REQUIRED; other tenants continue. |
| MC-SIT-15 | Run the declared 60-minute capacity profile with representative evidence sizes | Phase 2 API/agent/queue percentile and error targets pass; include failures/timeouts in the sample. |
| MC-SIT-16 | Restore a time-selected encrypted backup into isolated staging | RPO ≤15 min, RTO ≤4h; restored row counts, tenant policies, job/outbox lineage and evidence access reconcile. |

## [PM] Structured UAT plan

**Participants:** one daily operator (AP clerk), one accountable process owner (Controller), one independent controller/approver and BSA as observer. PM owns acceptance decisions. Recruit two design-partner organizations and record each participant's role and consent. Run in staging/sandbox first; a consented production pilot uses parallel review and the same control boundaries.

**Entry criteria:** all Must stories demonstrated; SIT-01 through SIT-16 pass; no open severity-1/2 defect; pilot data consent and DPA confirmed; reviewers can access the source evidence; recovery path demonstrated. Budget two 90-minute sessions per organization. Each scenario gets two attempts after a defect fix; silently coaching through failures does not count as success.

| UAT ID / goal | Actor and setup | Step-by-step business exercise | Acceptance criteria |
| --- | --- | --- | --- |
| MC-UAT-1: Secure intake | AP clerk; a signed-in clerk and a clean 2-page PDF of 2 MB | 1. Sign in with the intended role. 2. Open the source evidence and record baseline/version. 3. The upload completes. 4. Inspect the result and decision reasons. 5. Open audit/output evidence and compare source values. 6. Record pass/fail and active handling time. | the invoice appears as RECEIVED within 5 seconds and one processing job exists. invoice.status='RECEIVED'; processing_job count=1. No unexpected cross-tenant data or unapproved side effect. |
| MC-UAT-2: Three-way match | operations manager; PO-501 has ten units ordered but eight attested as received | 1. Sign in with the intended role. 2. Open the source evidence and record baseline/version. 3. Ten units are invoiced in THREE_WAY mode. 4. Inspect the result and decision reasons. 5. Open audit/output evidence and compare source values. 6. Record pass/fail and active handling time. | the case becomes AWAITING_RECEIPT and no accounting write is queued. invoice.status='AWAITING_RECEIPT'; external_write count=0. No unexpected cross-tenant data or unapproved side effect. |
| MC-UAT-3: Approval and QBO write | controller; all mandatory evidence is valid and invoice version is 4 | 1. Sign in with the intended role. 2. Open the source evidence and record baseline/version. 3. Two approve requests carry the same idempotency key and expected_version 4. 4. Inspect the result and decision reasons. 5. Open audit/output evidence and compare source values. 6. Record pass/fail and active handling time. | one approval and one QBO Bill write occur and both clients receive the same operation ID. approval count=1; external_write count=1; external_bill_id IS NOT NULL after success. No unexpected cross-tenant data or unapproved side effect. |
| MC-UAT-4: Xero draft | controller; an approved invoice has valid ContactID and AccountCode mappings | 1. Sign in with the intended role. 2. Open the source evidence and record baseline/version. 3. The Xero adapter submits it. 4. Inspect the result and decision reasons. 5. Open audit/output evidence and compare source values. 6. Record pass/fail and active handling time. | the payload has Type ACCPAY and Status DRAFT and the returned InvoiceID is recorded. external_write.provider='XERO'; invoice.external_bill_id IS NOT NULL. No unexpected cross-tenant data or unapproved side effect. |
| MC-UAT-5: Recovery and duplicate control | controller; a provider created a bill but its response was lost | 1. Sign in with the intended role. 2. Open the source evidence and record baseline/version. 3. The connector worker retries after reconciling its stable reference. 4. Inspect the result and decision reasons. 5. Open audit/output evidence and compare source values. 6. Record pass/fail and active handling time. | it records the existing bill and does not issue a second blind create. external_write.status='SUCCEEDED'; unique external reference count=1. No unexpected cross-tenant data or unapproved side effect. |

**Product outcome gate:** Extraction critical-header exact match ≥85%; line-amount F1 ≥80%; operational line-field accuracy ≥85%; auto-match ≥70%; five paying logos and controller NPS ≥40 by day 90. **Control gate:** No accounting write before controller approval. No payment execution. QBO drafts live internally; Xero DRAFT creation also waits for approval in this design. **Measured usability gate:** at least 4/5 scenarios completed without coaching by each daily operator; no critical accessibility barrier; all financial/identity controls pass for every participant. A high usability rate cannot compensate for a control failure.

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
