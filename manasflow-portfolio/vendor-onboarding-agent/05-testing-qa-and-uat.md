# Vendor Onboarding Agent — Testing, QA & UAT

[Portfolio index](../README.md) · [Phase 1](01-ideation-and-strategy.md) · [Phase 2](02-roadmapping-and-scoping.md) · [Phase 3](03-requirements-prd-frd.md) · [Phase 4](04-sprint-execution-and-triage.md) · [Phase 5](05-testing-qa-and-uat.md) · [Phase 6](06-launch-gtm-and-telemetry.md)

**Document control:** VO-P5, revision 1.0, authored 2026-10-17. Status: proposed portfolio design; implementation, customer research, tests and stakeholder approvals have not been performed.

**Source boundary:** Named parallel upsell, source p. 14. The source provides the name only; structured evidence collection and controlled vendor-master synchronization are proposed scope. [Primary document](../../MatchClerk-Project-Implementation.pdf). Proposed architecture and assumptions are explicitly recorded in [the evidence register](../source-and-assumptions.md).

## [BSA] Test strategy and controlled fixtures

This is an executable test **plan** and expected-state specification; no application exists in this workspace, so no SIT, security, load or UAT result is claimed. Test evidence must contain build identifier, schema migration version, synthetic fixture revision, run timestamp, observed result and reviewer. Planned acceptance gates differ from documentation validation performed on these Markdown artifacts.

| Layer | Scope | Method / expected result |
| --- | --- | --- |
| Unit/property checks | Decimal arithmetic, decision boundaries, state transitions and allocation invariants | Exact vendor matches use tenant, external organization and immutable external ID. Normalized legal name plus postal region only creates a candidate; it never auto-merges suppliers. A changed identity/contact payload invalidates prior approval and requires independent review. |
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
| VO-BR-01/02 | VO-FR-1 / US-1 | POST /api/v1/vendor-applications | VO-SIT-01 | vendor_application.application_reference='VEND-2027-041'; external_write count=0 |
| VO-BR-01/02 | VO-FR-2 / US-2 | POST /api/v1/vendor-applications/{id}/documents | VO-SIT-02 | submitted_document.scan_status='CLEAN'; submitted_document.sha256 IS NOT NULL |
| VO-BR-01/02 | VO-FR-3 / US-3 | POST /api/v1/vendor-applications/{id}/checks | VO-SIT-03 | verification_check.check_code='CONTACT_ADDRESS'; verification_check.passed=false |
| VO-BR-01/02 | VO-FR-4 / US-4 | POST /api/v1/vendor-applications/{id}/duplicate-check | VO-SIT-04 | duplicate_candidate.external_vendor_id='V-77'; external_write count=0 |
| VO-BR-01/02 | VO-FR-5 / US-5 | POST /api/v1/vendor-applications/{id}/approve | VO-SIT-05 | approval count=1; approval.approved_version=4 |
| VO-BR-01/02 | VO-FR-6 / US-6 | POST /api/v1/vendor-applications/{id}/sync | VO-SIT-06 | vendor_mapping.external_vendor_id IS NOT NULL; sync_request.status='SUCCEEDED' |
| VO-BR-02/04 | VO-FR-7 / US-7 | POST /api/v1/vendor-applications/{id}/publish | VO-SIT-07 | outbox_event.event_type='vendor.verified'; consumption_receipt count=1 |
| VO-BR-03 | VO-FR-8 / US-8 | PATCH /api/v1/vendor-applications/{id} | VO-SIT-08 | vendor_application.record_version=5; sync_request.status='BLOCKED' |

### SIT execution instructions

