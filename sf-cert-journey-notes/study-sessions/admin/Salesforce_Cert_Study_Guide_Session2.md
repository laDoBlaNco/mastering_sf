# Salesforce Architect Study Guide — Session 2

> **Cert Track:** Platform Administrator I → Nonprofit Cloud Consultant → Platform App Builder → Service Cloud Consultant → Platform Developer I → Agentforce Specialist → JavaScript Developer I → Platform Developer II → Data Architect → Sharing & Visibility Architect → ***Salesforce Certified Application Architect***

### Platform Fundamentals

**Session Date:** June 2026
**Topics:** Metadata Architecture · Multi-Tenancy · EAV Model · Standard & Custom Indexes · Selectivity & Cardinality · Skinny Tables · Data Skew · Data Cloud

---

## Table of Contents

1. [Salesforce Platform — Big Picture](#1-salesforce-platform--big-picture)
2. [Multi-Tenancy — The Warehouse Model](#2-multi-tenancy--the-warehouse-model)
3. [Metadata-Driven Architecture](#3-metadata-driven-architecture)
4. [Entity-Attribute-Value (EAV) Pattern](#4-entity-attribute-value-eav-pattern)
5. [Standard vs Custom Indexes](#5-standard-vs-custom-indexes)
6. [Selectivity & Cardinality](#6-selectivity--cardinality)
7. [Skinny Tables](#7-skinny-tables)
8. [Data Skew](#8-data-skew)
9. [Data Cloud — Lakehouse + CDP](#9-data-cloud--lakehouse--cdp)
10. [Key Mental Model Corrections](#10-key-mental-model-corrections)
11. [Architect Exam Watch-Outs](#11-architect-exam-watch-outs)

---

## 1. Salesforce Platform — Big Picture

### Origins

Founded in 1999 by **Marc Benioff** (Oracle/sales visionary), **Parker Harris**, **Dave Moellenhoff**, and **Frank Dominguez** (the engineering core). Benioff's "no software" SaaS insight disrupted on-premise CRM by delivering everything via browser over shared, multi-tenant infrastructure.

### Core Problem Salesforce Solves

Data silos and broken communication across the enterprise — especially in Sales, Service, Marketing, and Commerce workflows.

### Growth: Organic + Acquisitions

| Cloud / Product | Origin | Notes |
|---|---|---|
| Sales Cloud | Core | Original CRM — leads, opps, accounts, contacts |
| Service Cloud | Core | Case management, omni-channel, field service |
| Marketing Cloud | ExactTarget acquisition | Email, journeys, automation — **separate stack** |
| Commerce Cloud | Demandware acquisition | B2C & B2B commerce — **separate stack** |
| MuleSoft | Acquired 2018 | Integration platform, API-led connectivity |
| Tableau | Acquired 2019 | Analytics & BI |
| Slack | Acquired 2021 | Collaboration — central to Agentforce flows |
| Data Cloud | Built/evolved | Real-time CDP + Lakehouse — separate stack |
| Agentforce | Built | Agentic AI layer on core platform — SF's strategic frontier |

> 💡 **Key architectural fact:** Not all "clouds" share the same architecture. Sales, Service, Experience, and the Platform live on the multi-tenant metadata core. Marketing Cloud and Commerce Cloud run on entirely separate stacks. This is critical for integration architecture — there is no native shortcut between them.

### Hyperforce

Salesforce's re-platforming onto public cloud infrastructure (AWS, Azure, GCP). Delivers true elastic scale and data residency compliance. Replaces the "one physical data center per region" model with fully cloud-native deployment.

---

## 2. Multi-Tenancy — The Warehouse Model

### ❌ The Wrong Mental Model: Apartment Building

Apartments have physical walls — separate units, separate storage. This implies a level of physical isolation that Salesforce does **not** have.

### ✅ The Correct Mental Model: Shared Warehouse

- One giant warehouse — all tenants' data in the same physical space
- Every item tagged with an **OrgID** — the platform knows what belongs to whom
- **Software-enforced isolation** — not physical isolation
- Governor limits exist because the warehouse is shared — one tenant's runaway query cannot be allowed to starve other tenants (the **"noisy neighbor" problem**)

> ⚠️ **The key shift:** Data isolation on Salesforce is a software-enforced guarantee, not a physical one. This is why security architecture and governor limits matter at the architectural level — they are the walls the platform builds in code.

### Why Multi-Tenancy Enables Salesforce's Business Model

| Factor | Reality |
|---|---|
| Amortized infra cost | One codebase, shared hardware — SF's cost to serve drops with scale |
| 3× / year upgrades | All customers upgraded simultaneously — no per-customer migration |
| SMB → Enterprise span | Editions handle feature access; platform scales from small orgs to 100M+ records |
| Pricing | Salesforce is **premium-priced** — multi-tenancy lowers SF's cost, not the sticker price |

---

## 3. Metadata-Driven Architecture

### This is the Foundation — Everything Else is a Consequence

Multi-tenancy and governor limits are **consequences** of the metadata-driven model, not the foundation. The foundation is: Salesforce stores your schema as metadata and uses a runtime engine to reconstruct "your" objects on the fly.

| Concept | What It Means |
|---|---|
| What metadata drives | Object definitions, field types, relationships, layouts, validation rules, automation, security — all stored as metadata rows, not schema changes |
| Physical storage | Custom field values live in shared generic tables (`MT_Data` is the illustrative name). Values stored as flexible, variable-length structures — not typed columns |
| Schema changes | Adding a field = adding a metadata row. No `ALTER TABLE`, no table lock, no downtime. Instant on objects with 50M+ records |
| Three upgrades/year | Everyone runs from the same metadata layer — no per-customer schema migration needed |

### Why ALTER TABLE Takes Time in Traditional RDBMS

On a traditional 5M-row table, `ADD COLUMN` must touch every row to add the new column slot, potentially locking the table for the duration.

On Salesforce, **there is no table to alter** — the field definition is a new metadata row, and new values flow into a structure already shaped to hold arbitrary fields.

> 💡 **The chain of reasoning:** The metadata model is why skinny tables and custom indexes exist at all. If data lived in clean typed columns, the standard RDBMS performance toolkit would be available natively. It doesn't — so the platform builds compensating tools.

---

## 4. Entity-Attribute-Value (EAV) Pattern

### What EAV Is

Instead of storing data as typed columns in rows (traditional RDBMS), EAV stores data as triples:

- **Entity** — the Record ID
- **Attribute** — the field definition (from metadata)
- **Value** — the generic stored value (variable-length, universal type)

A record with 20 fields doesn't have one row with 20 typed columns. It has 20 rows each saying "record X, field Y, value Z." The metadata layer tells the runtime engine how to interpret each value at query time.

### EAV vs Data Lake — Where the Parallel Holds and Breaks

| | EAV (Salesforce Core) | Data Lake |
|---|---|---|
| Schema enforcement | At the **metadata/application layer** | At **read time** (schema-on-read) |
| Transaction model | **OLTP** — ACID guarantees, referential integrity, row-level security | **OLAP** — batch reads, flexible exploration, no transactional guarantees |
| Structure opinions | Strong opinions about data structure | No opinions until you query |
| Primary use | Transactional CRM operations | Analytics, exploration, high-volume scans |

> 💡 **Clean label:** Salesforce is an **EAV-based OLTP system** with schema enforced at the metadata layer rather than the physical layer. Data Cloud is where the OLAP/lake complement comes in — they are complementary, not redundant.

---

## 5. Standard vs Custom Indexes

> ⚠️ **Critical framing:** The distinction between standard and custom indexes has **nothing to do with standard vs custom objects or fields.** It is about whether Salesforce created the index automatically or whether it was explicitly requested.

### Why Native RDBMS Indexing Doesn't Work Here

Oracle (underneath) can't build a meaningful B-tree on a generic flex column holding dates, numbers, and text from thousands of orgs all stored as variable-length strings. So Salesforce maintains **separate, strongly-typed copies** of field values in dedicated index pivot tables — where the value is in its real type and Oracle *can* build a proper index.

### Standard Indexes — Auto-Created, ~30% Threshold

Salesforce auto-indexes these fields on **any object** (standard or custom):

- `Id`
- `Name`
- `OwnerId`
- `CreatedDate`
- `SystemModstamp`
- `RecordTypeId`
- All lookup and master-detail relationship fields
- Any field marked **Unique**
- Any field marked **External ID**

### Custom Indexes — Requested via Support, ~10% Threshold

Any field not in the auto-indexed list requires a custom index requested through Salesforce Support. Stricter selectivity threshold (~10% / ~5%), capped ~333K records. There are limits on how many custom indexes an object can hold — each is a separately maintained structure with real overhead on the shared platform.

### What Silently Kills Index Usage

Even with an index in place, the optimizer will **not** use it when:

- Query is not selective (result set exceeds the threshold)
- Leading wildcard in filter: `WHERE Name LIKE '%Smith'`
- Negative operators: `!=`, `NOT IN`, `NOT LIKE`
- `NULL` comparisons (nulls are generally not indexed)
- Filters on most formula fields

---

## 6. Selectivity & Cardinality

### Selectivity Thresholds

| Index Type | Threshold (first 1M rows) | Threshold (beyond 1M) | Cap |
|---|---|---|---|
| Standard | < 30% | < 15% | ~1M records |
| Custom | < 10% | < 5% | ~333K records |

### Cardinality — The Real Driver of Index Worthiness

**Cardinality** = how many distinct values a field has relative to the row count, and how evenly records spread across those values. This is what actually determines whether indexing a field is useful — **not business importance.**

| Type | Example | Index Candidate? |
|---|---|---|
| High cardinality | Email, External ID, serial numbers | ✅ Yes — almost any filter is selective |
| Low cardinality (even) | Status with 5 values, even split | ❌ No — common values never selective |
| Low cardinality (skewed) | 99.9% false / 0.1% true, filter for `true` only | ✅ Yes — per-query selectivity applies |

> ⚠️ **Business importance and selectivity are orthogonal.** A field can be mission-critical and a terrible index candidate (`Status__c`). Never index based on importance — index based on query selectivity.

### Selectivity is Per-Query, Not Per-Field

Salesforce maintains **histograms of value distribution** and decides selectivity per value, not per field. The same indexed field can yield:
- An **index seek** for a rare value
- A **full scan** for a common value

...in the same query run against different filter values.

### Quick Decision Framework

| Scenario | Index Useful? | Why |
|---|---|---|
| 50/50 true/false split, 8M rows | ❌ No | Low cardinality — neither value ever selective |
| 99.9% false / 0.1% true, filter only for `true` | ✅ Yes | Per-query selectivity — filtered value IS selective |
| `Status__c`, 5 values, 10M rows, `'Active'` = 60% | ❌ No | 60% far exceeds the 10% custom index threshold |
| Email on 5M records, filter for one address | ✅ Yes | High cardinality — near-unique value |
| External ID on any object | ✅ Auto | Auto-indexed by platform |

---

## 7. Skinny Tables

### The Problem They Solve

In the EAV/generic-table model, standard fields and custom field flex data live in different physical locations. A query selecting both requires a JOIN at the database level. At millions of rows, that join is expensive — slow reports, slow list views, slow SOQL.

### What a Skinny Table Is

A Salesforce-maintained, **denormalized physical table** that pre-joins frequently-queried standard and custom fields into one flat structure. The join cost is paid once (at write time) rather than on every read.

| Property | Detail |
|---|---|
| Pre-joins fields | Frequently-queried standard + custom fields in one flat structure |
| Excludes soft-deletes | Recycle bin rows excluded — queries don't waste time scanning them |
| Auto-maintained | Salesforce keeps it in sync automatically — you never write to it directly |
| Transparent | Query optimizer uses it automatically — no changes to SOQL needed |
| Requested via Support | Not self-serve — requires a Salesforce Support case |
| Cap ~100 columns | Cannot hold more than ~100 fields per table |
| Single-object only | Cannot include fields from other objects — no cross-object joins |
| No sandbox auto-sync | Doesn't automatically propagate to sandboxes |

> 💡 **When to reach for a skinny table:** Large objects (millions of rows) with consistently slow reports, list views, or SOQL queries that join standard + custom fields. This is a heavyweight tool — confirm the query plan and exhaust index options first.

### Warehouse Analogy

A pre-packed pallet of the items you request most, kept by the warehouse door and automatically restocked — so the clerk doesn't run around re-collecting your scattered tagged items every time you make the same request.

---

## 8. Data Skew

### What Data Skew Is

Data skew occurs when a disproportionate number of records share the same value for a field that drives record access, ownership, or processing. The field may be perfectly indexed — the problem is that one specific value is **not selective**.

### The Classic LDV Skew Scenario

`OwnerId` on a 10M-record object. One integration user owns 2 million records:

- 8M records have normal owners → queries are selective, fast, index used ✅
- 2M records match the integration user → not selective, full scan, index ignored ❌

**Same field. Same index. Wildly different behavior per value.**

> ⚠️ This is why "I added an index but my query is still slow" is a classic LDV support ticket. The field is indexed — but the value being queried isn't selective.

### The Second Cost: Sharing Recalculation

OwnerId skew isn't just a query performance problem — it's a **sharing recalculation** problem. When the integration user's ownership changes or sharing rules update, the platform must recalculate access for 2 million records simultaneously. This connects directly to the Sharing & Visibility Architect domain — we'll go deep on this in a future session.

### Types of Skew

| Skew Type | Description | Impact |
|---|---|---|
| OwnerId skew | One owner holds large % of records | Selectivity failure + sharing recalc cost |
| ParentId skew | One parent has millions of children | Master-detail cascade costs, locking |
| Lookup skew | Millions of records point to same lookup target | Record lock contention on writes |

---

## 9. Data Cloud — Lakehouse + CDP

### What Data Cloud Is (and Isn't)

Data Cloud is **not** a pure data lake. It is a **real-time Customer Data Platform (CDP)** built on a lakehouse architecture — combining lake-style storage with structured semantic layers and activation capabilities.

| Layer | What It Adds |
|---|---|
| Lake characteristics | Ingests raw data from multiple sources, stores high volumes flexibly, schema-on-read style flexibility |
| Semantic layer | Canonical data model baked in — Individual, Contact Point, Engagement, Party. Data Cloud has strong opinions about what data *means*. A pure lake has none |
| Identity resolution | Core job: take one human being with 10 different IDs across 10 source systems and collapse them into one unified profile. A data lake cannot do this |
| Activation | Unified profiles feed back into Salesforce and other channels for real-time personalization and automation. A lake can't do this either |
| Separate stack | Does **not** run on the core EAV/MT_Data platform. Separate architecture, limits, API surface — connecting Sales Cloud to Data Cloud requires real integration work |
| Snowflake integration | Zero-copy data sharing. SF is not trying to replace your data warehouse — Data Cloud sits alongside it |

> 💡 **Clean label:** Data Cloud is a **real-time CDP built on a lakehouse architecture**, designed for identity resolution, unification, and activation — not raw storage.

### The Positioning Chain

```
Snowflake / Databricks / your data lake
  └─ Raw storage + analytics (OLAP)
       │
       ▼
Data Cloud
  └─ Pull from lake, resolve identities, unify profiles, activate (CDP layer)
       │
       ▼
Sales Cloud / Service Cloud
  └─ Act on those profiles in real-time CRM context (OLTP)
       │
       ▼
MuleSoft
  └─ Connects all of the above through API-led integration
```

---

## 10. Key Mental Model Corrections

These are the specific corrections from this session. Each one compounds — future topics reference these foundations constantly.

| Old Model | Corrected Model |
|---|---|
| **Apartment building** → physical walls, separate units | **Shared warehouse** → software-enforced isolation, OrgID tags |
| Multi-tenancy is the foundation | **Metadata-driven EAV is the foundation** — multi-tenancy and governor limits are consequences |
| Standard/Custom index = Standard/Custom object | **Nothing to do with object type** — it's automatic vs requested |
| Important field → index it | **Cardinality and selectivity drive index decisions**, not business importance |
| Selectivity is per-field | **Selectivity is per-query** — same field, different behavior per value |
| Data Cloud = data lake | **Data Cloud = Lakehouse + CDP** — adds semantic model, identity resolution, activation |
| Multi-tenancy lowers the price | Multi-tenancy lowers **SF's cost to serve** — Salesforce is premium-priced |

---

## 11. Architect Exam Watch-Outs

> These are the traps Data Architect and Application Architect scenario questions love to exploit from this session's content.

**Index ≠ used**
Having an index on a field does not guarantee the optimizer uses it. Always check selectivity, operator type (no negatives or leading wildcards), and whether it's a formula field.

---

**"I added an index, still slow"**
Classic LDV symptom — the query targets a non-selective value (data skew). Adding an index on `OwnerId` doesn't help if one owner has 20% of all records.

---

**Skinny table ≠ indexed fields**
Skinny tables solve JOIN cost between standard and flex tables. Custom indexes solve filter selectivity. They address different problems — don't conflate them.

---

**Marketing Cloud / Commerce Cloud integration**
These are NOT on the core platform. Integration always requires external API calls (MuleSoft, REST, etc.) — there is no native "same database" shortcut.

---

**Data Cloud ≠ replaces CRM data**
Data Cloud unifies and activates. It doesn't replace the transactional record layer in Sales/Service Cloud. The two complement each other.

---

**Skew + sharing recalculation**
OwnerId skew isn't just a query performance problem — it's a sharing recalculation problem. If 2M records share an owner and that owner's access changes, the platform recalculates 2M record-access rows simultaneously. Full detail in the Sharing & Visibility session.

---

**NULL comparisons**
NULLs are generally not indexed on the platform. `WHERE Field__c = NULL` will typically result in a full scan regardless of index status.

---

## Session Notes

These concepts compound. The EAV model explains indexes. Indexes explain skinny tables. Skinny tables and skew explain LDV architecture. LDV architecture connects directly to sharing and visibility. Every session builds on this foundation.

---

*Session 2 of the Salesforce Application Architect study series — June 2026*
*Session 3 whenever you're ready.*
