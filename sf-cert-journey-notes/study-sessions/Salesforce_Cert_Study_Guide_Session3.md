# Session 3 — Standard Object Architecture & Relationship Infrastructure

> **Cert Track:** Platform Administrator I → Platform App Builder → Platform Developer I → Agentforce Specialist → JavaScript Developer I → Platform Developer II → Data Architect → Sharing & Visibility Architect → ***Salesforce Certified Application Architect***

> **Salesforce Admin Cert — Study Guide Series**
> Captured deep-dive session: concept explanations, mental-model clarifications, and a worked quiz with full answer rationale. Read the concepts, then test yourself against the quiz section before reading the graded answers.

**What this guide covers**
- The customer-lifecycle spine: Lead → Account / Contact / Opportunity
- Account as the gravitational hub of the entire data model
- Deep dives on Account, Contact, Lead, and Opportunity
- Relationship infrastructure: Lookup, Master-Detail, Junction, Hierarchical, Self-Lookup
- Junction objects mapped to RDBMS join tables (PK / FK mechanics)
- Does the architecture change across clouds? (Sales vs Service vs Data 360)
- Two quizzes: 8-scenario comprehension check + 5-scenario hard round

---

## 1. The Core Mental Model: A Spine Built on the Customer Lifecycle

Salesforce's standard architecture is organized around a customer lifecycle, and almost everything connects back to one hub object — the **Account**. Internalize that and you can reason about the model rather than memorize it.

The cleanest way to understand the core objects is to follow a prospect through the journey:

- **Unqualified interest** — a form fill, a trade-show scan, a purchased list entry → **Lead**
- **Qualified** — convert the Lead, which creates an **Account** (company), a **Contact** (person), and optionally an **Opportunity** (the deal)
- **Active pursuit** — the Opportunity moves through stages with an amount and close date
- **Closed** — Won or Lost; the Account persists and spawns future Opportunities (renewals, expansions, new lines of business)

### The Five Architectural Layers

Think in layers, not a flat list of objects:

- **Layer 1 — Core spine:** Account, Contact, Opportunity, Lead, Activity. Used by every cloud.
- **Layer 2 — Sales execution:** Product, Pricebook, OpportunityLineItem, Quote, Campaign / Campaign Member, Opportunity Contact Role.
- **Layer 3 — Service:** Case, Knowledge, Entitlement, Work Order (Service Cloud's layer on top).
- **Layer 4 — Relationship infrastructure:** Lookup, Master-Detail, Junction, Hierarchical — where most orgs go wrong.
- **Layer 5 — Platform:** the metadata-driven foundation (fields, layouts, validation rules, flows, sharing). You configure metadata, never the raw database.

> **Single most important idea**
> Account is the durable entity. People change jobs (Contacts come and go), deals open and close (Opportunities), support issues open and close (Cases) — but the company persists. Every meaningful question about a customer relationship is answered by looking at the Account and the children hanging off it.

---

## 2. The Core Objects in Depth

### Account — The Hub
A company or organization. The center of everything because it is the durable entity.

- **Account Hierarchies** via the ParentId field — models global parent + regional subsidiaries, or holding companies. (Technically a self-Lookup, not the Hierarchical type.)
- **Business vs Person Accounts** — standard Account is B2B; Person Accounts merge Account+Contact for B2C. Person Accounts are a one-way door once enabled.
- **Account Teams** — multiple Users on one Account with different roles (AE, CSM, SE). Critical for BPO where many internal teams touch one client.
- **Where fields go:** data true about the company and stable across deals — Industry, Employee Count, Annual Revenue, HQ, Parent Company.

### Contact — The People
- Account→Contact is a **Lookup** (not Master-Detail). A Contact can technically exist without an Account, though orphan Contacts are an anti-pattern.
- **Contacts to Multiple Accounts** — one person related to multiple Accounts with different roles (consultants, board members, partners).
- **Opportunity Contact Roles** — associate a Contact to a deal as Economic Buyer, Decision Maker, Champion, Influencer. Underused and powerful.
- **Where fields go:** data about the person — Title, Email, Phone, Reports-To, persona.

### Lead — The Pre-Qualification Holding Pen
A prospect not yet qualified, deliberately separate from the Account/Contact world so messy unverified data never pollutes the real customer database.

- **Denormalized:** holds both person fields (Name, Email, Title) and company fields (Company, Industry, Employees) flat on one row.
- **Conversion** splits those fields: person → Contact, company → Account, plus an optional Opportunity. The Lead becomes read-only after conversion.
- **Field mapping gotcha:** custom Lead fields do NOT auto-carry to Account/Contact/Opportunity — you must map them in Setup or data is lost at conversion.
- **Anti-pattern:** Leads that live forever because nobody converts or disqualifies them.

### Opportunity — The Revenue Event
- Always related to an Account (Lookup). Multiple Opportunities per Account is the NORM — initial sale, renewal, upsell, new LOB each get their own Opportunity.
- **Stages → Forecast Categories → Probability** drive forecasting.
- **Opportunity Line Items** build the Amount from Products in a Pricebook (covered in Session 5).
- **Where fields go:** data specific to THIS deal — Amount, Close Date, Stage, products, contract length, win/loss reason.

**How the relationships actually connect:**

```
Account (1) ---< (many) Contact          [Lookup]
Account (1) ---< (many) Opportunity      [Lookup]
Account (1) ---< (many) Case             [Lookup, Service]
Opportunity (m) >--< (m) Contact   [via OpportunityContactRole junction]
Lead          standalone - no relationships until conversion
```

---

## 3. The Relationship Infrastructure

Relationship choice is the single biggest source of spaghetti-org decisions. There are five types you'll actually use.

### 1 · Lookup
**RDBMS equivalent:** a nullable foreign key with no cascade (ON DELETE SET NULL).
- Link is optional; the two records have independent lifecycles.
- Deleting the parent **clears** the child's lookup by default (configurable: clear, prevent deletion, or cascade delete).
- Sharing/security on the two objects are **independent**.
- **No rollup summary fields** natively (DLRS or Flow can fill the gap).
- **Use when:** the child is meaningful without the parent. Account→Contact, Account→Opportunity.

### 2 · Master-Detail
**RDBMS equivalent:** required FK with ON DELETE CASCADE plus inherited ownership/permissions.
- Parent is **required**; cascade delete; child has **no own Owner** (inherits parent's sharing).
- **Rollup summary fields enabled** on the parent (SUM/COUNT/MIN/MAX).
- Max **two** Master-Detail relationships per object (this is what makes junctions work).
- Hard/impossible to convert back once it holds data.
- **Use when:** the child has no meaning without the parent. Opportunity→OpportunityLineItem.

### 3 · Many-to-Many (Junction Object)
**RDBMS equivalent:** a join table.
- A custom object with **two Master-Detail** fields, one to each parent.
- Holds data about the relationship itself (e.g., a proficiency rating, a role, a price).
- Standard examples: CampaignMember, OpportunityContactRole, OpportunityLineItem.
- **First Master-Detail = primary**: controls ownership, sharing inheritance, report color, and which parent the junction sits under by default.

### 4 · Hierarchical
**RDBMS equivalent:** a self-referencing nullable FK.
- A special self-lookup reserved for the **User object only** (the ManagerId field).
- Built-in circular-reference protection enforces a true tree.
- Powers approval routing, sharing by management chain, forecasting hierarchies.
- Account's ParentId is functionally similar but is classified as a plain Lookup, not Hierarchical.

### 5 · Self-Lookup (custom hierarchies)
- A Lookup field on a custom object pointing to itself — models category/sub-category or position trees. The Hierarchical TYPE is User-only; everywhere else you use a self-Lookup.

**The decision tree:**
- Child meaningful if parent disappears?  Yes → Lookup.  No → Master-Detail.
- Many-to-many?  → Junction (two Master-Details).
- Object relates to itself in a tree?  User reporting → Hierarchical; custom → Self-Lookup.
- Need a rollup on the parent?  Tilts toward Master-Detail (but DLRS/Flow soften this).
- Parent and child need DIFFERENT owners/sharing?  Must be Lookup.

---

## 4. Junction Objects and the RDBMS Mapping, Visually

A many-to-many between Opportunity and Contact needs three tables in an RDBMS, and three objects in Salesforce. The middle one is the join table / junction.

```
Opportunity (1) ---< OpportunityContactRole >--- (1) Contact
   PK: Id            PK: Id (surrogate)          PK: Id
                     FK: OpportunityId (M-D #1)
                     FK: ContactId     (M-D #2)
                     + Role, IsPrimary, CreatedDate
```

### Primary key mechanics
- **RDBMS option A — composite PK** (opportunity_id + contact_id): the FK pair uniquely identifies the row and prevents duplicate pairs automatically.
- **RDBMS option B — surrogate PK** (own id column) + a UNIQUE constraint on the FK pair. More flexible; what most modern designs use.
- **Salesforce always uses option B** — every record gets a system 15/18-char Id. The junction's Id is the PK; the two Master-Detail fields are the FKs. Duplicate-pair prevention is NOT automatic on custom junctions — enforce it with a unique rule.

### Does the order of the two Master-Details matter?
- **In RDBMS — no.** Both FKs are identical to the engine; join direction is a per-query choice.
- **In Salesforce — yes**, but for PLATFORM behavior, not data: the first M-D is the primary master and controls sharing inheritance, ownership, report color, and default related-list placement.

> **The sharpened cross-domain model**
> Salesforce is a relational database with strong opinions. Lookup ≈ nullable FK with ON DELETE SET NULL. Master-Detail ≈ NOT NULL FK with ON DELETE CASCADE plus inherited ACLs. A junction ≈ a join table with a surrogate PK and two FKs, where one FK is arbitrarily designated "the important one" for permission inheritance — a concept plain SQL doesn't have.

**Bonus: SQL JOIN to report-type analogy (forward reference to Session 7)**
The primary-master choice is like baking a default JOIN direction into the schema itself — and that default also determines whose access-control list applies to the result set.

---

## 5. Does the Architecture Change Across Clouds?

**Short answer:** the core spine never changes; each cloud adds a layer on top.

- **Sales Cloud** = core spine (Lead → Account/Contact/Opportunity) + sales execution (Products, Quotes, Campaigns).
- **Service Cloud** = same spine + Cases, Entitlements, Knowledge. Account and Contact stay central; Case becomes the "transaction" object instead of Opportunity.
- **Data 360 (formerly Data Cloud)** = a fundamentally different architecture. Uses Data Model Objects (DMOs), is a system of reference (not record), and unifies data from many sources. Learn it separately.

**Practical takeaway:** master Sales Cloud's architecture first and Service Cloud largely falls into place. Data 360 is its own beast.

### BPO mapping (the throughline for this whole series)
- Client company = **Account** (with child Accounts for subsidiaries)
- Procurement lead / ops VP / exec sponsor = **Contacts**
- MSA pursuit = **Opportunity** (stages: Discovery → Proposal → Negotiation → MSA Signed)
- Services in the MSA = **Opportunity Line Items** referencing Products in a Pricebook
- Buyer roles = **Opportunity Contact Roles**
- Year-later expansion into a new LOB = a NEW Opportunity on the same Account
- Cold prospect not yet met = a **Lead**, until qualified and converted

---

## 6. Quiz A — Relationship Type Comprehension (8 scenarios)

For each scenario, identify the relationship type: Lookup, Master-Detail, Many-to-Many (Junction), Hierarchical, or Self-Lookup.

**Q1.** Custom **Project** related to **Account**. Projects have their own owners; if an Account is deleted the project should remain for historical tracking.
**Answer:** Lookup — Independent lifecycles, own owner, must survive parent deletion.

**Q2.** **Invoice Line Item** related to a custom **Invoice**. Line items are meaningless without their invoice; you want to roll up total amount.
**Answer:** Master-Detail — Child meaningless without parent + rollup + cascade delete desired.

**Q3.** **Cases** associated with **Knowledge Articles**; one Case references many articles and one article serves many Cases.
**Answer:** Many-to-Many (Junction) — Bidirectional many relationship. (Salesforce has built-in CaseArticle — check before building.)

**Q4.** Custom **Position** object where senior roles have junior positions reporting to them within the same object.
**Answer:** Self-Lookup — Custom object referencing itself; Hierarchical type isn't available on custom objects.

**Q5.** **Users** where you model who reports to whom for approval routing.
**Answer:** Hierarchical — The User object's ManagerId — the one place the Hierarchical type applies.

**Q6.** Custom **Subscription** related to **Account**; must always belong to an Account, delete with the Account, and inherit its visibility.
**Answer:** Master-Detail — Required parent + cascade delete + inherited sharing = all three M-D criteria.

**Q7.** Custom **Skill** and **Employee**; each Employee has many Skills (proficiency 1–5) and each Skill is held by many Employees.
**Answer:** Many-to-Many (Junction) — The proficiency rating is an attribute of the LINK — hallmark of a junction.

**Q8.** Custom **Property** (real estate) where some properties belong to a larger development that is itself a Property record — a tree on a custom object.
**Answer:** Self-Lookup — Custom object, tree structure, Hierarchical unavailable outside User.

> **Result: 8/8.**
> Strongest signals — Q4 vs Q5 (Self-Lookup vs the User-only Hierarchical type) and Q7 (a relationship that has its own properties needs its own object). Real-world wrinkle on Q6: in production, Subscription→Account is sometimes a Lookup-with-required instead of true M-D, because Subscriptions often need their own owner (a CSM) separate from the Account owner (an AE).

---

## 7. Quiz B — Hard Round: Competing Signals & Judgment (5 scenarios)

These have defensible answers with trade-offs, not single clean answers.

### Q1 · The Ownership Conflict
A custom **Engagement** must always tie to an Account and delete with it, but needs its own Owner (Delivery Manager ≠ Account Owner) and its own sharing.
**Answer:** **Lookup with a required field + restrict-delete**. You give up inherited ownership/sharing (the trade-off) but gain independent ownership. Set delete behavior to "don't allow deletion of parent with children" to mimic M-D integrity.

### Q2 · The Three-Parent Problem
**Time Entry** needs to relate to Employee, Engagement, and Task Type, with rollups on all three.
**Answer:** Only **two Master-Details** are allowed, so pick two (the choice sets sharing inheritance — often Engagement as primary so PMs see all entries on their engagement) and make the third a required Lookup. Roll up the Lookup via **DLRS** (common) or Flow/Apex. The "which is primary master?" decision is the most consequential one here.

### Q3 · Why is Opportunity→Account a Lookup, not M-D?
**Answer (multiple reasons):** (1) accidental Account deletion would cascade-wipe the entire pipeline; (2) Opportunities need their own owner (deal moves through specialist reps); (3) sharing flexibility — confidential deals vs routine renewals; (4) forecasting runs through the Opportunity owner's management chain, not the Account owner's; (5) historical B2B design where deal records must survive Account merges/deletes.

### Q4 · The Conversion Trap
Convert a 2-year-old, 50k-record Master-Detail (Contract→Amendment) to Lookup because Amendments now need to exist pre-Contract and have their own owner.
**Answer:** Data is NOT lost on conversion itself, but the real risks are: conversion is only allowed if no rollup summaries depend on the children (those must be deleted first, breaking dependent reports/automation); 50k children suddenly need owners assigned; a massive sharing recalculation can lock the org for hours; and MD→Lookup is far easier than the reverse, so it's near-irreversible. **Better alternative:** a separate "Draft Amendment" object linked in after the Contract exists — avoids the destructive conversion entirely.

### Q5 · The Junction That Isn't
Track which Products are available in which Regions (each pair has its own price + launch date); Region is already a custom object.
**Answer / questions to ask first:** Who owns Regions? What's the data volume? Does deleting a Region or Product mean the pairing (and its historical pricing) should vanish? **Deeper insight:** if Region is essentially static reference data (US/EMEA/APAC), it probably shouldn't be a custom object at all — a **picklist** or **Custom Metadata Type** may be the right call, which changes the relationship question entirely. Push back on auto-defaulting to two Master-Details.

> **Takeaways from the hard round**
> Two recurring lessons: (1) an enormous share of Salesforce design hinges on **who can see the data** — sharing and ownership drive relationship choices as much as data integrity does; (2) sometimes the best architectural decision is **"this shouldn't be a custom object"** — picklists, Custom Metadata Types, and Custom Settings are underused alternatives. Knowledge gaps (conversion mechanics, alternative object types) fill in with study; the architectural thinking was sound.
