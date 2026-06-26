# Session 8 — Automation & Flow

> **Cert Track:** Platform Administrator I → Nonprofit Cloud Consultant → Platform App Builder → Platform Developer I → Agentforce Specialist → JavaScript Developer I → Platform Developer II → Data Architect → Sharing & Visibility Architect → ***Salesforce Certified Application Architect***

> **Salesforce Admin Cert — Study Guide Series**
> Exam weight: **15%** — third heaviest domain on the Platform Administrator exam.
> Mapped directly to official exam guide objectives. Includes full deep dive, Q&A clarifications, and worked quiz with complete rationale.

**What this guide covers**
- The automation spectrum and governing principle (start simple, escalate deliberately)
- The five Flow types and when to use each
- Flow building blocks — every element mapped to a programming concept
- Governor limits, bulk-safe design, and the collect-then-act pattern
- Flow vs Apex decision framework
- Approval Processes and how Flow integrates with them
- Flow security context — User, System with Sharing, System without Sharing
- Quiz: 8 exam-style scenarios with full rationale and clarifications

---

## The Governing Idea

**Automation & Flow is the "make the platform work for you" domain.** Every previous session built the foundation — the data model, relationships, security, sharing, data quality. Automation is what makes that foundation *active* rather than passive. Instead of users manually updating fields, sending emails, creating records, and following checklists, automation does it consistently, instantly, and at scale.

The exam tests two things simultaneously:
1. **What each tool does** — the mechanics of Flow and the automation spectrum
2. **When to use which tool** — the judgment to pick the right automation for the right scenario

That second point is the harder one and is where most exam questions live.

---

## The Automation Spectrum

Salesforce has several automation tools on a spectrum from simple to complex:

```
Simple ←————————————————————————————→ Complex

Validation    Formula    Flow      Apex
Rules         Fields     Builder   Code

(prevent      (calculate  (automate  (code for
 bad data)     values)     processes) edge cases)
```

**The governing principle:** always use the simplest tool that can express the logic you need. Reach for the next level only when the current level runs out of capability.

- **Validation Rules** — prevent bad data from being saved. Not "automation" in the triggering sense, but part of the same mindset of making the platform enforce business rules.
- **Formula Fields** — calculate values automatically. No user action required. Read-only output.
- **Flow Builder** — the primary declarative automation tool. Covers the vast middle ground between formula fields and code.
- **Apex** — code for logic that Flow genuinely cannot express. Last resort, not first instinct.

> **Important:** Workflow Rules and Process Builder were both retired December 31, 2025. They are no longer on the exam. Flow Builder is the sole declarative automation focus. Any study material referencing Workflow Rules or Process Builder as active tools is outdated.

### Where Validation Rules and Formula Fields Appear on the Exam

These tools are primarily tested in the Object Manager and Data & Analytics domains. However, the Automation domain tests your **judgment** about which tool is right for a given scenario. A question might ask "what's the best solution for requirement X?" where one answer is a Validation Rule, one is a Formula Field, one is a Flow, and one is an Approval Process. Knowing that a Validation Rule prevents saves but doesn't trigger processes, or that a Formula Field calculates but doesn't automate actions, is essential Automation domain knowledge even though those tools live elsewhere.

### The Exam Format

The exam is 60 questions presented in random order — not grouped by domain. Security questions, Flow questions, and Data questions are intermixed. Domain weights tell you how many questions come from each area, not that they appear in blocks. You can't "finish" a section and relax.

---

## The Five Flow Types

The most important first question about any Flow scenario: which Flow type do you need?

### Type 1: Record-Triggered Flow

**Triggered by:** a record being created, updated, or deleted.

The most important Flow type for the exam and the direct replacement for Workflow Rules and Process Builder. This is what fires automatically when a record changes.

**Three trigger options:**
- A record is created
- A record is created or updated
- A record is deleted

**The critical sub-decision: Before Save vs After Save**

```
User saves a record
         ↓
Before Save Flow runs    ← updates the SAME record being saved
                           without a separate DML operation
         ↓
Record is committed to the database
         ↓
After Save Flow runs     ← can interact with OTHER records,
                           send emails, call Apex, submit for approval
```

| | Before Save | After Save |
|---|---|---|
| Timing | Before record is written to database | After record is committed |
| Update same record | Yes — directly, no DML needed | No — requires separate Update Records element |
| Create/update OTHER records | No | Yes |
| Send emails | No | Yes |
| Submit for Approval | No — record needs a committed ID | Yes |
| HTTP callouts | No | Yes |
| Chatter posts | No | Yes |
| Performance | Faster — preferred when possible | Slightly more overhead |

> **Why emails are "external actions":** Before Save runs while the save is still in progress. The platform won't allow anything that reaches outside the current transaction — because if the transaction rolls back, you can't unsend an email, cancel an API call, or un-post to Chatter. After Save means the record is committed and the transaction is safe to interact with the outside world.

> **External actions (all require After Save):** emails, HTTP callouts to external systems, Platform Event publishing, creating/updating other records, Chatter posts, Submit for Approval.

**BPO example:** when an Opportunity moves to Closed Won:
- Stamp "Closed Won Date" on the Opportunity itself → **Before Save** (same record, no DML)
- Update Account's "Active Client" checkbox → **After Save** (different record)
- Create onboarding Tasks → **After Save** (creating other records)
- Send welcome email → **After Save** (external action)

