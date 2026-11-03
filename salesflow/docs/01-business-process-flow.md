# SalesFlow — 01 Business Process Flow (Order-to-Cash)

| Item | Value |
|---|---|
| Document | Deliverable 1 — Business Process Flow |
| Version | 0.1 (draft for approval) |
| Date | 2026-11-03 |
| Depends on | `00-assumptions-and-open-questions.md` |
| Notation | Mermaid flowcharts. Each `subgraph` is a swimlane (role). Diamonds = decisions. Steps are tagged `Pxx.yy` so the BRD / FSD / UAT can trace back to them. |

---

## 0. Roles (swimlanes)

| Lane | Role | Main responsibility | Typical permissions |
|---|---|---|---|
| CUS | **Customer** (external) | Sends RFQ / PO, receives goods, pays | — |
| SA | **Sales Admin** | Inquiry, quotation, sales order, returns and memo requests; fixes incompletion | Create/Change sales documents within their Sales Area |
| SV | **Sales Supervisor** | Approves discounts, releases credit blocks and delivery blocks, approves memo requests | Release, override price above limit |
| WH | **Warehouse / Shipping** | Creates deliveries, picks, packs, posts Goods Issue, receives returns | Create/Change delivery, post GI / GR |
| BF | **Billing / Finance** | Creates invoices and credit/debit notes, cancels billing, checks the AR posting | Create/Cancel billing, release billing block |
| AD | **Admin** | Organisation structure, master data governance, configuration, users/roles | Configuration and master-data maintenance |
| SYS | **SalesFlow (system)** | Pricing, ATP, credit check, incompletion, number ranges, output, document flow, AR interface | — |
| EXT | **External systems** | MM/WMS (stock), FI (AR posting, open AR) | API clients |

---

## 1. Level 0 — Value chain

```mermaid
flowchart LR
  A[Org & Master Data<br/>Setup] --> B[Pre-Sales<br/>Inquiry · Quotation]
  B --> C[Sales Order<br/>Pricing · ATP · Credit]
  C --> D[Outbound Delivery<br/>Pick · Pack · GI]
  D --> E[Billing<br/>Invoice]
  E --> F[AR Posting<br/>to FI]
  F --> G[Payment<br/>Received]
  C -.-> R[Returns / Memo<br/>Requests]
  R -.-> E
```

---

## 2. Level 1 — End-to-end O2C swimlane (happy path plus the main exceptions)

```mermaid
flowchart TB
  subgraph CUS[Customer]
    C1([RFQ / PO received])
    C9([Goods received])
    C10([Pays invoice])
  end

  subgraph SA[Sales Admin]
    S1[P02.01 Create Inquiry]
    S2[P02.03 Create Quotation<br/>with reference to Inquiry]
    S3{Customer accepts?}
    S4[P03.01 Create Sales Order<br/>with reference to Quotation or direct]
    S5[Complete missing data]
  end

  subgraph SYS[SalesFlow system]
    Y1[Determine Sales Area,<br/>partners, terms]
    Y2[Pricing: condition technique]
    Y3{Incomplete?}
    Y4[ATP check per schedule line]
    Y5{Credit check OK?}
    Y6[Assign number,<br/>save, output Order Confirmation]
    Y7[Delivery due list]
    Y8[Billing due list]
    Y9[Generate AR posting message]
  end

  subgraph SV[Sales Supervisor]
    V1{Release credit block?}
    V2[Reject / cancel order]
  end

  subgraph WH[Warehouse / Shipping]
    W1[P04.01 Create Outbound Delivery]
    W2[P04.03 Picking]
    W3[P04.04 Packing]
    W4[P04.05 Post Goods Issue]
  end

  subgraph BF[Billing / Finance]
    B1[P05.01 Create Invoice]
    B2[P05.04 Check AR posting]
  end

  subgraph EXT[External: FI / MM]
    X1[(Stock reduced)]
    X2[(AR open item)]
  end

  C1 --> S1 --> S2 --> S3
  S3 -- No / expired --> S2
  S3 -- Yes --> S4 --> Y1 --> Y2 --> Y3
  Y3 -- Yes --> S5 --> Y3
  Y3 -- No --> Y4 --> Y5
  Y5 -- Yes --> Y6
  Y5 -- No: credit block --> V1
  V1 -- Release --> Y6
  V1 -- Reject --> V2
  Y6 --> Y7 --> W1 --> W2 --> W3 --> W4
  W4 --> X1
  W4 --> C9
  W4 --> Y8 --> B1 --> Y9 --> X2 --> B2
  X2 --> C10
```

