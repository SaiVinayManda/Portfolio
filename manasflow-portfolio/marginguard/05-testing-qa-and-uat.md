# MarginGuard — Testing, QA & UAT

[Portfolio index](../README.md) · [Phase 1](01-ideation-and-strategy.md) · [Phase 2](02-roadmapping-and-scoping.md) · [Phase 3](03-requirements-prd-frd.md) · [Phase 4](04-sprint-execution-and-triage.md) · [Phase 5](05-testing-qa-and-uat.md) · [Phase 6](06-launch-gtm-and-telemetry.md)

**Document control:** MG-P5, revision 1.0, authored 2026-10-17. Status: proposed portfolio design; implementation, customer research, tests and stakeholder approvals have not been performed.

**Source boundary:** Named attachment, source p. 20: catalog price watch. Supplier-cost and margin-impact details below are proposed interpretations, not source claims. [Primary document](../../MatchClerk-Project-Implementation.pdf). Proposed architecture and assumptions are explicitly recorded in [the evidence register](../source-and-assumptions.md).

## [BSA] Test strategy and controlled fixtures

This is an executable test **plan** and expected-state specification; no application exists in this workspace, so no SIT, security, load or UAT result is claimed. Test evidence must contain build identifier, schema migration version, synthetic fixture revision, run timestamp, observed result and reviewer. Planned acceptance gates differ from documentation validation performed on these Markdown artifacts.

| Layer | Scope | Method / expected result |
| --- | --- | --- |
| Unit/property checks | Decimal arithmetic, decision boundaries, state transitions and allocation invariants | Gross margin = (net sell price - comparable unit cost) / net sell price. A cost increase from 100.00 to 104.00 at sell price 140.00 moves margin from 28.57% to 25.71%. Alert ≥3% cost change; hard review if resulting margin <25%. Never equate margin percentage with markup. |
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
| MG-BR-01/02 | MG-FR-1 / US-1 | POST /api/v1/price-changes | MG-SIT-01 | catalog_version.effective_from='2027-10-15'; catalog_version.sha256 IS NOT NULL |
| MG-BR-01/02 | MG-FR-2 / US-2 | POST /api/v1/price-changes/{id}/normalize | MG-SIT-02 | catalog_line.unit_cost=104.0000; sku.uom='EA' |
| MG-BR-01/02 | MG-FR-3 / US-3 | POST /api/v1/price-changes/{id}/evaluate | MG-SIT-03 | price_change.delta_pct=4.0000; price_change.status='NEEDS_REVIEW' |
| MG-BR-01/02 | MG-FR-4 / US-4 | GET /api/v1/price-changes/{id}/margin-impact | MG-SIT-04 | margin_impact.old_margin=0.285714; margin_impact.new_margin=0.257143 |
| MG-BR-01/02 | MG-FR-5 / US-5 | PATCH /api/v1/price-changes/{id}/evidence | MG-SIT-05 | margin_impact.new_margin IS NULL; price_change.status='NEEDS_REVIEW' |
| MG-BR-01/02 | MG-FR-6 / US-6 | POST /api/v1/price-changes/{id}/approve | MG-SIT-06 | price_proposal.proposed_sell_price=145.00; approval count=1 |
| MG-BR-02/04 | MG-FR-7 / US-7 | POST /api/v1/price-changes/{id}/publish | MG-SIT-07 | outbox_event.event_type='catalog.cost_changed'; consumption_receipt count=1 |
| MG-BR-03 | MG-FR-8 / US-8 | POST /api/v1/price-changes/reimport | MG-SIT-08 | UNIQUE(tenant_id,supplier_id,sha256); price_change count unchanged |

### SIT execution instructions

