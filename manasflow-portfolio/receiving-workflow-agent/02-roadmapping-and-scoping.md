# Receiving Workflow Agent — Roadmapping & Scoping

[Portfolio index](../README.md) · [Phase 1](01-ideation-and-strategy.md) · [Phase 2](02-roadmapping-and-scoping.md) · [Phase 3](03-requirements-prd-frd.md) · [Phase 4](04-sprint-execution-and-triage.md) · [Phase 5](05-testing-qa-and-uat.md) · [Phase 6](06-launch-gtm-and-telemetry.md)

**Document control:** RW-P2, revision 1.0, authored 2026-10-17. Status: proposed portfolio design; implementation, customer research, tests and stakeholder approvals have not been performed.

**Source boundary:** Expansion offer, source p. 20: proposed when AWAITING_RECEIPT volume is high. Detailed receipt workflow is an architectural proposal. [Primary document](../../MatchClerk-Project-Implementation.pdf). Proposed architecture and assumptions are explicitly recorded in [the evidence register](../source-and-assumptions.md).

## [PM] Release strategy and roadmap

Planning baseline starts 2027-04-05; these are scenario dates, not funded delivery promises. One six-week core MatchClerk build follows the source; expansion MVP durations are estimates. Dependencies and shared specialist capacity can move dates. All changes pass the gates in Phase 5.

```mermaid
gantt
    title Receiving Workflow Agent reference release plan
    dateFormat YYYY-MM-DD
    axisFormat %b %Y
    section Discovery and MVP
    Source and connector proof :a1, 2027-04-05, 7d
    Build, review and integration :a2, after a1, 28d
    SIT, UAT and pilot gate :a3, after a2, 7d
    MVP pilot :milestone, 2027-05-17, 0d
    section V1.0
    Offline capture :b1, 2027-05-17, 28d
    V1.0 general availability :milestone, 2027-06-14, 0d
    section V2.0 Enterprise
    Contracted enterprise hardening :c1, 2027-06-14, 70d
    V2.0 enterprise gate :milestone, 2027-08-23, 0d
```

| Quarter | Milestone / target date | Scope | Exit gate / owner |
| --- | --- | --- | --- |
| 2027 Q2 | Discovery starts 2027-04-05 | Integration evidence, baseline research and architecture decisions | Two partners plus source capability proof / PM + BSA |
| 2027 Q2 | MVP → 2027-05-17 | PO line lookup; packing-slip capture; partial accepted/damaged quantities; attestation; cumulative allocation; rematch event; audit | All Must stories accepted and no critical control defect / PM + BSA |
| 2027 Q2 | V1.0 → 2027-06-14 | Offline capture, site permissions and photo discrepancy evidence | Two paid cohorts meet outcome and service targets / PM |
| 2027 Q3 | V2.0 Enterprise → 2027-08-23 | Multi-warehouse receiving and entitlement-tested ERP receipt exports | Contracted connector proof, restore test, security evidence and revised capacity plan / architect |

MatchClerk portfolio V1.0 consolidates the source's weeks 7–10 v1.1 scope; version names here are planning labels. The [portfolio index](../README.md) shows the proposed expansion sequence. An MVP date cannot imply every downstream connector is available.

## [PM] MoSCoW scope contract

| Priority | Committed decision | Rationale / release treatment |
| --- | --- | --- |
| Must-have | PO line lookup; packing-slip capture; partial accepted/damaged quantities; attestation; cumulative allocation; rematch event; audit | Needed for the core outcome or a financial/tenant control; absent capability blocks pilot release |
| Should-have | Offline draft entry and site-based routing; discrepancy photos | Improves adoption or efficiency; can ship after control gates are stable |
| Could-have | Barcode-assisted item lookup | Experiment only after pilot demand and measurable incremental value |
| Won’t-have in this planning horizon | Warehouse management replacement; autonomous receipt attestation; inventory valuation journals | Explicitly excluded; a new business case and control design are required to reconsider |

### RICE model: ten ranked features

Reach is eligible deliverys affected per quarter in one representative account; total potential quarterly reach is 1000. Features overlap, so reach values must not be summed. Impact uses 3=massive, 2=high, 1=medium, 0.5=low; confidence is a fraction; effort is cross-functional person-weeks. **Score = reach × impact × confidence / effort.** Inputs are estimates, not measured adoption. A high score cannot displace mandatory security, approval or integrity controls.