**Exceptions shown elsewhere:** insufficient stock (§4), credit block release (§5), billing cancellation (§7), returns (§8), credit/debit memo (§9).

---

## 3. P02 — Pre-sales: Inquiry → Quotation

```mermaid
flowchart TB
  subgraph SA[Sales Admin]
    a1[P02.01 Create Inquiry IQ<br/>customer, items, req. date] --> a2[P02.02 Save Inquiry]
    a3[P02.03 Create Quotation QT<br/>copy from IQ] --> a4[Set valid-from / valid-to]
    a4 --> a5{Manual discount<br/>above own limit?}
    a5 -- No --> a7[P02.05 Send Quotation PDF]
    a9{Customer response}
  end
  subgraph SV[Sales Supervisor]
    a6{P02.04 Approve discount?}
  end
  subgraph SYS[System]
    s1[Price items<br/>indicative ATP, no commitment]
    s2[Output: Quotation PDF]
    s3[Status IQ = Completed<br/>when fully referenced]
    s4[Job: QT valid-to passed<br/>→ status Expired]
  end
  subgraph CUS[Customer]
    c1([Receives quotation])
  end
  a2 --> a3 --> s1 --> a4
  a5 -- Yes --> a6
  a6 -- Approve --> a7
  a6 -- Reject --> a4
  a7 --> s2 --> c1 --> a9
  a9 -- Accept --> o[[Go to P03 Sales Order]]
  a9 -- Reject --> r[Set rejection reason on items<br/>QT status Completed-Rejected]
  a9 -- No answer --> s4
  a3 --> s3
```

| Step | Role | Activity | Rule / control (BR-xx defined in BRD) |
|---|---|---|---|
| P02.01 | SA | Create inquiry for a Sold-to in an authorised Sales Area | Sales Area must be valid for the customer |
| P02.03 | SA | Create quotation **with reference** (copy rule IQ→QT) or without reference | Copy rule decides which fields are copied |
| P02.04 | SV | Approve a discount above the Sales Admin limit (Q-08 / Q-13) | Approval is written to the audit trail |
| P02.05 | SA | Send the quotation | Output template QT-PDF |
| — | SYS | Quotation expiry job (D-10) | Expired QT can't be referenced |

---

## 4. P03 — Sales Order: create, price, ATP, credit, save

