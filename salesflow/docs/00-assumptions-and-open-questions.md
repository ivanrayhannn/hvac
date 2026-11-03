# SalesFlow — Assumptions, Open Questions & Planned Deviations

| Item | Value |
|---|---|
| Product name (working) | **SalesFlow** — an order-to-cash (O2C) web app that copies how SD business logic works. It contains no SAP code, branding or UI. |
| Document | 00 — Pre-requisite to Deliverable 1 |
| Version | 0.1 (draft for approval) |
| Date | 2026-11-03 |
| Status | **Waiting for approval** |

> Rule applied: *"Before coding, list your assumptions and open questions. Do not assume silently."*
> Each assumption below has a proposed default. If you don't change it, we build on that default.
> Each open question (Q-xx) needs an answer before the phase shown in its "Blocks" column.

---

## 1. Naming convention (our own terms; the SAP equivalent is shown only so SD consultants can follow)

| SalesFlow term | Code prefix | SAP SD concept it corresponds to |
|---|---|---|
| Company | `CMP` | Company Code |
| Sales Unit | `SU` | Sales Organization |
| Channel | `CH` | Distribution Channel |
| Product Line | `PL` | Division |
| Sales Area | `SA` = SU+CH+PL | Sales Area |
| Site | `ST` | Plant |
| Storage Bin Location | `SL` | Storage Location |
| Dispatch Point | `DP` | Shipping Point |
| Sales Office / Sales Team | `SO` / `TM` | Sales Office / Sales Group |
| Business Partner – Customer | `CUS` | Customer Master (KNA1/KNVV) |
| Partner Role | SP / SH / BP / PY | Sold-to / Ship-to / Bill-to / Payer |
| Item (Material) | `MAT` | Material Master |
| Customer Item Reference | `CIR` | Customer-Material Info Record |
| Price Element | `PE` | Condition Type |
| Price Rule | `PR` | Condition Record |
| Lookup Strategy | `LS` | Access Sequence |
| Pricing Scheme | `PS` | Pricing Procedure |
| Document Type | `DT` | Sales/Delivery/Billing Document Type |
| Line Category | `LC` | Item Category |
| Copy Rule | `CR` | Copy Control |
| Required-Field Check | `RFC` | Incompletion Procedure |
| Output Template | `OT` | Output Determination |
| Availability Check | `ATP` | ATP |
| Credit Control | `CC` | Credit Management (FSCM-lite) |
| Goods Issue | `GI` | Post Goods Issue |
| Invoice / Credit Note / Debit Note | `INV` / `CRN` / `DBN` | Billing types F2 / G2 / L2 |
| AR Posting (interface) | `ARP` | Accounting document (stub) |

**Document type codes (seed configuration — all stored in tables, so you can change them):**

| Code | Name | Category | Next document |
|---|---|---|---|
| `IQ` | Inquiry | Pre-sales | QT |
| `QT` | Quotation | Pre-sales | OR, RO |
| `OR` | Standard Order | Order | Delivery `LD` → Invoice `INV` |
| `RO` | Rush Order | Order | Delivery created at save → `INV` |
| `FC` | Free-of-Charge Order | Order | Delivery `LD`, **no billing** |
| `RE` | Returns Order | Returns | Return delivery `LR` → Credit Note `CRN` |
| `CR` | Credit Note Request | Memo request | `CRN` (billing block until released) |
| `DR` | Debit Note Request | Memo request | `DBN` (billing block until released) |
| `LD` / `LR` | Outbound / Return Delivery | Delivery | GI / Goods Receipt |
| `INV` / `CRN` / `DBN` / `CAN` | Invoice / Credit Note / Debit Note / Cancellation | Billing | ARP |

---

## 2. Assumptions (A-xx)

