# SUQ — Qatar Hospitality & Retail Operating System

**Working name:** SUQ (سوق) — one platform, two editions: **SUQ Table** (restaurants, cafés, cloud kitchens) and **SUQ Shelf** (supermarkets, groceries, cold stores).

**Positioning:** Every POS sold in Qatar today is a Dubai or Riyadh product with `AED` swapped for `QAR`. The gap is not the currency — it is that Qatar's regulatory stack (no VAT yet, excise only, e-invoicing arriving, WPS payroll, Baladiya health cards, plastic-bag law), its payment rails (Himyan, NAPS, Fawran, QPay, QMP), its delivery market (Snoonu, Rafeeq, Talabat) and its workforce (Arabic + Malayalam + Hindi + Tagalog + Nepali on the same shift) are unlike anywhere else. SUQ is built Qatar-first and exports outward, not the reverse.

---

## 1. Why Qatar is genuinely a different market

### 1.1 The tax position (the single biggest design driver)

Qatar is the odd one out in the GCC. As of August 2026:

| Item | Status | Design consequence |
|---|---|---|
| VAT | **Not implemented.** GCC Framework signed; 5% expected; no enacted law or confirmed date | Ship with a **dormant tax engine**: rates, groups, and tax-invoice layouts already exist, set to 0% / "Not applicable". Flip by config, not by code. |
| E-invoicing | Draft law + implementing regulations approved by Cabinet **6 May 2026**. Peppol-style architecture with accredited service providers and structured XML widely expected; phased rollout speculated from 2027, large taxpayers first | Model every sale as a **structured invoice document** from day one (UBL-shaped fields, sequential numbering, buyer TRN slot, QR payload slot, 5-year immutable archive). Retrofitting this into a POS that only stores receipt text is a rewrite. |
| Excise Tax | **Live since 2019** (Law No. 25 of 2018): tobacco 100%, energy drinks 100%, carbonated drinks 50%, special-purpose goods 100% | Excise is levied at **import/production**, not at the till. It must appear in **landed cost and margin**, never as a POS add-on line. Most imported systems get this wrong and either double-tax or ignore it. |
| Corporate income tax | 10% on foreign-owned profit share; Qatari/GCC-owned entities generally exempt; filed via **Dhareeba** | Chart of accounts and year-end pack should export to Dhareeba-friendly formats. |
| Customs | GCC common external tariff (5% on most goods; higher/prohibited categories exist) | Landed-cost engine must apportion duty, freight, clearing, and demurrage across a shipment. |

**The thesis:** be the system that is *already* compliant on the day VAT is announced. Sell "VAT-ready" as the headline feature for the next 12–24 months. That is a Qatar-only sales pitch and it has a deadline attached.

### 1.2 Payments — Qatar's rails are national, not international

Qatar Central Bank operates its own retail payment infrastructure. A till here must speak:

- **NAPS / Himyan** — the national ATM & POS network and Qatar's domestic card scheme. Himyan is being pushed hard (government fee collection moved to it). Any POS that treats "debit" as a Visa/Mastercard variant is mispricing interchange and mis-reconciling settlement.
- **QPay** — the national e-commerce gateway for online/card-not-present.
- **Fawran** — QCB instant payment, 24/7, request-to-pay with a ~6-hour expiry, proxy by mobile number. Fees are per-transaction and tiny (roughly QAR 0.50 under QAR 1,000, ~QAR 2 above). **This is the cheapest tender in the country** and no POS makes it easy at the counter.
- **QMP** — Qatar Mobile Payment wallets.
- **Apple Pay / Google Pay** via local acquirers (Dibsy and others); **UPI** acceptance at QNB merchants — material given the size of the Indian community.
- Bank/PSP layer: QNB, CBQ, QIB, Doha Bank, Dukhan; PSPs including Dibsy, Skipcash, Fatora, Sadad.

**Product opportunity:** a *tender-cost optimiser*. Show the owner, per shift, what each tender actually cost them (card MDR vs Fawran flat fee vs cash handling) and how much a nudge toward Fawran saves per month. Nobody in this market does that.