```mermaid
flowchart TB
  subgraph SA[Sales Admin]
    o1[P03.01 Create Sales Order<br/>doc type OR / RO / FC] --> o2[Enter Sold-to, PO no.,<br/>items, qty, req. delivery date]
    o5[Fix missing fields]
    o8{Accept proposal?}
    o9[Manual price change]
  end
  subgraph SYS[System]
    y1[Check Sales Area and customer<br/>sales-area data exist]
    y2[Determine partners SP/SH/BP/PY,<br/>terms, Incoterms, tax class]
    y3[Customer item ref → item]
    y4[Line category from doc type<br/>+ item category group]
    y5[Pricing scheme → price elements<br/>→ lookup strategy → price rules]
    y6{Mandatory BASE price found?}
    y7[ATP per schedule line]
    y8{Full qty available<br/>on req. date?}
    y9[Proposal: partial now + rest backorder,<br/>or complete later]
    y10{Required-field check passed?}
    y11{Credit check<br/>per doc type + risk class}
    y12[Set credit block<br/>status = Blocked-Credit]
    y13[Number range → SO no.<br/>Save, audit, document flow]
    y14[Output: Order Confirmation PDF]
    y15[RO: auto-create delivery]
  end
  subgraph SV[Sales Supervisor]
    v1{Price override<br/>within role limit?}
  end

  o2 --> y1 --> y2 --> y3 --> y4 --> y5 --> y6
  y6 -- No --> e1[Line incomplete: pricing error<br/>can save but billing blocked]
  y6 -- Yes --> y7
  e1 --> y7
  y7 --> y8
  y8 -- Yes --> y10
  y8 -- No --> y9 --> o8
  o8 -- Yes partial/backorder --> y10
  o8 -- No --> o2
  o2 -.-> o9 --> v1
  v1 -- Yes --> y5
  v1 -- No --> e2[Error: exceeds limit,<br/>needs Supervisor]
  y10 -- No --> o5 --> y10
  y10 -- Save as incomplete allowed --> y13
  y10 -- Yes --> y11
  y11 -- Pass --> y13
  y11 -- Warning --> y13
  y11 -- Block --> y12 --> y13
  y13 --> y14
  y13 --> y15
```

| Step | Role | Activity | Key controls |
|---|---|---|---|
| P03.01 | SA | Pick the document type. The Sales Area is limited to the user's authorisation scope | RBAC on Sales Area |
| P03.02 | SYS | Partner determination from customer master | Ship-to, Bill-to and Payer must be valid for the Sales Area |
| P03.03 | SYS | Line category determination: DocType + ItemCatGroup (+ usage, + higher-level line category) | Configured in the line category assignment table |
| P03.04 | SYS | Pricing (scales, validity on pricing date) | A missing BASE price marks the line incomplete |
| P03.05 | SYS | ATP: confirm qty per schedule line; offer partial/complete | Respects customer partial-delivery flag (Q-09) |
| P03.06 | SYS | Required-field check per Doc type / Line category / Schedule line category | The document can be saved incomplete, but **not delivered or billed** (configurable per field) |
| P03.07 | SYS | Credit check (static/dynamic) | Block → no delivery until released |
| P03.08 | SYS | Number assignment, document flow record, audit, Order Confirmation output | Number range per Company + DocType + Year |
| P03.09 | SA/SV | Change order (qty, date, price, rejection reason) | Changing qty or date runs ATP again. Increasing value runs the credit check again |

---

## 5. P03.10 — Credit block release & blocked-order worklist

```mermaid
flowchart LR
  subgraph SYS[System]
    k1[Order saved with credit block] --> k2[Blocked orders worklist<br/>filtered by Sales Area + amount tier]
    k5[Recalculate exposure on release]
    k6[Remove block, write audit,<br/>order appears in Delivery due list]
    k7[Notify Sales Admin]
  end
  subgraph SV[Sales Supervisor]
    k3[Review exposure:<br/>limit, open AR, open orders, overdue]
    k4{Decision}
  end
  subgraph SA[Sales Admin]
    k8[Inform customer / ask for payment]
  end
  k2 --> k3 --> k4
  k4 -- Release --> k5 --> k6
  k4 -- Reject --> k9[Set rejection reason on all items<br/>status Completed-Rejected] --> k7
  k4 -- Hold --> k8
  k6 --> k7
```

Re-check points: credit is re-checked at **order change (value increase)**, at **delivery creation** and at **GI** (configurable per Doc type, A-21). A release is valid for a set number of days or until the value goes up by more than the configured %.

---

## 6. P04 — Outbound Delivery → Picking → Packing → Goods Issue

