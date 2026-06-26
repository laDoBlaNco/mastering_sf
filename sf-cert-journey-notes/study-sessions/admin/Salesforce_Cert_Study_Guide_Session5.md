# Session 5 — Product / Pricebook / OpportunityLineItem

> **Cert Track:** Platform Administrator I → Nonprofit Cloud Consultant → Platform App Builder → Platform Developer I → Agentforce Specialist → JavaScript Developer I → Platform Developer II → Data Architect → Sharing & Visibility Architect → ***Salesforce Certified Application Architect***

> **Salesforce Admin Cert — Study Guide Series**
> Captured deep-dive session: concept explanations, mental-model clarifications, and a worked quiz with full answer rationale. Read the concepts, then test yourself against the quiz section before reading the graded answers.

**What this guide covers**
- The problem this structure solves (vs a flat Amount field)
- The four objects: Product, Pricebook, PricebookEntry, OpportunityLineItem
- PricebookEntry as a junction; the Standard Pricebook requirement
- The pricing-copy mechanic (no cascade to existing line items)
- Opportunity Amount auto-rollup
- BPO end-to-end example + quirks
- Quiz: 6 real-world scenarios with full rationale

---

## 1. The Problem This Structure Solves

A naive design puts "Amount" and "Description" straight on the Opportunity. It falls apart:
- Can't report on units of Product X sold this year (no Product record)
- Can't separate "100 hrs Discovery @ $200" from "50 hrs Implementation @ $400" (one Amount)
- Can't model bundles, tiered pricing, or currency variations
- Can't feed CPQ, billing, or revenue recognition (no structured data)

Salesforce solves this with a four-object structure that's elegant once you see it.

---

## 2. The Four Objects

```
Product (Product2)        Pricebook (Pricebook2)
   what we sell      +       a price list / context
        \                          /
         \                        /
          --> PricebookEntry <--   (price of a Product in a Pricebook)
                    |
                    v
          OpportunityLineItem  <-- Opportunity
   (a Product at a price, in a quantity, on a deal)
```

### Product (Product2)
- An item/service in your catalog. Legacy API name Product2 (old "Product" object is reserved).
- Key fields: Name, ProductCode (SKU), Family (categorization for reporting), IsActive (soft delete).
- **Has NO price** — that's the architectural key. Price lives on the Pricebook side.

### Pricebook (Pricebook2)
- A price list / context. Every org gets one **Standard Pricebook** automatically — the master catalog of all products and their default prices.
- Custom Pricebooks model contexts: Enterprise, EMEA, Government, Partner, Pilot.
- Key fields: Name, IsStandard, IsActive.

### PricebookEntry — the junction
- Answers "what's the price of THIS Product in THIS Pricebook?" — a junction between Product and Pricebook.
- Same Product in three Pricebooks = three PricebookEntries, each with its own UnitPrice.
- Key fields: Pricebook2Id, Product2Id, UnitPrice, CurrencyIsoCode, IsActive, UseStandardPrice.
- Quirk: under the hood it uses Lookups, not strict Master-Details, but functions as a junction.
- **Standard Pricebook requirement:** a Product must have a Standard Pricebook entry before it can have any Custom Pricebook entries. Custom Pricebooks are essentially overrides of the master list.

### OpportunityLineItem — what's in this deal
- A specific Product, at a specific price, in a specific quantity, on a specific Opportunity.
- OpportunityLineItem→Opportunity is a true **Master-Detail** (cascade delete with the deal).
- OpportunityLineItem→PricebookEntry is a **Lookup** (so a line item survives if a PBE is archived).
- Key fields: OpportunityId, PricebookEntryId, Quantity, UnitPrice (overridable), TotalPrice, Discount, ServiceDate.

> **Opportunity Amount auto-rollup**
> When an Opportunity has line items, its Amount is automatically the SUM of all OpportunityLineItem TotalPrice values — a built-in rollup, no config needed. With no line items, Amount is whatever was entered manually.

---

## 3. BPO End-to-End Flow

- **Catalog setup (once):** create Products (e.g. "Contact Center Tier 1", "QA Monitoring", "Implementation Services") — each auto-creates a Standard Pricebook entry.
- **Pricing strategy:** create Custom Pricebooks (Enterprise, SMB, Pilot) and manually add PricebookEntries with the right price per book.
- **Opportunity creation:** a Globex (enterprise) deal — rep assigns the Enterprise Pricebook. One Pricebook per Opportunity.
- **Add products:** Salesforce shows only Products with an Active PricebookEntry in that Pricebook; rep sets quantities and may override price for negotiated discounts.
- **Line items created:** each becomes an OpportunityLineItem; Amount auto-sums.
- **Reporting unlocked:** revenue by Product Family, top products by region, average discount on a service — none possible with a flat Amount field.