### 2.1 Context & platform
| ID | Assumption (proposed default) | Why it matters |
|---|---|---|
| A-01 | **Industry = Manufacturing/distribution of discrete goods** (e.g. automotive spare parts). No variant configuration, batch management or serial numbers in v1. | Item and stock model |
| A-02 | **Stack = .NET 8 Web API (Clean Architecture) + SQL Server 2022 + React 18 / Next.js 14 (TypeScript)**. EF Core 8 for writes; Dapper for heavy reports. | All technical deliverables |
| A-03 | **Deployment = cloud-ready containers (Docker)**. They can also run on-premise using the same images. | TSD, DevOps |
| A-04 | **Multi-company in one database**, kept apart by a `CompanyId` on every transaction and an access filter on Sales Area. Not one database per tenant. | ERD, security |
| A-05 | **UI language = English first**, with i18n resource files ready for Bahasa Indonesia (`en`, `id`). | UI design |
| A-06 | **Default currency IDR (0 decimals)**. Multi-currency at document level, using a daily exchange-rate table. Amounts are stored in document currency **and** company currency. | Pricing, billing |
| A-07 | **Tax = Indonesian VAT (PPN)** as a pricing element (rate held in a table, seed 11% / DPP-adjusted 12% configurable). Only an output tax code, no withholding (PPh) in v1. e-Faktur numbering is out of scope. | Pricing, billing |
| A-08 | Units of measure: base UoM per item plus alternative UoMs with conversion factors (e.g. 1 BOX = 12 PC). Quantities use 3 decimals. | Item master, ATP |
| A-09 | Time zone is held per company (default Asia/Jakarta). Dates are stored in UTC. Business dates (requested delivery, billing date) are stored as `date`. | Scheduling |

### 2.2 Inventory & logistics (no MM module exists)
| ID | Assumption | Why it matters |
|---|---|---|
| A-10 | SalesFlow keeps a **simplified stock ledger** (on-hand per Item/Site/Storage Location plus stock movements). Goods Issue (GI) posts a movement. A REST endpoint lets an external MM/WMS **push stock** (overwrite or delta). | ATP, GI |
| A-11 | **Stock is not reserved at order save** (SAP-like). ATP counts confirmed open order quantities and open deliveries as "committed" (see A-12). | ATP |
| A-12 | ATP formula (per Item/Site, optional Storage Location): `Available = OnHand − SafetyStock − ConfirmedOpenOrders − OpenDeliveryQty(not yet GI)` plus planned receipts **only if** the "include receipts" flag is on (v1 = off). There are no replenishment lead times in v1. If stock is short, the system offers the **next available date = none** (backorder) unless planned receipts are switched on. | ATP |
| A-13 | Picking is **quantity-confirmation based** (no warehouse task / bin management). Packing = handling units (carton/pallet), one level, optional per delivery type. | Delivery |
| A-14 | No route, transport planning or shipment cost document in v1. Freight is priced as a pricing element on the order. | Delivery |

### 2.3 Sales & pricing
| ID | Assumption | Why it matters |
|---|---|---|
| A-15 | Pricing uses a **condition-technique engine** driven by tables: Pricing Scheme → steps → Price Element → Lookup Strategy → key-combination tables → Price Rules (validity, scales). Pricing Scheme is chosen from **Sales Area + Customer pricing group + Document pricing group**. | Pricing |
| A-16 | Seed price elements: `BASE` (base price, mandatory), `DCUS` (customer % discount), `DMAT` (item % discount), `DMAN` (manual discount, needs authorization), `FRT` (freight, fixed or per unit), `VAT` (tax). Subtotals: Gross, Net, Tax Base, Total. | Pricing |
| A-17 | Scales are **quantity-based, "from" scale** (graduated pricing is out of v1). | Pricing |
| A-18 | Re-pricing rule: copy from Quotation keeps prices (no re-price). At billing, prices are copied from the order (billing date ≠ pricing date). The user can trigger re-price manually. Re-price types: `A` keep all, `B` full re-price, `C` keep manual elements. | Copy control |
| A-19 | Rounding: line level, half-up, to currency decimals. The document total is the sum of rounded lines, not re-rounded. | Pricing |