```mermaid
flowchart TB
  subgraph WH[Warehouse / Shipping]
    d1[P04.01 Select from Delivery due list<br/>by Dispatch Point, date, Ship-to] --> d2[Create delivery LD]
    d4[P04.03 Picking: confirm picked qty<br/>per line / storage location]
    d5{Picked = delivery qty?}
    d6[Adjust delivery qty<br/>remainder stays open on order]
    d7[P04.04 Packing into handling units]
    d8[P04.05 Post Goods Issue]
  end
  subgraph SYS[System]
    s1{Order blocked?<br/>credit / delivery block / incomplete}
    s2[Group lines by Ship-to + Dispatch Point<br/>+ date + Incoterms]
    s3[Copy rule SO → LD<br/>ATP re-check, credit re-check]
    s4[Output: Delivery Note / Picking list]
    s5{Pre-GI checks:<br/>fully picked, packed if required,<br/>credit ok, stock ok}
    s6[Post stock movement 601-like<br/>OnHand −, open delivery −]
    s7[Update statuses: LD GI-done,<br/>SO delivery status]
    s8[Delivery appears in Billing due list]
    s9[Publish stock event to MM / WMS]
  end
  d1 --> s1
  s1 -- Blocked --> x1[Excluded from list,<br/>reason shown]
  s1 -- OK --> s2 --> d2 --> s3 --> s4 --> d4 --> d5
  d5 -- Yes --> d7
  d5 -- No --> d6 --> d7
  d7 --> d8 --> s5
  s5 -- Fail --> x2[Error log, GI not posted]
  s5 -- Pass --> s6 --> s7 --> s8
  s6 --> s9
```

| Step | Role | Activity | Key controls |
|---|---|---|---|
| P04.01 | WH | Create a delivery one by one or from the due list (collective) | Only lines with confirmed qty and no blocks |
| P04.02 | SYS | Dispatch Point determination (Site + shipping condition + loading group) | Configured table |
| P04.03 | WH | Picking confirmation | Picked qty ≤ delivery qty. Under-pick reduces the delivery (open qty returns to the order) |
| P04.04 | WH | Packing (optional per Delivery type) | All lines packed if "packing required" |
| P04.05 | WH | Post Goods Issue | Stock can't go negative (configurable). GI can be reversed until the delivery is billed |
| P04.06 | WH | Reverse GI | Only if not billed. Stock goes back. Status moves back to "Picked" |

---

## 7. P05 — Billing → AR posting → Cancellation

```mermaid
flowchart TB
  subgraph BF[Billing / Finance]
    b1[P05.01 Billing due list<br/>by Payer, billing date, Sales Area] --> b2[Create billing<br/>individual or collective]
    b5[P05.02 Release billing block<br/>e.g. pricing error fixed]
    b7[P05.03 Cancel billing doc]
    b8[P05.04 Monitor AR interface<br/>retry failed messages]
  end
  subgraph SYS[System]
    y1{Blocked?<br/>billing block / incomplete / pricing error}
    y2[Copy rule LD → INV or SO → INV<br/>pricing type A/B/C]
    y3[Split by Payer, currency, terms,<br/>Incoterms, Sales Area]
    y4[Number range, save,<br/>Output: Invoice PDF]
    y5[Create ARP message in outbox:<br/>Dr AR / Cr Revenue / Cr VAT]
    y6{FI ack?}
    y7[Status: Posted]
    y8[Status: Posting error<br/>retry with backoff]
    y9[Create CAN doc: reverse values,<br/>reopen LD/SO for billing,<br/>reverse ARP message]
    y10{Period open?<br/>Q-16}
  end
  subgraph EXT[FI]
    f1[(AR open item / reversal)]
  end
  b1 --> y1
  y1 -- Blocked --> b5 --> y1
  y1 -- OK --> y2 --> y3 --> y10
  y10 -- No --> x1[Error: period closed]
  y10 -- Yes --> y4 --> y5 --> f1 --> y6
  y6 -- Yes --> y7
  y6 -- No --> y8 --> b8
  b7 --> y9 --> f1
```

---

## 8. P06 — Returns: Returns Order → Return Delivery → Goods Receipt → Credit Note