### Why it matters for BPO
Set up Product/Pricebook correctly from the start and CPQ, Revenue Cloud/billing, subscription management, and contract management all come "for free" downstream. Skip it and retrofitting in a live org with thousands of Opportunities is brutal — the textbook "understanding standard architecture prevents spaghetti" case.

### Quirks worth knowing
- **Currency precision:** UnitPrice defaults to 2 decimals — set higher precision BEFORE creating PBEs if you sell sub-cent units.
- **Active cascade:** deactivating a Product doesn't deactivate its PBEs — handle both.
- **Standard Pricebook trap:** if Custom Pricebooks aren't activated for the team, everyone defaults to Standard pricing.
- **Multi-currency explosion:** a PBE per currency per pricebook per product multiplies fast (10 products × 5 books × 3 currencies = 150 entries).
- **Line-item parallels:** Quotes, Orders, and Contracts each have their own child line-item object (QuoteLineItem, OrderItem, ContractLineItem) — same pattern, separate tables, so each stage's data stays immutable.

---

## 4. Quiz — 6 Real-World Scenarios

### Q1 · The new product launch
"AI-Powered Sentiment Analysis" — $50/hr Enterprise, $65/hr SMB, $40/hr Pilot.
**Answer:** (1) Create the **Product** — this auto-creates its **Standard Pricebook** entry with a standard/list price. (2) Ensure Custom Pricebooks exist (Enterprise, SMB, Pilot). (3) Create a **PricebookEntry** in each at the right price. (4) It's now sellable as line items. **Correction from the quiz:** the Standard Pricebook is NOT "the SMB pricebook" — it's the prerequisite master catalog; SMB/Enterprise/Pilot are all Custom Pricebooks. **Result: right sequence, misidentified Standard Pricebook.**

### Q2 · The pricing update
Enterprise Contact Center Tier 1 rises $175 → $190 next quarter; 400 open Opps use $175.
**Answer (critical mechanic):** Editing a PricebookEntry does **NOT** cascade to existing OpportunityLineItems. When a product is added to an Opp, the price is **copied** onto the line item at that moment; the line item and PBE are independent thereafter (like a price tag already in your cart). So: update the PBE → affects only NEW line items. Existing Opps stay at $175 unless manually re-priced (a business decision; bulk update if needed). **Risk if done wrong:** if you assume a cascade, reports show $190 on deals quoted at $175. **Result: this was the main gap — the "no cascade" rule is counterintuitive and exam-likely.**

### Q3 · The mid-deal Pricebook switch
An Opp on the SMB Pricebook has 3 line items; client now wants Enterprise pricing.
**Answer:** You can't cleanly "just switch" — existing line items reference SMB PricebookEntries and Salesforce won't allow a mismatch. Simplest path: delete the line items, switch the Pricebook, re-add the products at Enterprise prices. Salesforce DOES warn you that switching removes existing products. Mature orgs snapshot the line items first. **Result: correct approach and reasoning.**

### Q4 · The reporting request
Revenue by Product Family from Closed Won (12 mo), broken down by Pricebook used.
**Answer:** Yes — possible with standard reporting. **Data path:** Opportunity → OpportunityLineItem → PricebookEntry → (Pricebook for context) & (Product → Product Family). Use the built-in **"Opportunities with Products"** report type. **Gotcha:** Opps with no line items (manual Amount only) won't show product data, understating revenue in mixed orgs. **Result: right answer; path needed OpportunityLineItem inserted, not Product→Pricebook directly.**

### Q5 · The discount problem
Rep overrides UnitPrice $250 → $200 on a line item; Ops Manager fears it "breaks the Pricebook."
**Answer:** Ops Manager is wrong — overriding UnitPrice affects only that line item; the PricebookEntry stays $250 for everyone else. Better practice is to use the **Discount %** field so the discount is explicit and reportable (avg discount by product/rep/segment). A **Discount Approval** process (Flow + Approvals) can gate discounts over a threshold — no code. **Result: 5/5, cleanest answer.**

### Q6 · The architectural decision
A leader wants to scrap Products/Pricebooks for a simple custom "Service Line" object.
**Answer:** Possible, simpler for reps, but you regress toward a flat Amount/description and **lose:** CPQ compatibility, Revenue Cloud/billing integration, native product-level forecasting, and the AppExchange ecosystem that expects standard Product/Pricebook. **Defensible when:** very early stage, no pricing complexity, no CPQ/billing plans, or as a throwaway proof-of-concept. Best line: "It works until it doesn't — the moment you want CPQ or billing you'll be migrating thousands of records." **Result: right trade-offs; add the downstream-integration argument.**

> **Two things to lock in**
> (1) PricebookEntry edits do NOT cascade to existing OpportunityLineItems — price is copied at add-time and is independent thereafter. (2) The Standard Pricebook is the prerequisite master catalog, not a pricing tier; a Product needs a Standard entry before any Custom Pricebook entry. Both are exam-likely.