### 1.3 Labour and workforce — this is where restaurants actually bleed

- **Wage Protection System (WPS)** is mandatory: salaries paid through Qatari banks, submitted as a structured **SIF file** with QID, IBAN, basic, allowances, deductions. Getting the file rejected costs businesses their MoL standing.
- **QID / work permit / residence permit expiry** — a lapsed QID stops an employee working. Restaurants track this on paper.
- **Baladiya (municipality) health cards** for every food handler, renewed annually with a medical. An expired card during a municipal inspection can close a kitchen.
- **Shift language reality:** the same shift runs Arabic, English, Malayalam, Hindi, Tagalog, Nepali, Urdu, Bengali. Manager-facing screens are English/Arabic; **staff-facing screens should be in the staff's own language**.

**Product opportunity:** a compliance dock that turns "your visa expires in 21 days" into a rota constraint, not an HR surprise. Payroll produces a validated WPS SIF file. This alone will sell the product.

### 1.4 The Qatari calendar and daily rhythm

- **Weekend is Friday–Saturday.** Friday is a half-day for most food businesses; many close for Jumu'ah (roughly 11:00–13:00).
- **Five daily prayers** shift by minutes each day. Malls and many outlets pause service at Athan. A rota engine that doesn't know prayer times will schedule breaks wrong every single day.
- **Ramadan** rewrites the entire business: no daytime dine-in service for fasting customers, restaurants closed or takeaway-only until Maghrib, then a massive iftar peak in a two-hour window, then suhoor trade until ~2–3 AM. Supermarkets see demand spike *before* Ramadan and again before Eid. Menus, prices, staffing, and opening hours all change for 30 days.
- **Fixed demand spikes:** National Day (18 December), National Sports Day (second Tuesday of February), Eid al-Fitr, Eid al-Adha, school terms, summer exodus (June–August, when a large share of residents leave the country and covers collapse).

**Product opportunity:** a **Hijri-aware forecasting engine**. Ramadan moves ~11 days earlier each Gregorian year, so a demand model trained on "last year, same week" is wrong by design. Forecast on the Hijri calendar and you beat every imported system in the market.

### 1.5 Category rules unique to Qatar

- **No alcohol, no pork** in general retail or restaurants — restricted to QDC-licensed premises and hotel outlets. Product master needs a permission model, not just a flag.
- **Halal certification** must be traceable to supplier lot for meat and poultry.
- **Single-use plastic bags are banned** — carrier bags are a chargeable, tracked SKU, and bag sales are an audit line.
- **MoCI consumer protection:** shelf price must equal till price (mismatch = fine), Arabic labelling, mandatory receipts, and periodic price-monitoring of basic commodities, especially in Ramadan.
- **Import dependency:** almost everything arrives via Hamad Port or the Abu Samra land border. Shelf life is consumed in transit, so expiry management must start at goods-receipt, not at the shelf.
- **Local sourcing preference:** Baladna, Widam, Qatar Farms and the "Made in Qatar" push — a `country_of_origin` field is not cosmetic, it drives merchandising and procurement policy.

---

## 2. Product architecture

### 2.1 Editions from one core

```
                      ┌─────────────────────────────┐
                      │        SUQ CORE             │
                      │  identity · tax engine ·    │
                      │  payments · compliance ·    │
                      │  inventory · reporting      │
                      └──────────┬──────────────────┘
                 ┌───────────────┴───────────────┐
        ┌────────▼─────────┐          ┌──────────▼────────┐
        │   SUQ TABLE      │          │    SUQ SHELF      │
        │  restaurants     │          │  supermarkets     │
        │  · table/floor   │          │  · scale + PLU    │
        │  · KDS + course  │          │  · self-checkout  │
        │  · recipe cost   │          │  · planogram      │
        │  · aggregators   │          │  · promo engine   │
        └──────────────────┘          └───────────────────┘
```

The two editions share ~65% of the codebase. Do **not** fork them — a supermarket with a hot-food counter and a restaurant with a retail fridge are the same customer in Qatar, and multi-format groups (e.g. a group running a café, a cloud kitchen, and a mini-market) are the highest-value accounts.