```mermaid
flowchart TB
  subgraph CUS[Customer]
    r0([Complains / returns goods])
  end
  subgraph SA[Sales Admin]
    r1[P06.01 Create Returns Order RE<br/>with reference to Invoice] --> r2[Enter return reason, qty]
  end
  subgraph SV[Sales Supervisor]
    r3{P06.02 Approve return?<br/>remove billing block}
  end
  subgraph WH[Warehouse]
    r4[P06.03 Create Return Delivery LR] --> r5[P06.04 Post Goods Receipt<br/>into Returns storage location]
  end
  subgraph BF[Billing / Finance]
    r6[P06.05 Create Credit Note CRN<br/>from Return Delivery]
  end
  subgraph SYS[System]
    y1[Copy prices from invoice<br/>qty ≤ invoiced − already returned]
    y2[Billing block 'Return check'<br/>set by doc type]
    y3[Stock + into Returns SL]
    y4[ARP: Dr Revenue / Dr VAT / Cr AR]
  end
  r0 --> r1 --> y1 --> r2 --> y2 --> r3
  r3 -- Approve --> r4 --> r5 --> y3 --> r6 --> y4
  r3 -- Reject --> rx[Rejection reason on lines]
```

---

## 9. P07 — Credit Note Request / Debit Note Request (price corrections, no goods movement)

```mermaid
flowchart LR
  subgraph SA[Sales Admin]
    m1[P07.01 Create CR or DR<br/>with reference to Invoice] --> m2[Enter qty/value correction<br/>and order reason]
  end
  subgraph SYS[System]
    m3[Billing block set automatically]
    m6[Order-related billing:<br/>CRN or DBN]
    m7[ARP posting]
  end
  subgraph SV[Sales Supervisor]
    m4{P07.02 Approve<br/>value tier}
  end
  subgraph BF[Billing / Finance]
    m5[P07.03 Bill from Billing due list]
  end
  m2 --> m3 --> m4
  m4 -- Approve: remove block --> m5 --> m6 --> m7
  m4 -- Reject --> m8[Rejection reason]
```

---

## 10. P01 — Master data & configuration (Admin lane)

```mermaid
flowchart LR
  subgraph AD[Admin]
    a1[P01.01 Org structure:<br/>Company, Sales Unit, Channel,<br/>Product Line, Sales Area, Site,<br/>Storage Location, Dispatch Point, Office, Team]
    a2[P01.02 Config: Doc types, Line categories,<br/>Copy rules, Pricing scheme, Required-field check,<br/>Output templates, Number ranges, Block reasons]
    a3[P01.03 Master data: Customer + sales area data,<br/>Partners, Item + sales data, UoM,<br/>Customer item ref, Price rules, Credit limits]
    a4[P01.04 Excel upload:<br/>validate → preview → commit]
    a5[P01.05 Users, roles, Sales Area scope]
  end
  subgraph SYS[System]
    s1[Validation, audit trail,<br/>change history]
    s2[Upload error report per row]
  end
  a1 --> a2 --> a3
  a3 --> s1
  a4 --> s2
  a4 --> s1
  a5 --> s1
```

Order of dependency: **Org structure → Configuration → Master data → Transactions.** Phase 1 of the build covers this lane and can be tested on its own.

---

## 11. Document flow (reference chain) and copy rules (preview)

```mermaid
flowchart LR
  IQ[Inquiry IQ] --> QT[Quotation QT]
  QT --> OR[Order OR / RO / FC]
  OR --> LD[Delivery LD]
  LD --> GI[Goods Issue]
  LD --> INV[Invoice INV]
  INV --> ARP[(AR posting)]
  INV --> CAN[Cancellation CAN]
  INV --> RE[Returns Order RE]
  INV --> CR[Credit Note Req CR]
  INV --> DR[Debit Note Req DR]
  RE --> LR[Return Delivery LR]
  LR --> GR[Goods Receipt]
  LR --> CRN[Credit Note CRN]
  CR --> CRN
  DR --> DBN[Debit Note DBN]
  CRN --> ARP
  DBN --> ARP
  CAN --> ARP
```