### 2.4 Credit
| ID | Assumption | Why it matters |
|---|---|---|
| A-20 | Credit is controlled at **Payer level per Credit Control Area (one per company)**. Exposure = Open AR (from the FI interface or local open invoices) + Open billing-due deliveries + Open confirmed orders (dynamic check, horizon in days). | Credit |
| A-21 | Check is configurable per Document Type + Customer risk class: **None / Static / Dynamic**, reaction **Warning / Block**. A credit check runs at order save, delivery creation and GI. | Credit |
| A-22 | Open AR is supplied by FI through an API. Until that exists, SalesFlow calculates open AR from its own invoices minus payments recorded through a **"payment received" stub endpoint**. | Credit |

### 2.5 Billing & accounting
| ID | Assumption | Why it matters |
|---|---|---|
| A-23 | Billing types: delivery-related (OR, RO, RE via return delivery) and order-related (CR, DR, service items). Collective billing groups by Payer + Billing date + Currency + Terms + Incoterms + Sales Area. | Billing |
| A-24 | The accounting document is an **outbound ARP message** (journal: Dr AR / Cr Revenue / Cr VAT payable). It is stored in an outbox table and posted to FI through REST. SalesFlow has **no** GL. | FI interface |
| A-25 | Cancelling a billing document creates a reverse document (`CAN`). It reopens the preceding delivery or order for billing. It never deletes. | Billing |

### 2.6 Security & audit
| ID | Assumption | Why it matters |
|---|---|---|
| A-26 | Authentication = OIDC (Azure AD / Keycloak) with JWT. Local user store for demo. Authorization = Role → Permission (transaction code + activity: Create/Change/Display/Release) × Sales Area scope. | RBAC |
| A-27 | Audit trail at field level for master data and documents (table, key, field, old value, new value, user, timestamp, reason). It is written by an EF Core interceptor. | Audit |
| A-28 | Output = server-side PDF (QuestPDF) using template tables. Email dispatch through SMTP is optional. Printing happens in the browser. | Output |

---

## 3. Open questions (Q-xx) — please answer or accept the default

| ID | Question | Proposed default | Blocks |
|---|---|---|---|
| Q-01 | Which industry do you mean: automotive (OEM parts / dealer), general manufacturing or distribution? Do you need **batch / serial / shelf-life** handling? | Spare-parts distributor, no batch/serial | ERD |
| Q-02 | Do you confirm the stack (.NET 8 + SQL Server + Next.js)? Or do you prefer a lighter demo stack (e.g. Node/NestJS + PostgreSQL)? | .NET 8 + SQL Server + Next.js | TSD, code |
| Q-03 | UI language for go-live: **English only, Bahasa only, or bilingual toggle**? | Bilingual toggle, English default | FSD |
| Q-04 | Which PPN rate and tax-base rule should we seed (11% or 12% with DPP 11/12)? Is **e-Faktur** (tax invoice numbering / CSV export) needed? | Configurable, e-Faktur out of scope | Pricing, billing |
| Q-05 | Is there an existing MM/WMS/ERP that owns stock? Or is SalesFlow the stock system of record for the demo? | SalesFlow owns stock, API for push | ATP, GI |
| Q-06 | Is there an existing FI/AR system to receive invoices and supply open AR? What format (REST/JSON, CSV, SFTP)? | REST outbox, stub receiver | Phase 4 |
| Q-07 | **Credit release authority**: is Sales Supervisor enough, or do we need value-based approval tiers (e.g. > IDR 500 M → Finance Manager)? | Tiered table, 1 tier seeded (Supervisor) | Credit |
| Q-08 | **Manual price override**: who may change it, and is there a max % limit per role? | Sales Admin ≤ 5%, Supervisor ≤ 15%, above = blocked | Pricing |
| Q-09 | Partial delivery policy default: per customer, per item or both? Maximum number of partial deliveries? | Per customer, overridable per line; max 9 | ATP, delivery |
| Q-10 | Do we need **Returns with inspection** (quality decision: restock / scrap) or a simple return-to-stock? | Simple: received into a "Returns" storage location, then a manual transfer | Returns |
| Q-11 | Number range format, e.g. `SO-2026-000001` (prefix + year + running number, reset yearly)? Shared or per Company? | Per Company + Doc type + Year | Number ranges |
| Q-12 | Is a **customer portal / external ordering** in scope, or are all orders entered by internal users? | Internal only; API for EDI-like order upload | Scope |
| Q-13 | Approval for **quotations** (e.g. discount > X% needs supervisor approval before sending)? | Yes, reuses the manual-discount rule | Quotation |
| Q-14 | Do you need **intercompany / third-party (drop-ship)** sales? | Out of scope v1 | Scope |
| Q-15 | Expected volumes (orders/day, lines/order, concurrent users)? These drive indexing, paging and the report strategy. | 2,000 orders/day, 20 lines, 100 users | TSD |
| Q-16 | Fiscal calendar and period lock: should billing be prevented in closed periods? | Calendar month; period lock table | Billing |