| Case | Steps | Pass condition / evidence |
| --- | --- | --- |
| MG-SIT-01 | 1. Seed: supplier V-77 provides a CSV effective 2027-10-15. 2. Invoke the mapped endpoint: I import the file. 3. Wait for its terminal or deliberate review state. 4. Inspect SQL rows, operation history and output. | a version and hash are stored and invalid rows are quarantined. Persist query output proving catalog_version.effective_from='2027-10-15'; catalog_version.sha256 IS NOT NULL. |
| MG-SIT-02 | 1. Seed: baseline is 100.00 per EA and new quote is 1248.00 per BOX of twelve. 2. Invoke the mapped endpoint: the mapped conversion is applied. 3. Wait for its terminal or deliberate review state. 4. Inspect SQL rows, operation history and output. | comparable new unit cost is 104.00 and original pack evidence remains visible. Persist query output proving catalog_line.unit_cost=104.0000; sku.uom='EA'. |
| MG-SIT-03 | 1. Seed: comparable baseline cost is 100.00 and new cost 104.00. 2. Invoke the mapped endpoint: the deterministic comparison runs. 3. Wait for its terminal or deliberate review state. 4. Inspect SQL rows, operation history and output. | the change is 4.00 percent and one material cost alert is opened. Persist query output proving price_change.delta_pct=4.0000; price_change.status='NEEDS_REVIEW'. |
| MG-SIT-04 | 1. Seed: net sell price is 140.00 and cost increases from 100.00 to 104.00. 2. Invoke the mapped endpoint: the impact is computed. 3. Wait for its terminal or deliberate review state. 4. Inspect SQL rows, operation history and output. | old margin is 28.5714 percent and new margin 25.7143 percent with unrounded calculation retained. Persist query output proving margin_impact.old_margin=0.285714; margin_impact.new_margin=0.257143. |
| MG-SIT-05 | 1. Seed: SKU STEEL-A has no current sell price. 2. Invoke the mapped endpoint: a new catalog line arrives. 3. Wait for its terminal or deliberate review state. 4. Inspect SQL rows, operation history and output. | cost delta remains visible but margin is null and export is blocked. Persist query output proving margin_impact.new_margin IS NULL; price_change.status='NEEDS_REVIEW'. |
| MG-SIT-06 | 1. Seed: a 145.00 proposed sell price has complete evidence. 2. Invoke the mapped endpoint: I approve current version 4. 3. Wait for its terminal or deliberate review state. 4. Inspect SQL rows, operation history and output. | the proposal is locked and one approved-export job is queued. Persist query output proving price_proposal.proposed_sell_price=145.00; approval count=1. |
| MG-SIT-07 | 1. Seed: an open procurement request uses STEEL-A at baseline 100.00. 2. Invoke the mapped endpoint: a verified 104.00 cost change is published. 3. Wait for its terminal or deliberate review state. 4. Inspect SQL rows, operation history and output. | the request receives a price-watch event and its buyer sees the evidence. Persist query output proving outbox_event.event_type='catalog.cost_changed'; consumption_receipt count=1. |
| MG-SIT-08 | 1. Seed: catalog hash H1 was already processed. 2. Invoke the mapped endpoint: the same file is uploaded again. 3. Wait for its terminal or deliberate review state. 4. Inspect SQL rows, operation history and output. | the original catalog result is returned with no second price-change alert. Persist query output proving UNIQUE(tenant_id,supplier_id,sha256); price_change count unchanged. |

| Control case | Failure injection / action | Expected result |
| --- | --- | --- |
| MG-SIT-09 | Repeat same action with same idempotency key, then change payload under that key | Same payload returns original result; changed payload returns 409 IDEMPOTENCY_CONFLICT; no second side effect. |
| MG-SIT-10 | Read/mutate Tenant-A record with Tenant-B session and a copied signed object URL | Record access returns 404; URL issuance denied; object route cannot expose another tenant object; no domain change. |
| MG-SIT-11 | Crash worker after target action but before local success persistence | Operation enters reconciliation; target is read by stable reference; ledger records one confirmed action. |
| MG-SIT-12 | Replay event and reorder source versions 4, 3, 5 | Consumer receipt prevents duplicate event execution; version 3 cannot overwrite 4; version 5 invalidates stale approvals. |
| MG-SIT-13 | Return malformed agent JSON, wrong units or invalid arithmetic | One schema repair attempt; unresolved values route to review; no invented missing business evidence. |
| MG-SIT-14 | Return 429 with Retry-After: 60 and revoke OAuth refresh | Affected connection waits ≥60 sec; revoked connection pauses with RECONNECT_REQUIRED; other tenants continue. |
| MG-SIT-15 | Run the declared 60-minute capacity profile with representative evidence sizes | Phase 2 API/agent/queue percentile and error targets pass; include failures/timeouts in the sample. |
| MG-SIT-16 | Restore a time-selected encrypted backup into isolated staging | RPO ≤15 min, RTO ≤4h; restored row counts, tenant policies, job/outbox lineage and evidence access reconcile. |