| Case | Steps | Pass condition / evidence |
| --- | --- | --- |
| VO-SIT-01 | 1. Seed: supplier Acme Supply Ltd has consented to registration. 2. Invoke the mapped endpoint: I enter legal name, address and contact details. 3. Wait for its terminal or deliberate review state. 4. Inspect SQL rows, operation history and output. | one application with reference VEND-2027-041 is created without an external vendor write. Persist query output proving vendor_application.application_reference='VEND-2027-041'; external_write count=0. |
| VO-SIT-02 | 1. Seed: the application requires registration evidence and an authorized invitation is active. 2. Invoke the mapped endpoint: I upload a clean PDF. 3. Wait for its terminal or deliberate review state. 4. Inspect SQL rows, operation history and output. | the document is scanned, encrypted and linked to the application with upload provenance. Persist query output proving submitted_document.scan_status='CLEAN'; submitted_document.sha256 IS NOT NULL. |
| VO-SIT-03 | 1. Seed: legal name is present but the contact address is missing. 2. Invoke the mapped endpoint: I run the verification checklist. 3. Wait for its terminal or deliberate review state. 4. Inspect SQL rows, operation history and output. | the application remains NEEDS_REVIEW with CONTACT_ADDRESS_MISSING and cannot be approved. Persist query output proving verification_check.check_code='CONTACT_ADDRESS'; verification_check.passed=false. |
| VO-SIT-04 | 1. Seed: Acme Supply Ltd resembles existing vendor V-77. 2. Invoke the mapped endpoint: the duplicate search runs. 3. Wait for its terminal or deliberate review state. 4. Inspect SQL rows, operation history and output. | a candidate appears with evidence and no automatic merge or new vendor write occurs. Persist query output proving duplicate_candidate.external_vendor_id='V-77'; external_write count=0. |
| VO-SIT-05 | 1. Seed: all evidence is verified and the preparer differs from the controller. 2. Invoke the mapped endpoint: the controller approves current version 4. 3. Wait for its terminal or deliberate review state. 4. Inspect SQL rows, operation history and output. | one approval is bound to the canonical supplier payload hash. Persist query output proving approval count=1; approval.approved_version=4. |
| VO-SIT-06 | 1. Seed: the approved application has no bank fields and no unresolved duplicate. 2. Invoke the mapped endpoint: the connector submits the authorized vendor/contact payload. 3. Wait for its terminal or deliberate review state. 4. Inspect SQL rows, operation history and output. | one external vendor ID is stored and no financial transaction is created. Persist query output proving vendor_mapping.external_vendor_id IS NOT NULL; sync_request.status='SUCCEEDED'. |
| VO-SIT-07 | 1. Seed: vendor sync succeeded for V-77. 2. Invoke the mapped endpoint: the completion event is consumed by Full AP. 3. Wait for its terminal or deliberate review state. 4. Inspect SQL rows, operation history and output. | one component task is marked verified with source application revision. Persist query output proving outbox_event.event_type='vendor.verified'; consumption_receipt count=1. |
| VO-SIT-08 | 1. Seed: a registration was approved at version 4. 2. Invoke the mapped endpoint: a user changes the legal name before synchronization. 3. Wait for its terminal or deliberate review state. 4. Inspect SQL rows, operation history and output. | the approval becomes stale and the new version returns to NEEDS_REVIEW. Persist query output proving vendor_application.record_version=5; sync_request.status='BLOCKED'. |

| Control case | Failure injection / action | Expected result |
| --- | --- | --- |
| VO-SIT-09 | Repeat same action with same idempotency key, then change payload under that key | Same payload returns original result; changed payload returns 409 IDEMPOTENCY_CONFLICT; no second side effect. |
| VO-SIT-10 | Read/mutate Tenant-A record with Tenant-B session and a copied signed object URL | Record access returns 404; URL issuance denied; object route cannot expose another tenant object; no domain change. |
| VO-SIT-11 | Crash worker after target action but before local success persistence | Operation enters reconciliation; target is read by stable reference; ledger records one confirmed action. |
| VO-SIT-12 | Replay event and reorder source versions 4, 3, 5 | Consumer receipt prevents duplicate event execution; version 3 cannot overwrite 4; version 5 invalidates stale approvals. |
| VO-SIT-13 | Return malformed agent JSON, wrong units or invalid arithmetic | One schema repair attempt; unresolved values route to review; no invented missing business evidence. |
| VO-SIT-14 | Return 429 with Retry-After: 60 and revoke OAuth refresh | Affected connection waits ≥60 sec; revoked connection pauses with RECONNECT_REQUIRED; other tenants continue. |
| VO-SIT-15 | Run the declared 60-minute capacity profile with representative evidence sizes | Phase 2 API/agent/queue percentile and error targets pass; include failures/timeouts in the sample. |
| VO-SIT-16 | Restore a time-selected encrypted backup into isolated staging | RPO ≤15 min, RTO ≤4h; restored row counts, tenant policies, job/outbox lineage and evidence access reconcile. |