### 2.2 Module map

**Core (both editions)**
1. **Tax & invoicing engine** — tax groups, dormant VAT, excise-at-cost, sequential invoice numbering, credit notes, e-invoice document store, QR payload builder, 5-year archive.
2. **Payments & settlement** — tender types per rail, split tenders, tips, Fawran request-to-pay, end-of-day settlement reconciliation against acquirer files, tender-cost analytics.
3. **Identity & roles** — cashier PIN, manager override, per-user permission grid, full audit trail (voids, discounts, no-sale, price overrides — the four fraud vectors).
4. **Inventory & procurement** — purchase orders, goods receipt with temperature and expiry capture, landed-cost apportionment (freight + duty + clearing + excise), stock transfers between branches, cycle counts, wastage with reason codes.
5. **Supplier & compliance registry** — trade licence, CR number, halal certificate expiry, food-safety certificates, blocked-supplier list.
6. **Workforce** — rota, clock-in (PIN/biometric/QR), overtime, WPS SIF export, document expiry watchtower (QID, work permit, health card, passport), multilingual staff UI.
7. **Reporting & BI** — Z-report, X-report, hourly heatmap, basket analysis, shrinkage, margin by category, Hijri-aware comparatives.
8. **Bilingual layer** — full RTL, Arabic-first receipts, Arabic product names as a first-class field (not a translation table).
9. **Offline-first sync** — Qatar has excellent connectivity, but a till must never stop. Local write-ahead log, conflict-free sync on reconnect.

**SUQ Table (restaurants)**
10. Floor plan, table states, covers, split/merge bills, course firing.
11. KDS (kitchen display) with station routing, prep timers, allergen and "no-onion"-style modifier propagation, Arabic + staff-language ticket printing.
12. Recipe & sub-recipe costing, yield, theoretical vs actual food cost, live plate margin.
13. Aggregator hub — **Talabat, Snoonu, Rafeeq**, plus own-brand ordering. One menu master, per-channel pricing (aggregators take 25–30%, so channel-specific pricing is survival), auto-accept rules, and a single reconciliation ledger for aggregator payouts.
14. Shisha service module where licensed — session timing, coal rounds, per-head charging. Real, common, and unsupported by imported systems.
15. Ramadan mode — iftar/suhoor menus, pre-booked iftar covers, timed auto-switch at Maghrib, buffet headcount.

**SUQ Shelf (supermarkets)**
16. Scale integration (weighed PLU barcodes), butchery/bakery/hot-counter production.
17. Promotions engine — multibuy, mix-and-match, timed markdown for near-expiry, loyalty-only pricing, Ramadan basket bundles.
18. Shelf-edge label printing, bilingual, with a **price-integrity check** that blocks a promo going live if the shelf label queue hasn't printed (directly prevents an MoCI shelf-vs-till fine).
19. Expiry & batch management from goods receipt, near-expiry markdown workflow, waste write-off with photo evidence.
20. Self-checkout / scan-and-go, carrier-bag charge prompt.
21. Cold-chain log — fridge/freezer temperature rounds with timestamps, exportable for MoPH inspection.

### 2.3 Technology stack (aligned to how you already build)

| Layer | Choice | Why |
|---|---|---|
| Backend | **Node.js + Express** | Matches your existing SIB CRM workflow; large local hiring pool. |
| Database | **PostgreSQL** (not SQLite) | Multi-branch, concurrent tills, row-level security, JSONB for e-invoice documents. Keep SQLite only as the **on-till offline store**. |
| Till app | **React + Vite + Tailwind**, packaged as PWA and Electron | One codebase for Android tills, Windows tills, and iPad. |
| KDS | React, full-screen PWA, touch-optimised, no keyboard | Runs on a cheap Android tablet in a hot kitchen. |
| Sync | Local WAL + idempotent event replay | A till must survive a router reboot mid-shift. |
| Hosting | Railway/Fly for pilot → **Qatar-resident hosting** (Ooredoo Cloud, MEEZA, or AWS Bahrain with a Doha edge) before enterprise sales | Data residency will come up in every serious procurement conversation. |
| Reporting | Materialised views + a nightly rollup job | Owners open reports on a phone; queries must be pre-aggregated. |
| i18n | ICU message format, Arabic as a peer locale | Arabic is not an afterthought translation file. |