| From \ To | QT | OR/RO | FC | RE | CR | DR | LD | LR | INV | CRN | DBN | CAN |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| IQ | ✔ | ✔ | | | | | | | | | | |
| QT | | ✔ | ✔ | | | | | | | | | |
| OR / RO | | ✔ (copy) | | | | | ✔ | | ✔ order-related lines only | | | |
| FC | | | | | | | ✔ | | — (no billing) | | | |
| RE | | | | | | | | ✔ | | ✔ order-related option | | |
| CR / DR | | | | | | | | | | ✔ (CR) | ✔ (DR) | |
| LD | | | | | | | | | ✔ | | | |
| LR | | | | | | | | | | ✔ | | |
| INV | | | | ✔ | ✔ | ✔ | | | | | | ✔ |
| CRN / DBN | | | | | | | | | | | | ✔ |

Every flow record stores: preceding doc/line, subsequent doc/line, quantity and value referenced, and the copy rule used. Status of the preceding document = f(referenced qty vs. original qty).

---

## 12. Status model overview (detailed state machines in FSD)

```mermaid
stateDiagram-v2
  direction LR
  [*] --> Open : saved
  Open --> Incomplete : required field missing
  Incomplete --> Open : completed
  Open --> Blocked : credit / delivery / billing block
  Blocked --> Open : released
  Open --> PartiallyProcessed : partly referenced
  PartiallyProcessed --> Completed : fully referenced
  Open --> Completed : fully referenced or all lines rejected
  Completed --> PartiallyProcessed : subsequent doc cancelled
  Open --> Expired : quotation valid-to passed
  Completed --> [*]
```

Status **dimensions** are kept as separate columns. For example, a Sales Order header carries: `OverallStatus`, `DeliveryStatus`, `BillingStatus`, `CreditStatus`, `IncompletionStatus` and `BlockStatus`. Each dimension uses the values `NotRelevant / NotProcessed / Partial / Complete`. `OverallStatus` is calculated from them (deviation D-09).

---

## 13. RACI (R = responsible, A = accountable, C = consulted, I = informed)

| Activity | SA | SV | WH | BF | AD | SYS |
|---|---|---|---|---|---|---|
| Maintain org structure & configuration | I | C | I | C | **A/R** | — |
| Maintain customer / item / price master | R | A | I | C | C | validates |
| Credit limit maintenance | I | C | | **A/R** | | validates |
| Inquiry / Quotation | **R** | A | | | | prices |
| Discount above limit approval | R | **A** | | | | enforces |
| Sales Order entry & change | **R** | A | I | | | ATP / price / credit |
| Credit block release | I | **A/R** | I | C | | re-checks |
| Delivery creation, picking, packing, GI | I | | **A/R** | I | | stock posting |
| Billing, cancellation | I | | | **A/R** | | AR message |
| Returns order & approval | R | **A** | R (GR) | R (CRN) | | |
| Credit / debit note request | R | **A** | | R | | |
| AR interface monitoring | | | | **A/R** | C | retries |
| Users & roles | | C | | | **A/R** | audits |

---

## 14. Traceability seed (used by BRD / FSD / UAT)

| Process | Flow ID | Source phase |
|---|---|---|
| Master data & config | P01 | Phase 1 |
| Pre-sales | P02 | Phase 2 |
| Sales order, pricing, ATP, credit | P03 | Phase 2 |
| Delivery, pick, pack, GI | P04 | Phase 3 |
| Billing, cancellation, AR interface | P05 | Phase 4 |
| Returns | P06 | Phases 3–4 |
| Credit / debit memo | P07 | Phase 4 |
| Reports & worklists | P08 (due lists, backlog, doc flow, analysis) | Phase 5 |

---

**Approval requested:** please confirm or correct the flows above and answer the open questions in `00-assumptions-and-open-questions.md` §3 (or accept the defaults). After approval, Deliverable 2 (BRD summary with the BR-xx list) follows.