| Rank | Feature | Reach / quarter | Impact | Confidence | Effort / person-weeks | Final score |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | PO lookup | 1000 | 3 | 85% | 3 | 850.00 |
| 2 | Packing-slip capture | 900 | 3 | 80% | 4 | 540.00 |
| 3 | Partial quantity attestation | 850 | 3 | 80% | 4 | 510.00 |
| 4 | Cumulative receipt ledger | 800 | 2 | 80% | 3 | 426.67 |
| 5 | Damage and shortage review | 750 | 3 | 75% | 4 | 421.88 |
| 6 | Invoice rematch events | 700 | 2 | 80% | 3 | 373.33 |
| 7 | Photo evidence | 600 | 2 | 70% | 3 | 280.00 |
| 8 | Offline draft capture | 500 | 1 | 65% | 2 | 162.50 |
| 9 | Barcode lookup | 300 | 1 | 60% | 2 | 90.00 |
| 10 | ERP receipt export | 200 | 0.5 | 50% | 6 | 8.33 |

**Sequencing rule:** Deliver Must controls and integration proof first, then highest-scoring feasible workflow features. Re-score after the first 100 eligible records or four pilot weeks, whichever comes first. Record changes with evidence in the next Phase 1 revision.

## [BSA] Non-functional requirements

All values are proposed acceptance targets unless identified as source-derived. Percentiles are calculated on successful and failed eligible operations, with timeouts included at their timeout value; do not hide failed work from latency reporting.

| ID | Dimension | Concrete requirement | Measurement / release proof |
| --- | --- | --- | --- |
| RW-NFR-01 | Capacity | 25 tenants × 1,000 deliverys/month; 250 concurrent UI sessions. | Load a representative 100,000-row tenant history; run a 60-minute mixed workload. |
| RW-NFR-02 | Throughput and queue | API accepts 10 requests/sec sustained. Expensive new jobs: 0.25/sec sustained and 0.5/sec for 60 sec; 32 worker slots scaling to 64. | Rate-limit excess costly work explicitly with 429 + Retry-After; maintain queue wait p95 <15 sec under the declared profile. |
| RW-NFR-03 | API latency | Read p95 ≤400 ms / p99 ≤1000 ms; accepted mutations p95 ≤600 ms / p99 ≤1500 ms. | Excludes agent/provider processing from synchronous handler; measure full async journey separately. |
| RW-NFR-04 | Agent and end-to-end latency | Agent p95 <60 sec; automated intake-to-review-ready p95 <120 sec for supported-size inputs. | Human waiting is excluded from automated duration and reported separately. Background jobs have 15-minute hard timeout and explicit recovery state. |
| RW-NFR-05 | Availability | MVP 99.5% monthly 24×7 API availability; GA target 99.9% after observed pilot reliability. | MVP error budget 216 min per 30-day month; GA 43.2 min. Source business-hour-weighted 99.5% is retained as a comparison, with calendar coverage proposed here. |
| RW-NFR-06 | Identity and authorization | OIDC login; MFA for OWNER and CONTROLLER; explicit RBAC on every request; short-lived sessions; 10-minute single-use email action nonce. | GET never approves. Link opens review; authenticated POST plus CSRF control performs action. Test every role and tenant negative case. |
| RW-NFR-07 | Encryption and isolation | TLS 1.2+; AES-256 managed encryption for DB/S3; KMS envelope encryption for OAuth tokens; tenant RLS with FORCE ROW LEVEL SECURITY. | Session-derived tenant only; composite tenant foreign keys; signed object URLs expire in 60 sec; no public bucket. |
| RW-NFR-08 | SOC 2 control readiness | Access reviews quarterly, change approvals, incident records, vendor DPAs and recovery evidence; no claim of SOC 2 certification. | Security owner maintains control/evidence register; audit report required before any certification claim. |
| RW-NFR-09 | Recovery and durability | Database PITR 30 days; RPO ≤15 min and RTO ≤4 hours; restore drill before launch and quarterly. | SQS retention 14 days, DLQ redrive via operation ledger; object versioning; encrypted backup restore proof. |
| RW-NFR-10 | Privacy and AI handling | Raw evidence default retention 24 months; active-store deletion within 30 days of verified offboarding request; backups age out within another 30 days unless legal hold. | Region-pinned EU deployment by default; approved inference vendors only; no training on tenant data; zero retention only when contractually supported. |
| RW-NFR-11 | Accessibility | WCAG 2.1 AA on onboarding, review and approval; full keyboard path and screen-reader labels. | Manual keyboard/screen-reader review plus automated checks; no critical accessibility blocker. |
| RW-NFR-12 | Correctness and observability | Decimal money; UTC timestamps plus tenant timezone; source lineage, actor and operation IDs on each mutation. | No duplicate external effects under replay tests; no prompt/PDF/token content in application logs. |

**Dependency constraint:** Requires synchronized PO lines and MatchClerk invoice rematch subscriber. **Business-rule constraint:** accepted_qty = delivered_qty - damaged_qty. Partial receipts accumulate per PO line. Returns are append-only negative adjustments linked to the original receipt; never erase a historical attestation. API licenses and entitlement checks are paid dependencies, not assumed free infrastructure. Current provider constraints and the architecture decision record live in [platform architecture](../platform-architecture.md).