### Type 2: Screen Flow

**Triggered by:** a user clicking a button, link, or action — or embedded on a record/app page.

An interactive guided process that presents screens to users — collecting input, displaying information, and performing actions based on what the user enters. Replaces manual multi-step processes with guided automation.

**Key behaviors:**
- Has a UI — presents actual screens with fields, text, buttons
- User input drives decisions and data within the Flow
- Can display data retrieved from Salesforce
- Runs in the context of the logged-in user by default (respects their sharing and permissions)

**BPO example:** a "New Client Onboarding" button on the Account page launches a Screen Flow that walks the delivery manager through creating the initial Case, assigning team members, setting SLA dates, and sending the welcome email — all in one guided experience.

### Type 3: Scheduled Flow

**Triggered by:** a time-based schedule — runs automatically at a specified time and frequency.

Processes a batch of records matching certain criteria at the scheduled time. No user interaction.

**Key behaviors:**
- Runs at a specified time: once, daily, or weekly
- Operates on a collection of records matching filter criteria
- Runs as a **single execution** — one transaction regardless of how many records it processes
- Runs in system context by default

**Important distinction from Record-Triggered Flows:** a Scheduled Flow fires once and processes all qualifying records in that single execution. A Record-Triggered Flow fires once PER record in a batch. This significantly changes the governor limit math — more on this in the Governor Limits section.

**Common use cases:**
- Send a weekly summary email every Monday morning
- Escalate Cases open more than 5 days without an update
- Update a "Days Since Last Activity" field on Accounts nightly
- Find dormant client Accounts and create check-in Tasks

### Type 4: Autolaunched Flow (No UI)

**Triggered by:** called programmatically — from Apex, another Flow (as a subflow), a REST API call, or a Platform Event.

A Flow with no screen elements and no direct user trigger. Designed to be called by something else — another automation, a developer's code, or an external system.

**Key behaviors:**
- No screens — purely logic and data operations
- Used for reusable logic that multiple other automations need to share
- Can be called from Apex, other Flows, external API calls

**BPO example:** a reusable "Create Standard Onboarding Tasks" Autolaunched Flow called by both a Record-Triggered Flow (automatic on Closed Won) and a Screen Flow (manual onboarding button) — rather than duplicating task-creation logic in two places.

### Type 5: Platform Event-Triggered Flow

**Triggered by:** a Platform Event message being published — from Salesforce or an external system via API.

Part of Salesforce's event-driven architecture. Used for real-time integration scenarios when an external system needs to trigger automation inside Salesforce.

For the Admin exam: know it exists and what triggers it. Deep implementation is developer-track.

### Flow Types Summary

| Flow Type | Triggered By | Has UI | Common Use Case |
|---|---|---|---|
| Record-Triggered | Record create/update/delete | No | Automate on data change |
| Screen | User action (button/link) | Yes | Guided user processes |
| Scheduled | Time/schedule | No | Batch processing, periodic jobs |
| Autolaunched | Apex, other Flow, API | No | Reusable logic |
| Platform Event-Triggered | Platform Event message | No | Real-time external integration |

---

## Flow Building Blocks

Flow Builder IS programming through point and click. Every element maps directly to a fundamental programming concept.

| Programming Concept | Flow Element | Purpose |
|---|---|---|
| Event listener | Trigger | Entry point — what starts the Flow |
| SELECT query | Get Records | Retrieve records from Salesforce |
| INSERT | Create Records | Create new records |
| UPDATE | Update Records | Modify existing records |
| DELETE | Delete Records | Remove records |
| IF/ELSE IF/ELSE | Decision | Branch based on conditions |
| Variable assignment | Assignment | Set variable values |
| FOR EACH loop | Loop | Iterate through a collection |
| Method/function call | Action element | Invoke pre-built or custom operations |
| Function definition + call | Subflow element | Call another Flow |

### Trigger

Every Flow starts here — the entry point that initiates execution. Configuration depends on Flow type: a record change, a schedule, a user action, or a programmatic call.

### Get Records

Retrieves records from Salesforce based on filter criteria. Equivalent to a SELECT query.

Configure: which object, filter conditions (WHERE clause), how many records (first only vs all matching), which fields to retrieve.

Output: a single record variable or a record collection variable.

### Create Records

Creates one or more new records. Equivalent to INSERT. Can create a single record or an entire collection in one operation.

### Update Records

Updates existing records. Equivalent to UPDATE. Can update the triggering record directly (in After Save flows) or a collection retrieved by Get Records.

### Delete Records

Deletes records. Deleted records go to the Recycle Bin. Use carefully — bulk deletions are hard to reverse.

### Decision

The branching element. Tests conditions and routes the Flow down different paths.

**Structure:**
- Define one or more **outcomes** — each with conditions that must be true
- Outcomes are evaluated in ORDER — the first matching outcome wins
- A **Default** outcome catches everything that doesn't match any defined outcome

> **Exam note:** order of outcomes matters. If two outcomes could both be true, the first one wins. This mirrors IF/ELSE IF — once a match is found, evaluation stops.

**BPO example:** Case Priority check:
- Outcome 1: Priority = Critical → escalation path
- Outcome 2: Priority = High → urgent path
- Default: everything else → standard path

### Assignment