## [PM] Structured UAT plan

**Participants:** one daily operator (Pricing manager), one accountable process owner (Procurement manager), one independent controller/approver and BSA as observer. PM owns acceptance decisions. Recruit two design-partner organizations and record each participant's role and consent. Run in staging/sandbox first; a consented production pilot uses parallel review and the same control boundaries.

**Entry criteria:** all Must stories demonstrated; SIT-01 through SIT-16 pass; no open severity-1/2 defect; pilot data consent and DPA confirmed; reviewers can access the source evidence; recovery path demonstrated. Budget two 90-minute sessions per organization. Each scenario gets two attempts after a defect fix; silently coaching through failures does not count as success.

| UAT ID / goal | Actor and setup | Step-by-step business exercise | Acceptance criteria |
| --- | --- | --- | --- |
| MG-UAT-1: Catalog intake | pricing manager; supplier V-77 provides a CSV effective 2027-10-15 | 1. Sign in with the intended role. 2. Open the source evidence and record baseline/version. 3. I import the file. 4. Inspect the result and decision reasons. 5. Open audit/output evidence and compare source values. 6. Record pass/fail and active handling time. | a version and hash are stored and invalid rows are quarantined. catalog_version.effective_from='2027-10-15'; catalog_version.sha256 IS NOT NULL. No unexpected cross-tenant data or unapproved side effect. |
| MG-UAT-2: Cost delta | pricing manager; comparable baseline cost is 100.00 and new cost 104.00 | 1. Sign in with the intended role. 2. Open the source evidence and record baseline/version. 3. The deterministic comparison runs. 4. Inspect the result and decision reasons. 5. Open audit/output evidence and compare source values. 6. Record pass/fail and active handling time. | the change is 4.00 percent and one material cost alert is opened. price_change.delta_pct=4.0000; price_change.status='NEEDS_REVIEW'. No unexpected cross-tenant data or unapproved side effect. |
| MG-UAT-3: Incomplete evidence hold | pricing manager; SKU STEEL-A has no current sell price | 1. Sign in with the intended role. 2. Open the source evidence and record baseline/version. 3. A new catalog line arrives. 4. Inspect the result and decision reasons. 5. Open audit/output evidence and compare source values. 6. Record pass/fail and active handling time. | cost delta remains visible but margin is null and export is blocked. margin_impact.new_margin IS NULL; price_change.status='NEEDS_REVIEW'. No unexpected cross-tenant data or unapproved side effect. |
| MG-UAT-4: Approved recommendation | pricing manager; a 145.00 proposed sell price has complete evidence | 1. Sign in with the intended role. 2. Open the source evidence and record baseline/version. 3. I approve current version 4. 4. Inspect the result and decision reasons. 5. Open audit/output evidence and compare source values. 6. Record pass/fail and active handling time. | the proposal is locked and one approved-export job is queued. price_proposal.proposed_sell_price=145.00; approval count=1. No unexpected cross-tenant data or unapproved side effect. |
| MG-UAT-5: Catalog replay | pricing manager; catalog hash H1 was already processed | 1. Sign in with the intended role. 2. Open the source evidence and record baseline/version. 3. The same file is uploaded again. 4. Inspect the result and decision reasons. 5. Open audit/output evidence and compare source values. 6. Record pass/fail and active handling time. | the original catalog result is returned with no second price-change alert. UNIQUE(tenant_id,supplier_id,sha256); price_change count unchanged. No unexpected cross-tenant data or unapproved side effect. |

**Product outcome gate:** ≥99% arithmetic correctness; ≥95% alert precision for material supplier changes; ≥80% review SLA attainment. **Control gate:** Margin estimates are operational calculations. Agents explain source deltas; a pricing owner approves every exported price recommendation. **Measured usability gate:** at least 4/5 scenarios completed without coaching by each daily operator; no critical accessibility barrier; all financial/identity controls pass for every participant. A high usability rate cannot compensate for a control failure.

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
