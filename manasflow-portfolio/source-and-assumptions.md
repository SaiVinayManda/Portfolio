# Source, evidence and assumption register

[Portfolio index](README.md) · [Platform architecture](platform-architecture.md)

Primary source: [MatchClerk Project Implementation PDF](../MatchClerk-Project-Implementation.pdf), 22 pages, document version 1.0 dated 2026-08-07. Read in full; expansion-map and integration pages were also rendered and visually checked. Portfolio authored 2026-10-17. The supplied PDF outranks public marketing pages for Manasflow objectives; current official vendor documentation informs integration constraints and competitor claims.

## Exhaustive product/offer extraction

| Source name / alias | Page evidence | Treatment | Core objective from source |
| --- | --- | --- | --- |
| MatchClerk / SME 3-Way Invoice Matcher | 1–19, 21–22 | Fully specified product; one six-phase pack | Invoice extraction → PO/receipt match → coded accounting entry → human approval |
| Full AP digital employee / full AP agent | 2, 18, 20 | Umbrella expansion bundle; separate orchestration pack, no duplicate invoice engine | Expand the daily AP wedge into broader supervised AP work |
| AP Exception Agent / AP exception agent | 14, 20 | Named expansion; separate pack | Vendor statement reconciliation; duplicate/fraud flags; payment-run preparation |
| Procurement Digital Employee / procurement agent / procurement employee | 2, 20 | Named expansion; aliases combined in one pack | PO drafting from approved requests; vendor scorecards; catalog price watch |
| MarginGuard | 20 | Named attachment; separate pack with explicitly inferred scope | Catalog price watch; cost/margin analysis is proposed interpretation |
| Receiving workflow agent | 20 | Upsell offer; separate pack | Address high AWAITING_RECEIPT volume; detailed attestation flow is proposed |
| Job-costing / project ops agents; job costing sync | 20 | Related capabilities combined in one pack | Contractor job-cost synchronization and project operations support |
| Vendor onboarding agent | 14 | Named expansion; separate pack with inferred functional scope | Supplier onboarding; exact verification and master-sync controls are proposed |
| Enterprise AI Systems / Enterprise Systems engagement | 2, 20 | Service/solution tier; separate reference architecture pack | AP/inventory/project hub, cross-department integration and custom multi-agent retainers |

Vendor onboarding agent is mentioned on p. 14 as a parallel upsell and receives its own six-phase pack. Full AP references its verified outputs rather than owning a second vendor master. Payment-run preparation, statement assist, mobile app, supplier portal and NetSuite/Sage/Exact connectors are features or explicit non-goals, not standalone named products. Coupa, GEP, Zycus, QBO, Xero, Slack and model providers are competitors/dependencies, not Manasflow products. No additional products are inferred from the manasflow.com domain.

## Source-preservation and refinement decisions