Sets the value of a variable. Used to build up data before using it in a Create or Update element, or to add records to a collection inside a Loop.

**Common uses:**
- Set field values on a variable record before creating it
- Increment a counter inside a Loop
- Add a record to a collection variable
- Concatenate text values

### Loop

Iterates through a collection of records one at a time.

**Structure:**
- Input: a collection variable
- The Loop automatically assigns the current record to a Loop Variable each iteration — you don't do this manually
- Inside the Loop: Decision, Assignment, and other elements operate on the current record
- After all iterations: Flow exits the Loop and continues

**Critical bulk-safe pattern:** do NOT perform Get Records or Create/Update Records inside a Loop. Instead, use Assignment inside the Loop to build a collection, then perform a single Create/Update Records OUTSIDE the Loop on the full collection. More on this in Governor Limits.

### Action Element

Invokes pre-built or custom operations. When you add an Action element and open the picker, you see categories including:

- Email Alerts
- Submit for Approval
- Post to Chatter
- Send Email
- **Apex Invocable Methods** ← "calling Apex" happens HERE, inside the Action element

> **Architecture clarification:** "Call Apex" is not a separate element — it's a selection you make INSIDE the Action element's picker. You use an Action element and select an Apex invocable method from the available options.

**Invocable Actions:** developers write Apex methods annotated with `@InvocableMethod` which then appear in the Action element's picker. This is how Flow's capabilities are extended without modifying Flow itself — the developer writes the method, the admin calls it from Flow.

### Subflow Element

A dedicated, separate element (NOT inside the Action element) that calls another Flow.

> **Architecture clarification:** Subflow is its own element on the Flow canvas — distinct from the Action element. You use a Subflow element and select the Autolaunched Flow to call by name.

```
Flow Canvas Elements
│
├── Action Element
│     └── picker includes:
│           ├── Email Alerts
│           ├── Submit for Approval
│           ├── Post to Chatter
│           └── Apex Invocable Methods  ← "Call Apex" lives here
│
└── Subflow Element  ← separate dedicated element
      └── references an Autolaunched Flow by name
```

**Why subflows matter architecturally:** instead of duplicating logic in multiple Flows, build it once as an Autolaunched Flow and call it from multiple parent Flows as a subflow. Changes to the shared logic only need to happen in one place.

---

## Governor Limits and Bulk-Safe Design

This is the concept that separates "knows Flow" from "understands Flow."

### What Are Governor Limits?

Salesforce is a multi-tenant platform — many organizations share the same underlying infrastructure. To prevent any one org from consuming disproportionate resources, Salesforce enforces **Governor Limits** — hard caps on what any single transaction can do.

**Key limits for the Admin exam:**

| Limit | Cap |
|---|---|
| SOQL queries per transaction | 100 |
| DML operations per transaction | 150 |
| Records retrieved by SOQL | 50,000 |
| Records processed by DML | 10,000 |

These limits apply **per transaction**. Exceeding them causes the entire transaction to fail and roll back — no partial saves.

### What Is a Transaction?

A transaction is one complete unit of work. Everything that happens as a result of one save operation is part of one transaction — including all Flows, triggers, and Apex that fire from that save.

**The critical difference between Flow types:**

**Record-Triggered Flow:** fires once PER RECORD in a batch. When Data Loader updates 200 Opportunities in one batch, the Flow fires 200 times — all within ONE transaction. Governor limits apply to the transaction, so they must cover all 200 executions combined.

**Scheduled Flow:** fires ONCE as a single execution, regardless of how many records it processes. One transaction covers the entire run. This significantly reduces governor limit risk compared to Record-Triggered Flows at scale.

### Step-by-Step: How Limits Become a Problem

**Step 1 — Single record (no problem):**

User saves one Opportunity. Flow fires once:
- 1 Get Records = 1 SOQL
- 1 Create Records = 1 DML

Total: 1 SOQL, 1 DML. Well within limits.

**Step 2 — Batch of 200 records (potential problem):**

Data Loader updates 200 Opportunities. One transaction. Flow fires 200 times:
- 200 Get Records = 200 SOQL queries → **exceeds the 100 limit**
- 200 Create Records = 200 DML operations → **exceeds the 150 limit**

This is why testing with one record in Sandbox gives false confidence. The Flow works perfectly for one record and fails catastrophically in production with bulk data.

**Salesforce's automatic bulkification:** for simple, unconditional queries, the Flow runtime attempts to batch multiple executions' SOQL queries together — combining 200 identical queries into far fewer actual database calls. This works reliably for straightforward linear Flows with consistent, predictable queries.

### What Makes a Query "Consistent" (and Therefore Bulkifiable)

A consistent query has the same structure on every execution — same object, same fields, same filter pattern — with only the variable value (like a record ID) changing between executions. The runtime can predict and batch these.

```
CONSISTENT — runtime can bulkify:
Get Records: Cases WHERE AccountId = [current Opportunity's AccountId]
← Same object, same fields, same filter field — only the ID changes per execution

INCONSISTENT — runtime cannot bulkify:
Decision: is Priority = High?
  Yes → Get Records: Cases WHERE AccountId = X AND Priority = High
  No  → Get Records: Contacts WHERE AccountId = X AND Title = 'Manager'
← Different objects, different fields — runtime can't predict which query runs
```