---

## 4. Planned deviations from standard SAP SD behaviour (D-xx)

| ID | Deviation | Reason |
|---|---|---|
| D-01 | Configuration (document types, copy rules, pricing, incompletion) is kept in **plain relational tables with an admin UI**, not IMG/SPRO with its many sub-tables. Some rarely used SAP fields are left out. | Simpler to maintain; same concept |
| D-02 | **No requirement/formula routines in ABAP.** Copy-rule and pricing "requirements" are a fixed, named list of rule keys (e.g. `REQ_ITEM_NOT_REJECTED`, `REQ_CUSTOMER_TAX_LIABLE`) implemented in code and chosen in configuration. | We can't run user code safely; a named rule set covers the common cases |
| D-03 | **Single-level partner hierarchy.** Partner functions are SP, SH, BP, PY plus optional Sales Employee (ZE). No customer hierarchy pricing. | Scope |
| D-04 | **Simplified credit management**, close to classic SD credit management (FD32 / OVA8 style). No SAP FSCM credit scoring or case management. | Scope |
| D-05 | **Picking without WM/EWM.** Pick quantity is confirmed directly on the delivery. Packing has one handling-unit level. | No warehouse module |
| D-06 | **ATP without MRP**: no planned orders or replenishment lead time. It checks against stock, plus optional planned receipts (A-12). The backorder "rescheduling" job is a simple re-run in FIFO by requested date. | No PP/MM |
| D-07 | **Accounting is an interface message**, not a document posted in a local GL (A-24). | No FI module |
| D-08 | Output determination uses a **template + rule table** (Doc type × Partner × Medium) instead of the full condition technique for output. | Simpler; meets the requirement |
| D-09 | Statuses are stored as explicit **status columns with documented state machines** instead of the SAP VBUK/VBUP status field families. They can be calculated from flow. | Readable and queryable |
| D-10 | The quotation "valid-to" date can **auto-expire** the quotation to status `Expired` (SAP only uses it as a date check). | Common business ask |

---

## 5. Phase plan & approval gates

| Gate | Deliverable | Status |
|---|---|---|
| G0 | This document (assumptions / questions / deviations) | **Waiting for approval** |
| G1 | 01 Business Process Flow (swimlanes) | **Waiting for approval** (delivered with G0) |
| G2 | 02 BRD summary + BR-xx list | Not started |
| G3 | 03 ERD | Not started |
| G4 | 04 FSD per module | Not started |
| G5 | 05 TSD | Not started |
| G6 | 06 Source code, Phases 1–5 (each runnable and testable on its own) | Not started |
| G7 | 07 Testing strategy + UAT scenarios | Not started |
| G8 | 08 Seed data | Not started (outline in G2) |