| Decision ID | Source statement / ambiguity | Portfolio decision | Confidence / validation |
| --- | --- | --- | --- |
| D-01 | Source lists multiple auth, ORM, queue and hosting options | Next.js + TypeScript; NestJS; Prisma/PostgreSQL; Auth.js/OIDC; SQS; Redis cache; AWS regional workers/storage. | Proposed; validate team skill and deployment proof |
| D-02 | Source describes QBO draft and human approval in potentially conflicting order | Hold all QBO proposals internally until human approval; Xero DRAFT creation also requires approval in this design. | Conservative capability assumption; prove exact QBO semantics in the target subscription |
| D-03 | Source implies goods receipts can be read from accounting | Use a distinct attested receiving ledger/CSV or inventory adapter; no assumption of universal native QBO/Xero goods-receipt API. | Integration discovery required |
| D-04 | Source asserts SME three-way match gap | Treat differentiation as research hypothesis; BILL/Ramp publish three-way match capability. | Current vendor documentation; competitive demos required |
| D-05 | Source says learn from tenant corrections | Use tenant-scoped approved mapping rules; no model training/fine-tuning on Xero API data. | Xero developer terms FAQ; verify legal and vendor contract review before production |
| D-06 | Source describes max(percent,absolute) tolerance with optional both rule | ALLOW_EITHER uses max; REQUIRE_BOTH uses min; zero quantity tolerance by default; compare line net price and tax separately. | Source-aligned; boundary tests required |
| D-07 | Source places duplicates in v1.1 | Exact duplicate financial-write prevention is mandatory in MVP; fuzzy duplicate investigation remains expansion scope. | Safety-driven design decision |
| D-08 | Source has 85% operational extraction and 80% QA F1 | Keep separately defined metrics; neither is relabeled as the other. | Golden-set evaluation required |
| D-09 | Source scale/pricing belongs to MatchClerk | Carry source scale and prices into MatchClerk; expansion scale, pricing and ROI are declared assumptions. | Pilot measurement required |
| D-10 | Source 6-week MVP and later releases | Preserve MatchClerk 6-week scenario; expansion calendars are uncommitted phased estimates. | Actual staffing and funding not established |
| D-11 | Source optional free–500 pilot and optional implementation fee | Use USD 500 credited pilot and source-recommended fees; design-partner setup fee waived. | Commercial hypothesis; no actual signed terms |
| D-12 | Source open non-PO invoice decision | Block non-PO external posting in MVP; later alternate approval controls require explicit scope. | Proposed conservative scope decision |
| D-13 | Full AP, receiving, MarginGuard and enterprise detail is brief | Design distinct bounded contexts while reusing a shared control plane. | Architectural proposal; no claim of source-defined detailed features |
| D-14 | Source mentions vendor onboarding as a named upsell | Give it a separate bounded context and phase pack; Full AP consumes verification events; no bank-detail changes. | Source p. 14; customer control validation required |

## Research integrity

Personas, roadblocks, tickets, sprint capacity, future roadmaps, ROI scenarios and metric targets are designed examples. No customer interviews, revenue, measured performance, test execution, compliance certification or stakeholder signatures are fabricated. Proposed dates and prices are concrete planning inputs and can be replaced by observed evidence.

Competitor references appear beside claims in each Phase 1 matrix. Only official vendor material supports capability statements. Published marketing figures are not treated as independent benchmark evidence. No competitor pricing is copied into the proposal.

## Critical external constraints

| Official source | Observed constraint | Design implication |
| --- | --- | --- |
| [QBO PurchaseOrder reference](https://developer.intuit.com/app/developer/qbo/docs/api/accounting/all-entities/purchaseorder) | Purchase-order and Bill relationships exist; exact line behavior and entitlements require proof. | Do not substitute estimates/bills for a true PO and continue to market it as three-way matching. |
| [Intuit purchase order help](https://quickbooks.intuit.com/learn-support/en-us/help-article/purchase-orders/add-purchase-orders-expenses-bills-checks-online/L9pRK49uC_US_en_US) | PO support depends on subscription; published help identifies Plus/Advanced. | Qualify SKU/region early; use supported PO source or decline three-way qualification. |
| [Xero Invoices](https://developer.xero.com/documentation/api/accounting/invoices) | ACCPAY invoices support DRAFT; AUTHORISED changes accounting state. | Keep states and authorization commands distinct. |
| [Xero idempotent requests](https://developer.xero.com/documentation/guides/idempotent-requests/idempotency/) | Provider idempotency keys expire after six minutes. | Persist local operation identity indefinitely within retention; reconcile after cache expiry rather than retry blind. |
| [Xero limits](https://developer.xero.com/faq/limits) | Published per-connection limits include five concurrent calls, 60/minute and 5,000/day; 429 supplies Retry-After. | Use lower internal per-connection caps and current response headers; reserve capacity for approved writes. |
| [Xero developer FAQ](https://developer.xero.com/faq) | API-derived data may not train AI/ML models; platform fees and tiers can apply. | Use consented inference under approved terms, not training; include API licensing in operating cost. |
| [Intuit webhook best practices](https://static.developer.intuit.com/output_html/qbo/docs/develop/webhooks/best-practices.html) | Webhooks require robust notification handling. | Validate signature, acknowledge durably and reconcile current source state on replay. |
| [BILL AP](https://www.bill.com/product/accounts-payable/) and [Ramp receiving/match](https://support.ramp.com/3-way-match-with-ramp-procurement/) | Both publish matching capabilities. | Validate a specific SME deployment/evidence advantage rather than claiming no alternatives. |