The compiler analogy: a strongly typed language gives the compiler enough information at compile time to optimize — it knows the unchanging facts about the program. Salesforce's runtime does the same thing. Queries outside conditional branches are predictable facts the runtime can optimize. Queries inside conditional branches are ambiguous until execution — the runtime can't safely batch what it doesn't know will run.

**The bottom line:** move SOQL queries outside of Decision branches whenever possible so the runtime can see them as predictable, batchable operations.

### The Anti-Pattern: SOQL or DML Inside a Loop

The most dangerous Flow design mistake:

```
BAD — SOQL inside Loop:
Loop through 200 Account records
  └── Get Records: Cases for this Account  ← 1 SOQL per iteration
                                              = 200 SOQL queries
                                              = FAILS at iteration 101

BAD — DML inside Loop:
Loop through 200 Account records
  └── Create Records: Task for this Account ← 1 DML per iteration
                                              = 200 DML operations
                                              = FAILS at iteration 151
```

### The Pattern: Collect Then Act

```
GOOD:
Get Records: ALL relevant Cases (before the Loop, one query)
Loop through Accounts
  └── Assignment: add Task to Task collection  ← memory only, no SOQL/DML
Exit Loop
Create Records: Task collection  ← 1 DML for all records
```

**The principle:** think in sets, not individuals. Get all the data you need before the Loop. Build your output collection inside the Loop using Assignment. Write everything in one operation outside the Loop.

### When Collect Then Act Isn't Enough

"Collect then act" fixes the Loop anti-pattern within a single Flow execution. It doesn't fully solve the problem of a Record-Triggered Flow being executed hundreds of times in one batch transaction.

**The fix for complex Record-Triggered Flows:**

- **Redesign:** move queries outside conditional branches so the runtime can bulkify them. Queries that are unconditional and consistent give the runtime the predictability it needs to optimize.
- **If the logic genuinely can't be restructured:** escalate to Apex. A developer writes explicit bulk-safe code — processing all 200 records with 2 SOQL queries total using Apex collections and maps. The developer does manually what the runtime was trying to do automatically.

**For Scheduled Flows:** the concern is different. One execution, one transaction, limits apply to the whole run. SOQL inside a Loop still consumes your 100-query budget, but it's not multiplied by batch size. A Scheduled Flow looping through 500 Accounts with a Get Records per iteration still hits 500 SOQL queries — still exceeds the limit — but the failure mode is different from a Record-Triggered Flow.

---

## When to Use Flow vs Apex

### Use Flow When:
- The logic can be expressed declaratively — conditions, field updates, record creation, emails, approvals
- The process is triggered by record changes, schedules, or user interaction
- Admins need to maintain it without developer involvement
- The automation needs to be visible and auditable by non-developers

**Flow covers the vast majority of automation needs.** Always try Flow first.

### Use Apex When:
- **Complex logic Flow can't express** — deeply nested conditions, complex mathematical operations, proprietary algorithms
- **Performance at extreme scale** — for very high-volume operations where Flow's overhead matters
- **External callouts requiring distributed transaction guarantees** — if Salesforce records AND an external system must stay in sync, and failure in one must roll back the other, Apex provides patterns Flow cannot
- **Custom invocable actions** — when Flow needs a capability it doesn't have natively, a developer writes an Apex invocable action that Flow can call

### The Atomicity Point

Salesforce transactions are atomic — if anything fails, everything within that transaction rolls back. For operations entirely within Salesforce, Flow handles this natively and correctly.

The guarantee breaks when a transaction spans Salesforce AND an external system. If Salesforce commits but the external API call fails, you have inconsistent state across systems — Salesforce records exist with no corresponding external entry. Apex provides patterns (Platform Events, compensating transactions) to manage this. No point-and-click tool can fully guarantee distributed transaction consistency.

**For the exam:** if a requirement involves external system sync where failure in one must roll back the other → Apex.

### The Decision Framework

```
Can Flow express this logic?
  ├── Yes → Use Flow
  └── No → Can I extend Flow with an Apex invocable action?
              ├── Yes → Flow + Apex hybrid (Flow for process, Apex for the specific step)
              └── No → Pure Apex
```

### The Hybrid Approach

Many real-world implementations use Flow for overall process logic and Apex only for the specific step requiring code. This keeps as much as possible in admin hands while using developer resources surgically.

**BPO example:** a Closed Won Flow handles Task creation, email sending, and Account updates in Flow. The one step that calculates a complex weighted revenue score using a proprietary algorithm uses an Apex invocable action. Flow calls it as an Action element and writes the returned score to a custom field.

### Retired Tools

| Tool | Status | Replacement |
|---|---|---|
| Workflow Rules | Retired Dec 31, 2025 | Record-Triggered Flow (Before Save) |
| Process Builder | Retired Dec 31, 2025 | Record-Triggered Flow |
| Flow Builder | Active — primary tool | N/A |
| Apex | Active — code-required scenarios | N/A |

---

## Approval Processes

Approval Processes are a separate Salesforce feature that works alongside Flow. They formalize multi-step review workflows where a record must be approved before a state change is finalized.

### Key Components

**Entry Criteria:** conditions a record must meet to be submitted. If not met, the record can't be submitted.