### 2.4 Core data model (the parts that are Qatar-specific)

```sql
-- Products carry Arabic as a peer field, not a translation
product (
  id, sku, barcode,
  name_en, name_ar,                 -- both required
  category_id, uom, is_weighed,
  country_of_origin,                -- drives "Made in Qatar" merchandising
  restriction_class,                -- none | alcohol | pork | pharmacy | tobacco
  halal_required boolean,
  excise_class,                     -- none | tobacco_100 | energy_100 | carbonated_50 | special_100
  tax_group_id,                     -- points at the dormant VAT engine
  shelf_life_days, storage_temp_band
)

-- Tax is a versioned config, never a hardcoded rate
tax_group (id, code, name_en, name_ar)
tax_rate  (tax_group_id, rate_pct, effective_from, effective_to, legal_reference)
-- Today every row is 0.00 with legal_reference = 'No VAT enacted in Qatar as at <date>'
-- VAT day = one INSERT per group, effective_from = go-live. No deployment.

-- Excise lives in cost, not at the till
purchase_line (
  po_id, product_id, qty, unit_cost_fob, freight_alloc,
  customs_duty_alloc, excise_amount, clearing_alloc,
  landed_unit_cost   -- generated
)

-- Every sale is a structured document, ready for e-invoicing
invoice (
  id, branch_id, series, sequence_no,       -- gapless per series
  issued_at, buyer_name, buyer_trn,         -- TRN slot ready
  currency 'QAR', subtotal, tax_total, rounding_adj, grand_total,
  einvoice_status,                          -- not_applicable | queued | cleared | rejected
  einvoice_uuid, einvoice_hash, qr_payload,
  document_json                             -- UBL-shaped payload
)

-- Employees: the compliance fields are the product
employee (
  id, name_en, name_ar, qid, qid_expiry,
  work_permit_expiry, health_card_no, health_card_expiry,
  passport_expiry, iban, bank_code,
  basic_salary, allowances, ui_locale        -- en|ar|hi|ml|tl|ne|ur|bn
)
```

**Rules baked into the engine:**
- Cash tenders round to the nearest **0.25 QAR** (configurable 0.05 / 0.25 / 0.50); non-cash never rounds. The rounding adjustment is its own posted line, not a silent adjustment.
- Excise **never** appears as a till line. It appears in landed cost and in the margin report.
- A price change cannot go live until its shelf label is queued for print.
- Voids, discounts above threshold, price overrides, and no-sales always require a manager PIN and always write an audit row.

---

## 3. Signature features — the ones competitors can't copy quickly

1. **VAT Day switch.** One config change turns the whole estate VAT-compliant: rates activate, receipts become tax invoices, buyer TRN capture appears, returns filing pack generates. Marketed as an insurance policy.
2. **Prayer-aware operations.** Prayer times drive break rotas, kitchen firing, drive-thru staffing, and an optional soft service-hold at Athan. Jumu'ah handled as a first-class weekly event.
3. **Hijri demand forecasting.** Ramadan and Eid comparatives that actually align year over year, plus a pre-Ramadan procurement planner that tells a supermarket what to order six weeks out.
4. **WPS-ready payroll.** Validated SIF file, IBAN and QID checks before submission, rejection-reason decoder.
5. **Document expiry watchtower.** QID, work permit, health card, trade licence, halal certificate, civil defence certificate — one board, escalating alerts, and rota blocking when a document lapses.
6. **Tender-cost optimiser.** Real cost per tender rail, with a Fawran nudge that pays for the subscription.
7. **Aggregator P&L truth.** Talabat/Snoonu/Rafeeq orders reconciled to actual payouts, with commission, promotion funding, and cancellation losses stripped out so the owner sees the real channel margin — not gross order value.
8. **Bilingual by construction.** Arabic and English receipts, Arabic-first customer display, staff screens in the language the staff member actually speaks.
9. **Inspection Mode.** One tap prints the pack a Baladiya or MoPH inspector asks for: health cards, temperature logs, supplier halal certificates, waste records, pest control. Turns a stressful hour into a button.

