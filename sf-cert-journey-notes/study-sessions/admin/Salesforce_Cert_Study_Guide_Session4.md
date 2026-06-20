# Session 4 — The Activity Model

> **Cert Track:** Platform Administrator I → Platform App Builder → Platform Developer I → Agentforce Specialist → JavaScript Developer I → Platform Developer II → Data Architect → Sharing & Visibility Architect → ***Salesforce Certified Application Architect***

> **Salesforce Admin Cert — Study Guide Series**
> Captured deep-dive session: concept explanations, mental-model clarifications, and a worked quiz with full answer rationale. Read the concepts, then test yourself against the quiz section before reading the graded answers.

**What this guide covers**
- Task vs Event — the two real Activity objects
- Why "Activity" is a UI concept, not a queryable object
- Polymorphic relationships: WhoId (people) and WhatId (things)
- Automatic Account rollup via the system-managed AccountId field
- Open Activities vs Activity History
- EmailMessage and logged Calls
- Quiz: 5 BPO-flavored scenarios with full rationale

---

## 1. The Two Core Objects: Task & Event

- **Task** — something to be done. Has a due date, status (Not Started / In Progress / Completed), priority. e.g. "Call the client back," "Send proposal."
- **Event** — something scheduled on the calendar with a start/end time and possible attendees. e.g. "Discovery meeting at 2pm," "QBR next Tuesday."

Conceptually simple. The weirdness starts with how they relate to other objects.

---

## 2. The Weird Part: "Activity" Isn't a Real Object

The Activity Timeline, "Activity History," and "the Activity object" all look like a parent object that Task and Event inherit from. **There isn't one.** Task and Event are separate tables with no shared parent record. The unified "Activity" is:

- A virtual UI concept (the timeline merges Tasks and Events for display)
- A reporting construct (the "Activities" report type unions the two)
- A set of shared fields both objects happen to have (Subject, Status, WhoId, WhatId, …)

**Consequences:**
- You can't query "all Activities" in SOQL — you query Task and Event separately and merge.
- You can't build automation on "any Activity" — separate Flows/triggers per object.
- Custom fields don't auto-appear on both; the special **Activity Custom Fields** feature exists precisely because of this quirk.

> **Why it's like this**
> Legacy. Salesforce built Tasks and Events as separate entities very early, then realized users think of them as one concept ("activities I did with this customer") and bolted a unified UI layer on top.

---

## 3. The Really Weird Part: WhoId & WhatId (Polymorphic)