**Approval Steps:** sequential (or parallel) steps the record moves through. Each step designates an approver:
- A specific user
- The record owner's manager (uses the User ManagerId / Role Hierarchy from Session 3)
- A queue
- A related user field on the record

**Actions — configured per outcome:**

| Outcome | Available Actions |
|---|---|
| Approval (per step or final) | Field Update, Email Alert, Task, Outbound Message |
| Rejection (per step or final) | Field Update, Email Alert, Task, Outbound Message |
| Recall | Field Update, Email Alert, Task, Outbound Message |

**While pending approval:** the record is **locked** — the submitter cannot edit it (unless the admin configures otherwise). The approver can Approve, Reject, or Reassign.

### Flow + Approval Process Integration

- A **Record-Triggered Flow (After Save)** can submit a record for approval automatically using the Submit for Approval action — when certain conditions are met, without requiring the user to click Submit manually
- The **Approval Process** handles everything that happens during and after the approval decision
- Flow handles "when to submit"; the Approval Process handles "what happens during and after"

**Why After Save:** Submit for Approval requires a committed record ID. Before Save runs before the record is written to the database — no ID exists yet.

**On batching:** Submit for Approval works per record and is handled efficiently by the runtime. If Data Loader updates 200 Opportunities and 50 qualify for approval submission, each gets its own Approval Request. The submission itself doesn't create governor limit problems — it's a straightforward platform action, not a complex SOQL operation inside a conditional.

**On approval/rejection timing:** the approval or rejection decision happens asynchronously — when the approver acts, which is after the original transaction is complete. Rejection doesn't fail the original batch. The original transaction simply submits and locks the record.

**On "Closed Won before approval" — a real design consideration:**

Setting Stage = Closed Won before approval is granted is technically inaccurate. Real-world patterns:

- **Simple approach (platform standard):** allow Closed Won, lock the record, trust the approval process validates the discount terms
- **Accurate approach:** use a Validation Rule preventing Stage = Closed Won when Discount > 20% unless Approved_Discount__c = TRUE. The rep can't close until approved.
- **Visible approach:** use a custom Stage value "Pending Approval" — makes the approval status visible in pipeline reporting (the instinct from the quiz was architecturally sound)

The exam tests the simple platform-standard approach. The others are real-world refinements.

---

## Flow Security Context

Flow runs in a security context that determines what it can see and do — and which context applies depends on the Flow type and configuration.

### The Three Contexts

| Context | Bypasses FLS | Bypasses Record-Level Sharing (OWD/Hierarchy/Rules) |
|---|---|---|
| User or Profile Access | No | No |
| System Context with Sharing | **Yes** | No |
| System Context without Sharing | **Yes** | Yes |

> **Critical clarification:** BOTH System Context modes bypass FLS. The only difference between them is record-level sharing. "With Sharing" respects OWD, Role Hierarchy, and Sharing Rules. "Without Sharing" ignores them entirely.
>
> This is counterintuitive because the word "sharing" sounds like it could include FLS. In Salesforce's technical vocabulary, "sharing" refers specifically to the record-level sharing model (Layer 4 from Session 6). FLS is a Layer 2-3 mechanism that both system contexts bypass.

### Default Context by Flow Type

| Flow Type | Default Context |
|---|---|
| Record-Triggered (Before/After Save) | System Context without Sharing |
| Screen Flow | User or Profile Access |
| Scheduled Flow | System Context without Sharing |
| Autolaunched Flow | Depends on how it's called |

### Configuring Security Context in Flow Builder

Screen Flows default to User Context but can be reconfigured in the Flow's properties panel in Flow Builder — no changes to Sharing Rules or Permission Sets needed.

**Choosing the right context for a Screen Flow:**

- Need to bypass FLS (show a hidden field) but keep record-level security intact? → **System Context with Sharing**
- Need to access records beyond the user's sharing access AND bypass FLS? → **System Context without Sharing**
- Want to respect ALL of the user's permissions (most secure)? → **User or Profile Access** (default)

**When to use System Context with Sharing:** a guided Screen Flow for help desk agents needs to display Account billing details hidden from agents via FLS — but the business process requires it within this specific workflow. Run the Flow in System Context with Sharing to expose the field without changing the agent's permanent FLS permissions and without bypassing record-level security.

**The cleaner production alternative:** a Permission Set granting scoped FLS visibility to the agents is more precise than running the entire Flow in system context. Both work; the Permission Set is more targeted. If the exam says "fix it in Flow Builder without changing FLS or Sharing Rules," the security context setting is the answer.

### Task Ownership in Flows

A record created by a Flow (including Tasks) is assigned to the **running user** by default unless the OwnerId field is explicitly set in the Create Records element. If a Flow needs to create a Task assigned to someone other than the current user — a supervisor, a queue, a specific manager — the OwnerId field must be explicitly mapped in the Create Records element. The Flow typically needs a Get Records on the User object first to retrieve the target user's ID.

---

## The Full Automation Decision Framework