---

## 4. Delivery roadmap

**Phase 0 — Design partners (weeks 1–4).** Sign three pilots: one independent restaurant, one 3-branch café group, one neighbourhood supermarket. Free for 6 months in exchange for access and honest feedback. Do not build before this.

**Phase 1 — Till + core (weeks 5–14).** Offline-first till, product master with Arabic, dormant tax engine, tenders incl. Fawran, Z/X reports, receipt printing, cash management, audit trail. Restaurant floor plan and supermarket scale support both land here.

**Phase 2 — Back of house (weeks 15–24).** Purchasing, goods receipt with expiry and temperature, landed cost with excise, recipe costing, wastage, stock counts, multi-branch transfers.

**Phase 3 — People & compliance (weeks 25–32).** Rota with prayer times, clock-in, document watchtower, WPS export, Inspection Mode.

**Phase 4 — Channels & intelligence (weeks 33–44).** Aggregator hub, own-brand ordering, loyalty, promotions engine, Hijri forecasting, tender-cost optimiser.

**Phase 5 — E-invoicing (triggered by regulation).** Build the connector when the GTA publishes the technical specification and the accredited-provider list. Until then, keep the document model and archive ready and say so publicly.

---

## 5. Commercial model

| Tier | Fit | QAR / month per branch |
|---|---|---|
| Starter | 1 till, café or small grocery | 249 |
| Growth | up to 4 tills, KDS, inventory | 649 |
| Multi-branch | unlimited tills, consolidated reporting, aggregator hub | 1,199 |
| Enterprise | 10+ branches, on-prem/Qatar-resident hosting, API, SLA | custom |

Add-ons: payroll + WPS (QAR 12/employee/month), self-checkout lane, scale integration, hardware bundles.

**Go to market:** Doha is small and relationship-driven. Sell through (a) accounting and audit firms serving SMEs, who will need to migrate their clients for VAT anyway; (b) POS hardware resellers in Salwa Road and the Industrial Area; (c) the aggregators themselves, who benefit from integrated merchants; (d) direct walk-in to Al Sadd, Bin Mahmoud, Al Wakra and Barwa Village independents. Arabic-language marketing is not optional.

**Moat:** every feature above is regulatory or cultural, and each one is a month of work for a competitor who doesn't live here. The compounding asset is the compliance ruleset kept current.

---

## 6. Open questions to verify before writing production code

These are current-state items that move, and they should be confirmed with the source authority rather than assumed:

1. GTA's published e-invoicing technical specification, accredited service provider list, and phase-1 taxpayer thresholds.
2. Whether VAT legislation has been enacted, its rate, registration threshold, and zero-rated/exempt schedules for food.
3. Current excise rates and the full special-purpose goods schedule.
4. Applicable customs duty rates by HS code for your clients' main import categories.
5. Current WPS SIF file specification from the Ministry of Labour / QCB and the acquiring banks' variants.
6. Fawran merchant onboarding and current per-transaction pricing by acquirer.
7. Aggregator API access terms and commission structures for Talabat, Snoonu, and Rafeeq.
8. MoCI rules on price display, receipts, and any commodity price-monitoring lists.
9. Baladiya health-card and food-safety documentation requirements by municipality.
10. Data residency expectations for hosted business records.

---

## 7. What to build first, concretely

If you want one thing running this month, build the **tax and invoice engine plus the till**, because everything else attaches to it and because the "VAT-ready" story is the reason a Qatari owner switches systems before they have to. The prototype accompanying this document demonstrates that layer: bilingual till, dormant VAT, embedded excise, Qatar tender rails, cash rounding, carrier-bag levy, prayer-aware service, and the compliance dock.