## [PM] Structured UAT plan

**Participants:** one daily operator (AP intake specialist), one accountable process owner (Controller), one independent controller/approver and BSA as observer. PM owns acceptance decisions. Recruit two design-partner organizations and record each participant's role and consent. Run in staging/sandbox first; a consented production pilot uses parallel review and the same control boundaries.

**Entry criteria:** all Must stories demonstrated; SIT-01 through SIT-16 pass; no open severity-1/2 defect; pilot data consent and DPA confirmed; reviewers can access the source evidence; recovery path demonstrated. Budget two 90-minute sessions per organization. Each scenario gets two attempts after a defect fix; silently coaching through failures does not count as success.

| UAT ID / goal | Actor and setup | Step-by-step business exercise | Acceptance criteria |
| --- | --- | --- | --- |
| VO-UAT-1: Registration intake | AP intake specialist; supplier Acme Supply Ltd has consented to registration | 1. Sign in with the intended role. 2. Open the source evidence and record baseline/version. 3. I enter legal name, address and contact details. 4. Inspect the result and decision reasons. 5. Open audit/output evidence and compare source values. 6. Record pass/fail and active handling time. | one application with reference VEND-2027-041 is created without an external vendor write. vendor_application.application_reference='VEND-2027-041'; external_write count=0. No unexpected cross-tenant data or unapproved side effect. |
| VO-UAT-2: Checklist validation | controller; legal name is present but the contact address is missing | 1. Sign in with the intended role. 2. Open the source evidence and record baseline/version. 3. I run the verification checklist. 4. Inspect the result and decision reasons. 5. Open audit/output evidence and compare source values. 6. Record pass/fail and active handling time. | the application remains NEEDS_REVIEW with CONTACT_ADDRESS_MISSING and cannot be approved. verification_check.check_code='CONTACT_ADDRESS'; verification_check.passed=false. No unexpected cross-tenant data or unapproved side effect. |
| VO-UAT-3: Independent approval | controller; all evidence is verified and the preparer differs from the controller | 1. Sign in with the intended role. 2. Open the source evidence and record baseline/version. 3. The controller approves current version 4. 4. Inspect the result and decision reasons. 5. Open audit/output evidence and compare source values. 6. Record pass/fail and active handling time. | one approval is bound to the canonical supplier payload hash. approval count=1; approval.approved_version=4. No unexpected cross-tenant data or unapproved side effect. |
| VO-UAT-4: Vendor sync | controller; the approved application has no bank fields and no unresolved duplicate | 1. Sign in with the intended role. 2. Open the source evidence and record baseline/version. 3. The connector submits the authorized vendor/contact payload. 4. Inspect the result and decision reasons. 5. Open audit/output evidence and compare source values. 6. Record pass/fail and active handling time. | one external vendor ID is stored and no financial transaction is created. vendor_mapping.external_vendor_id IS NOT NULL; sync_request.status='SUCCEEDED'. No unexpected cross-tenant data or unapproved side effect. |
| VO-UAT-5: Identity revision recovery | controller; a registration was approved at version 4 | 1. Sign in with the intended role. 2. Open the source evidence and record baseline/version. 3. A user changes the legal name before synchronization. 4. Inspect the result and decision reasons. 5. Open audit/output evidence and compare source values. 6. Record pass/fail and active handling time. | the approval becomes stale and the new version returns to NEEDS_REVIEW. vendor_application.record_version=5; sync_request.status='BLOCKED'. No unexpected cross-tenant data or unapproved side effect. |

**Product outcome gate:** Required-field completeness 100% before approval; duplicate review precision ≥95% on labeled cases; zero bank-field writes; ≥85% two-business-day completion. **Control gate:** No bank account collection or update. A human verifies supplier evidence and approves the exact vendor-master payload; the agent cannot certify legal or tax compliance. **Measured usability gate:** at least 4/5 scenarios completed without coaching by each daily operator; no critical accessibility barrier; all financial/identity controls pass for every participant. A high usability rate cannot compensate for a control failure.

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