```
What needs to happen?
│
├── Prevent bad data from being saved?
│     └── Validation Rule
│
├── Calculate a value automatically on a record?
│     └── Formula Field
│
├── Something needs to happen when a record is created/changed?
│     └── Record-Triggered Flow
│           ├── Updating fields on the SAME record? → Before Save
│           └── Involving OTHER records or external actions? → After Save
│
├── Something needs to happen on a schedule?
│     └── Scheduled Flow
│
├── Need to guide a user through a multi-step process?
│     └── Screen Flow
│
├── Need a record to go through an approval process?
│     └── Approval Process
│         (Flow submits; Approval Process handles approval steps and actions)
│         (Any Flow type can submit for approval — not only Record-Triggered)
│
├── Need reusable logic called from multiple places?
│     └── Autolaunched Flow (called as subflow)
│
├── Logic is too complex for Flow?
│     └── Apex (or Flow + Apex hybrid)
│
└── Need to respond to real-time external events?
      └── Platform Event-Triggered Flow
```

> **Any Flow type can trigger an Approval Process** using the Submit for Approval action — not only Record-Triggered Flows. Screen Flows, Scheduled Flows, and Autolaunched Flows can all submit records for approval when the Submit for Approval action is used.

---

## Industry Standard: Flow Design Principles

### One Trigger Context Per Flow

Before Save and After Save are obligatorily separate Flows — they run at different points in the transaction and cannot be combined. This is a platform constraint.

Within the same trigger context, combine all logic into ONE Flow:
- One Before Save Flow per object
- One After Save Flow per object (combine all After Save automation for that object)

**Why:** every Flow on the same trigger fires for every save on that object. Five separate After Save Flows on Opportunity means five separate evaluations on every Opportunity save, even when most conditions aren't met. One Flow with a Decision element at the top is dramatically more efficient.

### The Clean BPO Example

**When Opportunity moves to Closed Won:**

```
Flow 1: Opportunity Before Save
  └── Condition: Stage changed to Closed Won
  └── Set Closed Won Date = TODAY() (same record — Before Save)

Flow 2: Opportunity After Save
  └── Condition: Stage changed to Closed Won
  └── Get Account → Update Active_Client__c = TRUE
  └── Create 5 onboarding Tasks
  └── Send welcome email to primary Contact
  └── Decision: Amount > $500K? → Submit for Approval

Flow 3: Scheduled Flow (weekly)
  └── Get Accounts where Active_Client__c = TRUE
       AND no Case created in last 7 days
  └── Loop → Assignment: collect escalation Tasks
  └── Create Records: Task collection (bulk-safe — one DML)
```

Three Flows, not seven. Before Save and After Save necessarily separate. Everything within each trigger context combined into one Flow.

---

## Quiz — 8 Exam-Style Scenarios with Full Rationale

### Q1 · The Right Tool (Objective: Automation Spectrum)

Four requirements — correct tool for each.

**Requirement A:** Auto-populate "Full Name" by combining First Name + Last Name.
**Answer:** Formula Field. A simple calculated read-only value requires no automation trigger. Start simple — Formula Field handles this without any Flow.

**Requirement B:** Block a Case save when Status = Closed and Resolution is blank.
**Answer:** Validation Rule. This is preventing bad data, not triggering automation. Validation Rule is the right and simplest tool.

**Requirement C:** Every Monday, find Cases with Last Modified Date > 5 days ago and create escalation Tasks.
**Answer:** Scheduled Flow. Time-based trigger with batch processing — exactly what Scheduled Flows are designed for.

**Requirement D:** When Opportunity moves to Closed Won, create a Task, update Account's Active Client checkbox, and send a welcome email.
**Answer:** Record-Triggered Flow, After Save. Too complex for Validation Rule or Formula Field. All three actions involve other records or external actions (email) — all require After Save.

### Q2 · Before Save vs After Save (Objective: Record-Triggered Flow)

Four requirements on an Opportunity update:

**Requirement 1:** Stamp "Last Stage Change Date" on the Opportunity when Stage changes.
**Answer:** Before Save. Updating a field on the same triggering record — the canonical Before Save use case. No DML needed; faster.

**Requirement 2:** Create a new child "Revenue Forecast" custom object record.
**Answer:** After Save. Creating another record is a separate DML operation — only possible After Save when the triggering record is committed.

**Requirement 3:** Update the related Account's "Total Pipeline Value" by summing open Opportunity amounts.
**Answer:** After Save. Two reasons: (1) updating another object (the Account) requires After Save; (2) the current Opportunity's Amount must be committed to be included in the sum — After Save ensures it's in the database.

**Requirement 4:** Send an email to the CFO when Amount exceeds $1M.
**Answer:** After Save. Sending email is an external action — not allowed Before Save. The transaction must be committed before the platform will send anything outside of Salesforce.

### Q3 · Flow Design — Scheduled Flow (Objective: Building Blocks + Governor Limits)

**Requirement:** nightly at midnight, find all Accounts where Active_Client__c = TRUE and most recent Case was closed more than 30 days ago. Create a Check-In Task per qualifying Account, assigned to the Account Owner, due in 7 days.

**Answer:**

**Flow type:** Scheduled Flow. Time-based trigger, batch processing.

**Design:**

- **Trigger:** configure to run nightly at midnight
- **Get Records (A):** retrieve all Accounts where Active_Client__c = TRUE — outside any loop, before any conditionals, so the runtime can bulkify
- **Get Records (B):** retrieve all Cases for those Accounts (filter: AccountId IN collected Account IDs, sorted by Close Date) — one query before the loop
- **Assignment:** initialize an empty Task collection variable
- **Loop:** iterate through the Account collection
  - **Decision:** does the most recent Case for this Account have CloseDate > 30 days ago? (reference the pre-fetched Case data)
  - **Yes path — Assignment:** build a Task record (OwnerId = Account Owner, Due Date = TODAY() + 7, Subject = "Client Check-In") and add to Task collection
  - **Default path:** nothing — the Loop proceeds to the next Account