Tasks and Events must relate to many different KINDS of records. Instead of separate FK columns per parent type, Salesforce uses **polymorphic relationships** — one field whose referenced object type varies per row (the ID's first 3 chars encode the object type).

- **WhoId** — the PERSON relationship. Points to a **Lead OR a Contact** (one or the other).
- **WhatId** — the THING relationship. Points to an **Account, Opportunity, Case, Campaign, Contract, or custom object** with Activities enabled.

**RDBMS contrast:** Like a foreign key column with no fixed referenced table. Normal JOINs don't work on WhatId; SOQL needs the special **TYPEOF** clause; custom logic often needs a CASE on the parent's object type before acting.

### WhoId vs WhatId — the distinction that trips up beginners
- **WhoId** = a person you interacted WITH.
- **WhatId** = a record the interaction was ABOUT.

A call with a Contact about an Opportunity: WhoId → Contact, WhatId → Opportunity. A call with just a Lead: WhoId → Lead, WhatId → null (Leads go in WhoId, not WhatId — though WhatId could point to a Campaign).

---

## 4. Automatic Account Rollup

A hidden system field **AccountId** on Task and Event is auto-populated by Salesforce, so activities bubble up to the Account page without being logged against the Account directly.

**The rollup follows multiple known paths — this is the key correction most people miss:**

- WhoId = a Contact → AccountId taken from the Contact's parent Account.
- WhatId = an Account directly → AccountId is that Account.
- WhatId = an Opportunity / Case / Contract (objects that themselves have an Account) → AccountId taken from that record's Account.
- **WhatId = a CUSTOM object → NO automatic rollup**, because Salesforce only traverses specific known paths, not arbitrary custom relationships. Use Flow/Apex or a different logging approach to surface those at the Account level.

> **The single most important fact from this session**
> AccountId auto-population works through MULTIPLE paths — WhoId AND WhatId — not just WhoId. Getting this wrong leads to bad designs (e.g. recommending a Sharing Rule for something the platform already handles, or assuming a custom-object WhatId will roll up when it won't).

---

## 5. Open Activities vs Activity History

- **Open Activities** — Tasks not yet completed + Events scheduled in the future.
- **Activity History** — completed Tasks + past Events.

Not separate tables — the same Task/Event records, filtered by status/date for display.

### 6. Activity-Adjacent Objects
- **EmailMessage** — the actual email content sent/received. Different from a Task of type Email (which is the "I followed up" log entry). They coexist.
- **Logged calls** — usually a Task with TaskSubtype = "Call" rather than a separate Call object.

**What breaks the standard patterns (summary)**
- No real parent object — "Activity" is UI only
- Polymorphic WhoId/WhatId — no fixed parent type
- System-populated AccountId rollup — not user-controlled
- Special Activity Custom Fields handling
- UI-level merging of two separate tables

**Exam note:** WhoId vs WhatId and Open vs History are Admin-level; the polymorphic/SOQL details lean developer-track.

---

## 7. Quiz — 5 Scenarios (with full rationale)

### Q1 · WhoId vs WhatId — the basics
30-min call with Sarah Chen (VP Ops at Acme) about the "Acme Q3 Expansion" Opportunity.
**Answer:** WhoId = Sarah Chen (Contact); WhatId = Acme Q3 Expansion (Opportunity); Salesforce auto-populates AccountId = Acme. **Result: correct.**

### Q2 · The Lead call
Call to qualify a Lead, Marcus Patel, who downloaded a whitepaper; not ready to buy.
**Answer:** WhoId = Marcus Patel (Lead); WhatId = null (no Account yet — though it COULD point to a Campaign if he came in through one). No Account rollup because Leads have no parent Account. **Result: correct, with the Campaign nuance added.**

### Q3 · The reporting problem
"All activities on any Enterprise-segment Opportunity in the last 90 days, by activity type."
**Answer:** The concept that "there is no Activity object" is right and is WHY the report builder is structured the way it is — but in practice Salesforce already solves the union for you via built-in report types (e.g. "Activities with Opportunities", "Tasks and Events"). The real gotchas: pick the right report type (wrong one = incomplete data), filter on WhatId = Opportunity without double-counting, handle custom subtypes, and ensure the segment field is reachable. **Result: right concept, missed how the platform handles it.**

### Q4 · The Account 360 question
Four activities appear on the Globex Account page though only one was logged against the Account directly. How?
**Answer:** Salesforce auto-populates the hidden AccountId from each activity's WhoId (Contact → parent Account) or WhatId (Opportunity/Case → Account). Activities logged on related Contacts and records bubble up; the one logged directly on the Account is the simple case. **Result: excellent.**

### Q5 · The tricky one
A custom **Client Health Check** (related to Globex Account) has Activities enabled. A CSM logs a Task with WhatId = the Client Health Check. Will it appear on Globex's Activity Timeline?
**Answer:** **No** — but for the right reason. The auto-AccountId logic only follows known paths (Contact, Opportunity, Case, Contract → Account). A **custom-object WhatId does not trigger it**, because Salesforce doesn't know which field on the custom object points to Account. Fixes: (1) log against the Account directly and reference the Health Check elsewhere; (2) a Flow/trigger to copy the Health Check's Account onto the Task's AccountId; (3) also set WhoId to a Globex Contact, which DOES trigger the standard rollup. **Result: conclusion right, but the reason "WhoId matters not WhatId" was wrong — WhatId matters too, just not for custom objects.**

> **Two corrections to lock in**
> (1) AccountId auto-population works through both WhoId and WhatId via known paths — not WhoId alone. (2) Standard report types already union Tasks and Events for you, so you rarely hand-merge in the report builder; the "no Activity object" fact is the reason behind the tooling, not a barrier to it.