- **Create Records:** Task collection — ONE DML operation outside the Loop for all Tasks

**Bulk-safe:** Get Records happens before the Loop (no SOQL per iteration). Create Records happens after the Loop (no DML per iteration). The Decision checks pre-fetched data — memory only.

**Note on SOQL and Scheduled Flows:** a Scheduled Flow runs as a single execution (one transaction covering all records), unlike Record-Triggered Flows which run once per record per batch. This reduces the governor limit risk, but SOQL inside a Loop still consumes the transaction's 100-query budget and can still fail at sufficient volume.

### Q4 · The Governor Limit Problem (Objective: Governor Limits)

**Design under review:** Record-Triggered Flow on Case update. Get Records inside a Decision branch, another Get Records inside the same branch, Create Records inside the branch. 300+ Cases updated via Data Loader.

**Answer:**

**The problem:** both Get Records elements are inside the Decision branch. 300 Cases processed in batches of 200 = one transaction with 200 Flow executions. Because the queries are inside a conditional, the runtime cannot bulkify them. Each execution runs both queries independently:
- 200 executions × 2 Get Records = up to 400 SOQL queries → exceeds the 100 limit
- Create Records inside the branch = up to 200 DML operations → exceeds the 150 limit

**The fix:**

```
Move Get Records OUTSIDE the Decision — before any conditional branching:
  Get Records: Account for this Case (unconditional — runtime can bulkify)
  Get Records: all open Cases on this Account (unconditional — runtime can bulkify)

Decision: Priority = High?
  Yes path:
    Assignment: add Task to Task collection
    Assignment: add to email notification list
  No path: nothing

Create Records: Task collection  ← outside Decision, one DML
Send Emails                      ← outside Decision
```

The Decision remains — it checks field values already in memory (no SOQL cost). SOQL and DML move outside all conditional branches where the runtime can see them as predictable, batchable operations.

**Why not remove the Decision entirely:** moving all Get Records unconditionally is correct. Removing the Decision and relying on filters in Get Records would mean querying Account and Cases data for every Case update regardless of Priority — unnecessary SOQL cost for non-High Priority Cases. The Decision stays; the data retrieval moves before it.

### Q5 · The Approval Process (Objective: Approval Processes + Flow Integration)

**Requirement:** Opportunities with Discount > 20% must be approved by the rep's manager before Closed Won. Approval stamps "Approved Discount" checkbox. Rejection reverts Stage to Negotiation and emails the rep.

**Answer:**

**What triggers the submission:** a Record-Triggered Flow, After Save, with trigger filter: Stage = Closed Won AND Discount > 20%. The Flow uses the Submit for Approval action. After Save is required — Submit for Approval needs a committed record ID.

**Why Record-Triggered, not Screen Flow:** the trigger is a record state change (Stage = Closed Won), not a user manually initiating a process. Record-Triggered is correct.

**Approval Process components:**
- **Entry Criteria:** Discount > 20%
- **Approval Step:** routes to the record owner's manager (ManagerId from the User object — Role Hierarchy)
- **Final Approval Actions:** Field Update — Approved_Discount__c = TRUE
- **Final Rejection Actions:** Field Update — Stage = Negotiation; Email Alert — notify the rep

**On batching:** Submit for Approval works per record and is handled efficiently by the runtime. Each qualifying Opportunity gets its own Approval Request. No governor limit concern from the submission itself.

**On the locked record:** while pending approval the Opportunity is locked — Stage stays at Closed Won but is locked and cannot be edited. The approval is validating the discount terms, not the close itself. A more accurate design would use a custom "Pending Approval" Stage or a Validation Rule preventing Closed Won until Approved_Discount__c = TRUE — but the exam tests the standard platform approach.

**On rejection timing:** the approval/rejection decision is asynchronous — it happens when the manager acts, after the original transaction is long complete. Rejection does NOT fail the original batch transaction.

### Q6 · Flow vs Apex (Objective: Flow vs Apex Decision)

**Scenario A:** Account Industry changes to Healthcare → create three compliance Tasks and send notification email.
**Answer:** Flow. Creating Tasks and sending emails are native Flow capabilities. No complexity requiring code.

**Scenario B:** Case closes → calculate a complex weighted satisfaction score using a proprietary logarithmic algorithm, write to a custom field.
**Answer:** Flow + Apex hybrid. The algorithm is outside Flow's declarative capability — Apex invocable method handles the calculation. Everything else (trigger, writing the result to the field) stays in Flow. The developer writes the calculation method; the admin calls it from an Action element.

**Scenario C:** Opportunity closes Won → update Salesforce records AND write to an external financial system — failure in either must roll back the other.
**Answer:** Pure Apex. The distributed transaction guarantee (roll back Salesforce if external system fails, or vice versa) cannot be provided by any declarative tool. Apex with explicit compensation logic is required.

**Scenario D:** Screen Flow guides agents through client intake — at one step, check an external identity verification system via REST API and display the result.
**Answer:** Flow (with Apex hybrid if needed). Standard REST callouts are within Flow's native HTTP callout capability. If the callout involves complex authentication or request signing beyond Flow's ability, escalate to a Flow + Apex hybrid where Apex handles the callout and returns the result as an invocable action output.

### Q7 · The Security Context (Objective: Flow Security)

**Problem:** Screen Flow for contact center agents. Contract Value field (hidden via FLS) doesn't appear. Supervisor Task isn't being created correctly.

**Answer:**

**Issue 1 — Contract Value not appearing:**
Screen Flows run in User or Profile Access by default — this respects FLS, so the hidden Contract Value field isn't visible. Fix in Flow Builder: change the Flow's security context to **System Context with Sharing**. This bypasses FLS (making the field visible) while still respecting the org's record-level sharing model.

> **Key fact:** BOTH System Context modes bypass FLS. The difference between "with sharing" and "without sharing" is only record-level sharing. "With Sharing" is the more conservative and correct choice here — it bypasses FLS without also bypassing OWD and sharing rules.

If the exam says "fix without changing FLS or Sharing Rules" — the security context setting in Flow Builder IS the answer. A Permission Set granting scoped FLS visibility would also work and is the cleaner production solution, but it changes FLS (which the question excluded).

**Issue 2 — Supervisor Task not created correctly:**
A Task created by a Flow is assigned to the running user (the agent) by default. To assign it to the supervisor, the OwnerId field must be explicitly mapped in the Create Records element to the supervisor's User ID. The Flow needs a Get Records on the User object to retrieve the supervisor's ID first — likely using the agent's ManagerId field to find their supervisor — then sets OwnerId = supervisor's ID in the Create Records element. Record owners always see their own records regardless of sharing rules, so once the Task is assigned to the supervisor, visibility is handled automatically.

### Q8 · The Design Review (Objective: Governor Limits + Bulk-Safe Design)

**Flow under review:** Record-Triggered on Opportunity. Elements 1-2: Get Records (Account and Contacts) run unconditionally. Element 3: Decision (Amount > $100K). Yes path: Loop through Contacts → Get Records for Cases per Contact (inside Loop) → Create Records for Tasks (inside Loop). No path: Assignment sets a "Low Value" flag variable that isn't used elsewhere.

**Problems identified:**

**Problem 1 — Get Records inside Loop (Element 5):**
Querying Cases per Contact inside the Loop is SOQL inside a Loop — the most dangerous anti-pattern. In a batch of 200 Opportunities, this query runs once per Contact per Opportunity execution, rapidly exhausting the 100-query limit.

**Problem 2 — Create Records inside Loop (Element 6):**
DML inside a Loop — the second anti-pattern. Up to 200 DML operations per batch execution, exceeding the 150 limit.

**Problem 3 — Unused Assignment (Element 8):**
The "Low Value" flag variable is set but never used. Dead code — adds confusion and should be removed or connected to something meaningful.

**Problem 4 — Elements 1 and 2 before the Decision:**
Get Records for Account and Contacts run unconditionally for every Opportunity update, regardless of Amount. For a $50K Opportunity the queries run and the data is never used. Better design: move Get Records inside the Yes path of the Decision — but that puts them inside a conditional, which the runtime can't bulkify. The real problem is this Flow is trying to do too much: querying Contacts and their Cases for every high-value Opportunity update is architecturally questionable and should be reconsidered.

**The fix:**

```
Trigger: Opportunity created or updated
Get Records: Account (unconditional — runtime can bulkify)
Decision: Amount > $100K?
  Yes path:
    Get Records: Contacts on this Account (outside Loop)
    Get Records: ALL open Cases for those Contacts (outside Loop — one query)
    Assignment: initialize Task collection
    Loop: through Contacts with open Cases
      Assignment: add Task to Task collection
    Create Records: Task collection  ← outside Loop, one DML
    Send Email to Account Owner
  No path:
    [empty — remove the unused Assignment]
```

**On the exam:** if presented with "what is the primary problem with this Flow design?" the answer is SOQL and DML inside the Loop — the most direct governor limit violation. The architectural concern about the Flow doing too much would appear as a secondary answer choice about "consider redesigning or using Apex for complex logic."

---

## The 12 Most Exam-Likely Facts

1. **Workflow Rules and Process Builder are retired — Flow Builder is the sole declarative automation tool**
2. **Five Flow types: Record-Triggered, Screen, Scheduled, Autolaunched, Platform Event-Triggered**
3. **Before Save: updates the SAME record, no DML, faster — After Save: other records and external actions**
4. **External actions (emails, callouts, Chatter, Submit for Approval, other record DML) all require After Save**
5. **Decision element outcomes are evaluated in order — first match wins; Default catches everything else**
6. **Loop: the Loop variable is assigned automatically — you don't do it manually**
7. **SOQL and DML inside a Loop are anti-patterns — collect with Assignment inside, act outside with one operation**
8. **Record-Triggered Flows fire per record in a batch; Scheduled Flows fire once for all records**
9. **Both System Context modes bypass FLS — the only difference is record-level sharing**
10. **System Context with Sharing: bypasses FLS, respects OWD/Role Hierarchy/Sharing Rules**
11. **Submit for Approval requires After Save (committed record ID); rejection is asynchronous — doesn't fail the batch**
12. **Any Flow type can submit for approval — not only Record-Triggered Flows**
