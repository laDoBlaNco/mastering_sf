# Session 7 — Data & Analytics Management

> **Cert Track:** Platform Administrator I → Nonprofit Cloud Consultant → Platform App Builder → Service Cloud Consultant → Platform Developer I → Agentforce Specialist → JavaScript Developer I → Platform Developer II → Data Architect → Sharing & Visibility Architect → ***Salesforce Certified Application Architect***

> **Salesforce Admin Cert — Study Guide Series**
> Exam weight: **17%** — the single heaviest domain on the Platform Administrator exam.
> Five official exam objectives, fully covered. Captured deep-dive session including concept explanations, mental-model clarifications, and a worked quiz with full answer rationale.

**What this guide covers**
- Objective 1: Data import, export, update, transfer, mass delete & archival
- Objective 2: Data validation tools — validation rules, duplicate & matching rules
- Objective 3: Report creation and customization — types, formats, filters, bucket fields, formulas
- Objective 4: Impact of the sharing model on reports
- Objective 5: Dashboard creation and modification

---

## The Governing Idea

Data & Analytics is the **proof-of-concept domain**. Everything in previous sessions — the data model, object relationships, security, sharing — only matters if it produces trustworthy, visible, and actionable data. This domain tests whether you understand how to get clean data in, keep it clean, make it visible through reports, and control who sees what.

The three sub-topics connect like this:

```
Data Quality & Import/Export  →  Reports  →  Dashboards
  (clean data going in)        (make it     (make it
                                 visible)    actionable)
         ↑                          ↑
  Sharing Model affects        Sharing Model affects
  who can manage data          what appears in results
```

The sharing model from Session 6 threads through all of it — which is why this domain is weighted so heavily. It is not just "can you build a report." It is "do you understand the whole platform well enough to make data work correctly end to end."

---

## Objective 1 — Data Import, Export, Update, Transfer, Mass Delete & Archival

### What Counts as a Bulk Operation

In Salesforce, a bulk operation is any operation processing more than one record in a single transaction. In practice, "bulk" means using one of the data management tools to process a significant volume of records in a batch — as opposed to a user manually editing one record at a time through the UI. All five tools below are bulk tools by this definition.

### Tool 1: Data Import Wizard

Built directly into Setup — no download or installation required.

**Supported objects:** Accounts, Contacts, Leads, Solutions, Campaign Members, and custom objects. Does NOT support Opportunities, Cases, Products, or most other standard objects.

**Record limit:** 50,000 records per import.

**Operations supported:**
- **Insert** — create new records
- **Update** — modify existing records matched by Salesforce record ID or another specified field
- **Upsert** — insert new records AND update existing ones in a single pass, using a matching field to determine which records already exist

**Duplicate handling:** integrates with your org's existing Duplicate Rules and Matching Rules automatically.

**Best for:** admins doing routine moderate-volume imports on supported objects where no installation or scheduling is needed.

### Tool 2: Data Loader

A separate desktop application that must be downloaded and installed.

**Supported objects:** ALL standard and custom objects — including Opportunities, Cases, Products, and everything the Import Wizard cannot touch.

**Record limit:** up to 5 million records per operation.

**Operations supported:**
- **Insert** — create new records
- **Update** — modify existing records (requires Salesforce record ID in the file)
- **Upsert** — insert new + update existing using an External ID field to match records
- **Delete** — delete records, sending them to the Recycle Bin (requires Salesforce record ID)
- **Hard Delete** — permanently delete records, bypassing the Recycle Bin (requires a special permission)
- **Export** — extract active records matching your criteria to a CSV file
- **Export All** — extract all records including those currently in the Recycle Bin

**Can be scheduled:** via command line or batch scripts for automated recurring operations.

**Best for:** large volume operations, objects not supported by Import Wizard, automation needs, and bulk delete operations.

### The Decision Framework

| Condition | Tool |
|---|---|
| Volume over 50,000 records | Data Loader |
| Object is Opportunity, Case, or unsupported | Data Loader |
| Need to schedule automated imports | Data Loader |
| Need to permanently delete (bypass Recycle Bin) | Data Loader Hard Delete |
| Simple Contact/Lead/Account import under 50K | Import Wizard |
| Need upsert on any object | Both support it; Data Loader works on all objects |

### Tool 3: Mass Delete Records

A Setup UI tool — not Data Loader. Found in Setup under **Data Management → Mass Delete Records**.

**What it does:** lets admins delete large numbers of records from specific objects using filter criteria set in the UI — without needing a CSV file.

**Supported objects:** Leads, Contacts, Accounts, Cases, Activities, Products, and Solutions.

**Key behaviors:**
- Filter-driven — you set criteria in the UI; Salesforce identifies matching records
- Deleted records go to the Recycle Bin by default (recoverable for 15 days)
- Has a permanent delete option that bypasses the Recycle Bin
- Admin-only functionality

**Distinction from Data Loader Delete:** Mass Delete is a UI-based Setup tool for common objects, driven by filter criteria without needing a file. Data Loader Delete requires a CSV with specific record IDs and works on any object. Same end result, different mechanism.

### Tool 4: Mass Transfer Records

Found in Setup under **Data Management → Mass Transfer Records**.

**What it does:** reassigns record ownership from one user to another in bulk.

**Supported objects:** Leads, Accounts, and custom objects.

**Key behaviors:**
- Filter which records to transfer using field criteria
- Optionally transfer open Activities along with the records — almost always the right choice when handing off an active rep's accounts
- Respects the sharing model — the new owner gets full owner access
- Admin action — the user whose records are being transferred does not need to approve it

### Tool 5: Data Export and Backup

**Data Export (Setup → Data Management → Data Export):**
- Exports all org data as CSV files, one per object
- Schedule options: **weekly or monthly only** — no other frequencies
- Files available for download for 48 hours after generation
- Used primarily for backup and compliance

**Data Loader Export vs Export All:**
- **Export** — extracts active records matching your filter criteria
- **Export All** — extracts all records including those in the Recycle Bin; useful for auditing deletions

### Risks of Bulk Operations and Tool-Specific Behavior

| Risk | Data Import Wizard | Data Loader | Mass Delete | Mass Transfer | Export |
|---|---|---|---|---|---|
| Validation Rules fire | Yes | Yes (default) | No (deletes bypass) | Yes | No (read-only) |
| Triggers & Flows fire | Yes | Yes (default) | Before/after delete triggers | Yes (on update) | No |
| API limits consumed | Yes | Yes (primary concern) | No (UI tool) | No (UI tool) | Data Loader yes; Setup Export no |

**The no-easy-undo risk applies to all write operations.** Always test in Sandbox first. A bulk delete or update is very hard to reverse.

### Archival Considerations

As orgs mature, records accumulate and consume storage. Salesforce has per-edition storage limits.

**Options:**

**Recycle Bin:** deleted records stay for 15 days before permanent removal. Not an archival strategy — just a safety net.

**External archival:** export old data to an external warehouse (AWS S3, Snowflake, etc.) and delete from Salesforce. Reduces storage but data is no longer searchable, reportable, or accessible inside Salesforce.

**Big Objects:** a Salesforce-native archival mechanism that stores billions of records directly in Salesforce in compressed format.
- Queryable but no standard reports, no triggers, no workflows
- Limited SOQL support
- Used for compliance/audit data that must be retained but is rarely accessed
- Deep implementation is developer-track; for the exam know it exists, what it's for, and its key constraints

---

## Objective 2 — Data Validation Tools

### The Purpose of Data Validation

Reports are only as useful as the data underneath them. Three validation layers operate at different points:

```
Matching Rules + Duplicate Rules  →  prevent duplicate records
Validation Rules                  →  enforce field-level data quality on save
Required Fields                   →  enforce presence of critical data
```

### Validation Rules

A Validation Rule prevents a record from being saved when a formula evaluates to **TRUE**. The formula describes the **error condition** — the state that should NOT be allowed.

> **The counterintuitive framing:** you are not writing "when should this save succeed." You are writing "when should this save FAIL." Think of it as describing the crime. TRUE = the crime is happening = block it.

```
Example:
Requirement: A Case cannot be closed unless Resolution is populated
             AND the related Account has an active contract.

Validation Rule Formula (the ERROR condition):
AND(
  ISPICKVAL(Status, "Closed"),
  OR(
    ISBLANK(Resolution__c),
    Account.Active_Contract__c = FALSE
  )
)

TRUE  → Status is Closed AND either Resolution is blank OR
        Account has no active contract → Save BLOCKED

FALSE → Status is not Closed, OR both Resolution is populated
        AND Account has an active contract → Save proceeds
```

> **Common logic trap:** "both must be true to allow save" translates to "block if either is false." The OR lives inside the error condition. Getting this backwards is one of the most frequently missed validation rule concepts.

**Key behaviors:**
- Fire on both record create AND edit
- Apply to ALL users including System Administrators — unless the rule formula explicitly excludes them (e.g. `$Profile.Name <> "System Administrator"`)
- Nobody gets a free pass by virtue of permissions alone — the exemption must be written into the rule
- Can reference parent object fields using cross-object syntax (e.g. `Account.Active_Contract__c` on a Case rule)
- Can be temporarily deactivated for data migrations

**Modify All and Validation Rules — the precise mechanic:**
- **Object-level Modify All** — bypasses record-level sharing (Layer 4) only. Validation Rules still fire.
- **System permission Modify All Data** — bypasses record-level sharing across all objects. Validation Rules STILL fire unless the rule formula explicitly excludes that profile.
- Validation Rules are data integrity controls, not access controls. Their job applies regardless of who is saving the record. One being bypassed doesn't imply the other should be.

**Required fields — three levels:**

| Level | Enforced Where | Security Control? |
|---|---|---|
| Field definition (required) | UI and API — universal | Yes — true data integrity |
| Validation Rule (ISBLANK check) | UI and API — conditional | Yes — under specific conditions |
| Page layout (required) | UI only on that layout | No — easily bypassed via API |

### Matching Rules

Matching Rules define HOW Salesforce identifies potential duplicate records.

**Match methods:**
- **Exact** — must match character for character
- **Exact (case-insensitive)** — ignores capitalization
- **Fuzzy** — handles slight variations and common typos (e.g. "Sara" vs "Sarah")
- **First N characters** — only the first N characters must match

**Key facts:**
- Can combine multiple field criteria with AND/OR logic
- Standard Matching Rules come out of the box for Leads, Contacts, and Accounts
- A Matching Rule by itself does nothing — it is dormant until referenced by a Duplicate Rule

### Duplicate Rules

Duplicate Rules define WHAT HAPPENS when a Matching Rule finds a potential duplicate.

**Three actions:**

| Action | What Happens | User Experience |
|---|---|---|
| Block | Save prevented entirely | Must resolve before proceeding |
| Allow with Alert | Warning shown; user can save anyway | User makes the final call |
| Report Only | Silent logging; no interruption | User sees nothing |

**What gets created:**
When a duplicate is saved despite an alert, or when Report Only mode fires, Salesforce creates a **Duplicate Record Set** — a specific record (on the Duplicate Record Sets object) that groups the potential duplicates together for admin review and merging.

**The merge operation:** for Accounts, Contacts, and Leads, Salesforce has a native merge tool. Select up to three records, choose which field values to retain, and Salesforce combines them into one master record. Child records (Activities, Cases, Opportunities) are re-parented to the surviving master.

**The relationship:**
```
Matching Rule      →    Duplicate Rule    →    Action
(how to find           (what to do when        Block /
 potential dups)        a match is found)       Alert /
                                                Report Only
                                                → Duplicate Record Set created
```

You need BOTH. Neither works without the other.

---

## Objective 3 — Reports: Creation and Customization

### Report Types — The Foundation

The Report Type is the single most architecturally important choice in report building. It determines:
- Which primary object the report is based on
- Which related objects' fields are available
- Whether records without related records are included or excluded (the relationship pattern)

> **Critical:** the relationship pattern is baked INTO the Report Type itself — not a separate formatting choice you make afterward. Choosing "Accounts with Opportunities" locks you into an inner-join pattern. If you need the left-outer version, you must select a different report type entirely.

**Standard Report Types** are pre-built by Salesforce. **Custom Report Types** are built by admins and offer additional customization including renaming fields and choosing exactly which fields are available.

### The Four Relationship Patterns

| Pattern | SQL Equivalent | Behavior |
|---|---|---|
| A with B | INNER JOIN | Only A records with at least one B record. A records with zero B children are excluded entirely. |
| A with or without B | LEFT OUTER JOIN | All A records appear. B fields are blank for A records with no children. |
| A with B with or without C | INNER + LEFT OUTER JOIN | Must have at least one B. C is optional — blank if no C children exist. |
| A with or without B with or without C | Two LEFT OUTER JOINs | All A records appear. Both B and C are optional. |

**Maximum objects in a custom report type: 4**

> **The most exam-likely scenario:** a user runs "Accounts with Opportunities" and a specific Account doesn't appear. The cause is that Account has zero Opportunities — the inner-join pattern excluded it. This is expected behavior, not a bug.

> **The subtle companion scenario:** a parent Account appears in "Accounts with Opportunities" but its Opportunity columns are blank. The cause is that the Account HAS Opportunities (satisfying the inner join) but none of them pass the report's field filters. The join determines appearance; filters determine which child data shows. These are two separate mechanisms operating sequentially.

### Cross Filters

Cross Filters are a type of report filter — they live in the report builder's filter section. They filter the primary object based on the presence or absence of related records, **without adding the related object's fields to the report**.

| SQL Equivalent | Cross Filter | Result |
|---|---|---|
| WHERE EXISTS | Accounts WITH Opportunities | Only Accounts that have at least one Opportunity |
| WHERE NOT EXISTS | Accounts WITHOUT open Opportunities | Only Accounts with no open Opportunities |

**The key distinction — "no related object fields come through":**

Imagine you want a list of Accounts that have no open Opportunities.

- **Option A — "Accounts with or without Opportunities" report type:** your report shows Account columns AND Opportunity columns (blank for Accounts with no Opps). Opportunity is part of the report's data structure.
- **Option B — Cross Filter on an "Accounts" report:** your report shows only Account columns. You add a Cross Filter: "Accounts WITHOUT Opportunities." Opportunity columns cannot be added — the Opportunity object is being used as a gate to include/exclude Account records, not as a data source.

Same Account records in the result. Completely different report structure. Cross Filters are for "filter by relationship" not "show data from relationship."

**Cross Filters vs relationship patterns — are they the same thing?**
They are related but operate at different levels:
- **Relationship pattern** — structural, part of the report type, determines the JOIN behavior for all records
- **Cross Filter** — conditional, applied per report in the filter section, uses relationships to include or exclude specific records

Both use object relationships. Relationship patterns are permanent to the report type; Cross Filters are per-report filter conditions.

### The Four Report Formats

| Format | Grouping | Subtotals | Drives Dashboard | Best Use |
|---|---|---|---|---|
| Tabular | None | None | Table only (with row limit) | Simple lists |
| Summary | Up to 3 row groupings | Yes | Yes | Grouped totals |
| Matrix | Row + column groupings | Both axes | Yes | Two-dimensional analysis |
| Joined | Multiple blocks per report | Per block | Limited | Cross-object comparison |

> **Exam decision pattern:** Does it need grouping? (eliminates Tabular) Does it need two-dimensional grouping? (points to Matrix) Does it need multiple report types together? (points to Joined)

### Filtering

**Standard Filters:** built into every report automatically. Include the "Show Me" filter (All records / My records / My team's records) and date range filters. Always present, cannot be removed.

**Field Filters:** conditions on specific fields. "Stage equals Closed Won" or "Amount greater than 50,000." The primary way to narrow report results.

**Filter Logic:** defines how multiple Field Filters combine. Default is AND. Filter Logic lets you express "Filter 1 AND (Filter 2 OR Filter 3)" using Boolean operators and parentheses.

**Cross Filters:** covered above — filter by related object presence/absence without pulling in related fields.

### Bucket Fields

A Bucket Field groups report values into custom categories **without creating a new field on the object**. It exists only within the report and produces a column in the report results — one calculated category value per row.

**Three types:**

| Type | Groups | Example |
|---|---|---|
| Numeric | Number ranges | Amount into Small ($0–$50K) / Medium ($50K–$200K) / Large ($200K+) |
| Picklist | Picklist values into categories | Stage into Early / Active / Closed |
| Text | Text values into categories | Region text into East / West / Other |

**Bucket Field vs formula field on the object:**

| | Bucket Field | Formula Field on Object |
|---|---|---|
| Lives | Report only | Object schema — permanent |
| Visible on record pages | No | Yes |
| Available in automation | No | Yes |
| Consistent across reports | No — per report | Yes — always available |
| Schema impact | None | Yes — adds a field |
| Use when | One-off analysis | Permanent categorization needed everywhere |

> **The scenario where you need BOTH:** a formula field calculates standard tiers for automation and record display. A Bucket Field in a specific report uses different tier boundaries for a one-off analysis. Same concept, different definitions, serving different purposes simultaneously.

### Report Formulas

Two distinct types — the exam tests whether you know the difference.

**Row-Level Formulas:**
- Calculate a value for each individual record row
- Operate on field values from that specific record
- Produce a column in the report — one value per row
- Example: `CloseDate - CreatedDate` = Days to Close for each Opportunity row individually
- Created entirely within the report builder — no field is added to the object

**Summary Formulas:**
- Calculate values at the GROUP level — operate on aggregated data
- Only available in Summary and Matrix formats (where grouping exists)
- Use aggregate functions: SUM, COUNT, AVG, MIN, MAX
- Example: `DIVIDE(SUM(Amount), COUNT(Id))` = Average Deal Size per Stage grouping
- Appear in subtotal and grand total rows — not on individual record rows

> **The key distinction:** Row-Level Formula = individual record. Summary Formula = aggregate group. A report can have both simultaneously.

**Row-Level Formula vs Formula field type on the object:**
Both produce a calculated column. The difference is scope and permanence:
- **Formula field on object** — permanent schema addition, visible on records, available everywhere, drives automation
- **Row-Level Formula in report** — exists only in this report, no schema change, invisible outside this report

Same end result (a calculated column). Completely different scope.

### Additional Report Features

**Hide Details:** toggles between showing individual record rows (details shown) and showing only group summary rows (details hidden). Useful when sharing summary-level data with leadership without exposing individual records.

**Grouping and Subtotals:**
- Summary reports: up to 3 levels of row grouping, each with its own subtotal row
- Matrix reports: row AND column groupings, subtotals on both axes
- For each numeric field: choose SUM, AVG, MIN, MAX, or COUNT per grouping
- Grand totals appear at the bottom of Summary reports and in the corner of Matrix reports
- Subtotals and grand totals are what Dashboard components consume — Tabular reports have neither, which is why they can't drive most Dashboard widgets

**Chart Settings:**
- Chart types: bar, line, pie, donut, funnel, scatter
- Combination charts: overlay two chart types (e.g. bars for Amount + line for Count) on one chart
- Axis settings: min/max values, axis labels, data label visibility
- Chart grouping maps to the report's grouping fields

**Troubleshooting Missing Fields — check in this order:**

1. **FLS** — if the running user doesn't have Field-Level Security visibility on a field, it won't appear in reports (Layer 3 from Session 6, enforced everywhere including reports)
2. **Custom Report Type field layout** — for custom report types, fields must be explicitly added to the type's field layout to be available in reports
3. **Related object not in report type** — if the field lives on an object not included in the report type's relationship chain, it's not accessible
4. **Relationship chain too deep** — the 4-object maximum means some deeply nested fields simply can't be reached

**Custom Report Type field renaming:** when building a Custom Report Type, you can rename any field as it appears in that report type — "Contract_Start_Date__c" can appear as "Service Start Date." The rename is scoped to that report type only; the field's actual API name and label are unchanged everywhere else.

---

## Objective 4 — Impact of the Sharing Model on Reports

### How Sharing Affects Report Results

Every report respects the **running user's** sharing access at the moment it runs. The sharing model is the invisible first filter — it constrains which records are even eligible to appear before any explicit report filter runs.

Two users running the identical report with identical filters at the same time can see completely different record counts and totals. This is intended behavior — the report is a window into the data the running user is entitled to see.

**What determines the running user's record visibility:**
- OWD access on the object
- Position in the Role Hierarchy (records owned by subordinates)
- Sharing Rules that apply to them or groups they belong to
- Manual Shares on specific records
- View All permission — bypasses record-level security entirely; user sees all records in report results

### The "Show Me" Standard Filter

Every report includes a built-in "Show Me" filter:
- **All records** — everything the running user has sharing access to
- **My records** — only records they personally own
- **My team's records** — records owned by them and Role Hierarchy subordinates

This is a user-level convenience filter, not an admin security control. It operates within whatever sharing access the user already has — it cannot expand access beyond their permissions.

### Report Folders and Visibility

Reports live in folders. Folder access is a separate control layer from the data sharing model. A user needs folder access to see and run a report AND sharing access to see the data it returns. Both must be present.

**Folder access levels:**

| Level | Can Run | Can Edit | Can Share Folder |
|---|---|---|---|
| Viewer | Yes | No | No |
| Editor | Yes | Yes | No |
| Manager | Yes | Yes | Yes |

**Share folders with:** specific Users, Roles, Roles and Subordinates, or Public Groups.

**The Unfiled Public Reports trap:**
- Default folder for reports not explicitly placed somewhere
- Accessible to ALL internal users by default — no folder permission required
- A report saved here accidentally is visible to every user in the org, regardless of what data it exposes
- Fix: move to a properly permissioned folder immediately; audit Unfiled Public Reports regularly as admin hygiene

> **The nuance:** Unfiled Public Reports gives any user the ability to RUN the report. What data they see is still constrained by their own sharing access. A contact center agent can run a pipeline report saved there — but they'll only see Opportunities their sharing access allows. Folder access and record-level access are independent.

**Private folder:** every user has a personal Private folder visible only to them and System Administrators.

### Sharing Fields in Reports

Key fields that surface ownership and sharing context:
- **Owner / Owner Name** — who owns each record; critical for "my records vs all records" analysis
- **Created By / Last Modified By** — audit fields showing who touched each record
- **Record ID** — useful for referencing records in bulk operations or external analysis

---

## Objective 5 — Dashboards: Creation and Modification

### What a Dashboard Is

A Dashboard is a visual display of aggregated report data. Every visual element is a **component**, and every component is powered by a **source report**.

### Dashboard Components

**Report-powered components:**

| Component | Best Used For |
|---|---|
| Chart | Trends, comparisons, proportions (bar, line, pie, donut, funnel, scatter) |
| Gauge | A single metric against a colored range (red/yellow/green) — KPI vs target |
| Metric | A single number prominently displayed — one critical number front and center |
| Table | A list or summary rows from the source report |

**Non-report widgets:**

| Widget | Purpose |
|---|---|
| Rich Text | Free-form text, formatting, links — for labels and section headers |
| Image | Embed a logo or status icon |

> **Critical source report requirement:** Chart, Gauge, Metric, and Table components require a **Summary or Matrix** source report. Tabular reports have no grouping or aggregation, so most component types have nothing meaningful to visualize. A Table component can use Tabular with a row limit set, but this is limited. This is one of the most commonly tested Dashboard facts on the exam.

### Running User — Who Sees What

The running user setting determines what data every Dashboard viewer sees. It answers: "whose sharing context should be used when generating this Dashboard's data?"

| Mode | Running User | Data Shown | Drill-Through | Email Subscriptions |
|---|---|---|---|---|
| Specified User | One fixed user | All viewers see identical data based on that user's access | No | Yes |
| Logged-in User | Each viewer | Each viewer sees their own data | No | Yes |
| Dynamic Dashboard | Each viewer | Each viewer sees their own data | Yes | **No** |

**Specified User:** everyone sees data through one user's lens. If that user has View All, everyone sees everything — regardless of their own access. Used for a consistent "company view." Risk: can expose data to viewers who shouldn't see it.

**Logged-in User:** each viewer sees data based on their own sharing access. A rep sees their pipeline; their manager sees the team's. Fully respects the sharing model per viewer.

**Dynamic Dashboard:** same personalized data as Logged-in User, but additionally enables drill-through — viewers can click a component and see the underlying records in the source report, also filtered by their own sharing access. Two key limitations:
- **Cannot be scheduled for email delivery** — because data is personalized per viewer, Salesforce cannot pre-generate one snapshot to send to all subscribers
- **Edition licensing cap** — the number of Dynamic Dashboards available is capped per Salesforce edition. The exam tests that a limit EXISTS, not the specific numbers

> **The clean mental model:**
> Running user on a Report = always YOU, automatic, no config.
> Running user on a Dashboard = a configurable choice that determines whose report-running context is used to generate the data all viewers see.
> Dynamic Dashboard = Logged-in User mode + drill-through capability. Same sharing rules. Not a mixture of Specified and Logged-in.

### Dashboard Filters

Dashboard Filters let viewers slice the Dashboard's data interactively without editing the Dashboard or running separate reports.

**How they work:**
- Admin adds a filter based on a field that exists across the Dashboard's source reports
- Viewers select filter values from a dropdown while viewing
- The filter dynamically narrows data across all compatible components simultaneously
- Filter selections reset when the viewer navigates away — not saved per user

**Limitation:** only applies to components whose source reports include the filtered field. Components whose source reports don't include the field are unaffected.

### Dashboard Subscriptions

Subscriptions deliver a scheduled email snapshot of a Dashboard to a user's inbox.

**Key behaviors:**
- Any user who can view a Dashboard can subscribe to it
- Schedule options: daily, weekly, or monthly at a specific time
- Salesforce emails a static image snapshot at the scheduled time
- The snapshot reflects the Dashboard's running user setting — not necessarily the subscriber's own access
- Subscribers can set conditions: "only email me if a metric meets a threshold"
- System Administrators can subscribe other users on their behalf
- **Dynamic Dashboards cannot use subscriptions** — personalized data cannot be pre-generated as a single email

### Schedule Refresh

Controls when the Dashboard's underlying data is updated — completely independent from subscriptions.

**Manual refresh:** any user with edit access can refresh on demand. Not admin-only.

**Scheduled refresh:** admins configure automatic refreshes at specific times (e.g. every morning at 6am).

> **The distinction the exam tests:**
>
> | Setting | Controls | Who Configures |
> |---|---|---|
> | Schedule Refresh | When Dashboard data is updated | Admin |
> | Subscription | When a snapshot is emailed to a user | User (or Admin on their behalf) |
>
> These are completely independent. A Dashboard can refresh every morning at 6am AND send weekly email snapshots to subscribers — two separate configurations that don't affect each other.

### Chart Types and Settings

- **Standard chart types:** bar, column, line, pie, donut, funnel, scatter
- **Combination charts:** overlay two chart types on one component (e.g. columns for Amount + line for Count)
- **Axis settings:** min/max values, axis labels, data label visibility
- Dashboard component chart configuration is independent of the source report's embedded chart

### The Dashboard Decision Tree

| Need | Solution |
|---|---|
| Each viewer sees their own data | Dynamic Dashboard (check edition cap) |
| Everyone sees the same data | Specified running user |
| Email snapshots on a schedule | Subscription (not available for Dynamic) |
| Data to update automatically | Schedule Refresh |
| Single KPI number prominently displayed | Metric component |
| Value against a target range | Gauge component |
| Viewers slice data without editing | Dashboard Filter |
| Source report is Tabular | Most component types unavailable — switch to Summary or Matrix |
| Two chart types overlaid in one component | Combination chart |

---

## Bringing It All Together: End-to-End BPO Scenario

Your VP of Client Services wants a Dashboard showing open Case volume by client and SLA compliance rate. Delivery Managers should each see only their own portfolio. It should refresh automatically each morning and send the VP a weekly email summary.

**Objective 1 — Data management:** Cases from a legacy system are imported via Data Loader (Cases aren't supported by Import Wizard). Upsert matches existing Cases by External ID to avoid duplicates from re-imports.

**Objective 2 — Data quality:** a Validation Rule prevents Cases from being saved without a linked Account and SLA type. A Matching Rule + Duplicate Rule combination identifies and blocks duplicate Cases from the same client on the same issue. The Validation Rule formula blocks saves where Status = Closed AND either Resolution is blank OR the Account has no active contract.

**Objective 3 — Reports:** a Summary report on Cases grouped by Account Name (first grouping) and SLA Status (second grouping). A Row-Level Formula calculates days each Case has been open. A Bucket Field groups SLA compliance into "Within SLA," "At Risk," and "Breached" without adding a field to the Case object. A Summary Formula calculates SLA compliance rate per Account group. Report type: "Cases with Accounts" — inner-join ensuring only Cases with linked Accounts appear.

**Objective 4 — Sharing:** each Delivery Manager running the report sees only Cases on Accounts within their sharing access (OWD + Role Hierarchy). The report lives in a "Client Services" folder shared with the Delivery Manager role — not in Unfiled Public Reports.

**Objective 5 — Dashboard:**
- Component 1: Bar chart showing open Case count by Account (Summary report source)
- Component 2: Gauge showing SLA compliance % against a target
- Running user: Dynamic Dashboard — each Delivery Manager sees their own portfolio with drill-through
- Schedule Refresh: every morning at 6am
- VP needs a weekly email: Dynamic Dashboard can't use subscriptions. Solution: a separate non-dynamic Dashboard with a specified running user (high-access exec view) configured with a weekly subscription for the VP

Every objective exercised by one business request. That's why this domain is weighted at 17%.

---

## Quiz — 8 Exam-Style Scenarios with Full Rationale

### Q1 · The Missing Records (Objective 3)

A manager runs "Accounts with Opportunities" with a filter "Stage = Prospecting." Acme Corp doesn't appear at all. Globex Industries appears but its Opportunity columns are blank.

**Answer:**
- **Acme Corp** — has zero Opportunities. The inner-join pattern excludes Accounts with no Opportunity children entirely. Not an access issue — if she couldn't see Acme, it wouldn't appear in any report.
- **Globex Industries** — HAS Opportunities (so it passes the inner join and appears), but none of them are in Prospecting stage. The join determines whether the parent appears; the filter determines which child data shows. These are two separate mechanisms. Globex appears because the join is satisfied; Opportunity columns are blank because no Opportunity passed the filter.
- Neither is a bug. Both are expected behaviors.

### Q2 · The Import Decision (Objective 1)

Operation A: 8,000 Leads, match by email, update matches, create non-matches.
Operation B: 75,000 Opportunities from a legacy CRM.
Operation C: Transfer 340 Accounts and open Activities from a departing rep to her replacement.

**Answer:**
- **Operation A:** Data Import Wizard, **Upsert** operation. Under 50K, Leads are supported, and upsert handles "update matches + create non-matches" in one pass.
- **Operation B:** Data Loader. Two independent reasons: over 50K records AND Opportunities aren't supported by Import Wizard.
- **Operation C:** Mass Transfer Records. This is an ownership transfer, not an import or deletion. Also transfer open Activities along with the Accounts.

### Q3 · The Duplicate Situation (Objective 2)

Matching Rule: Email (exact) + First Name (fuzzy). Duplicate Rule: Allow with Alert. New Contact: Sarah Chen, sarah.chen@globex.com. Existing: Sara Chen, same email.

**Answer:**
- Matching Rule fires — exact email match triggers it; fuzzy match catches Sarah vs Sara.
- Rep sees a warning they can bypass.
- Rep saves anyway — record is saved AND Salesforce creates a **Duplicate Record Set** grouping the two records for admin review.
- Admin looks at **Duplicate Record Sets** (a specific Salesforce object) — not a generic report — to review pairs and decide whether to merge using the native merge tool.

### Q4 · The Validation Rule (Objective 2)

Rule: Case cannot be closed unless Resolution is populated AND the related Account has Active_Contract__c = TRUE.

**Answer:** Lives on the Case object. Formula describes the ERROR condition:

```
AND(
  ISPICKVAL(Status, "Closed"),
  OR(
    ISBLANK(Resolution__c),
    Account.Active_Contract__c = FALSE
  )
)
```

TRUE = Status is Closed AND either Resolution is blank OR Account has no active contract → Save blocked. The OR is inside the error condition because "both must be true to allow save" means "block if either is false." This cross-object reference to Account.Active_Contract__c is valid syntax on a Case rule.

### Q5 · The Report Format Decision (Objective 3)

**Report A** — list of open Cases last 7 days with Case number, subject, owner, Account name.
**Answer:** Tabular. Simple list, no grouping needed.

**Report B** — total Case volume grouped by Account with subtotals and grand total.
**Answer:** Summary. Single row grouping with subtotals.

**Report C** — Case volume by Account (rows) and Case Priority (columns).
**Answer:** Matrix. Two-dimensional grouping — the canonical Matrix use case.

**Report D** — open Opportunities AND open Cases for the same Accounts side by side.
**Answer:** Joined. Two separate report types (Opportunities and Cases) connected via a common object (Account). Each block has its own filters.

### Q6 · The Dashboard Problem (Objective 5)

Issue 1: Dashboard shows Friday's numbers on Monday morning.
Issue 2: Diego sees the same pipeline as Carlos including deals he shouldn't see.
Issue 3: VP subscribed for weekly Monday email but never receives it. Dashboard is Dynamic.

**Answer:**
- **Issue 1:** Dashboard data hasn't been refreshed since Friday. Immediate fix: manual refresh (any user with edit access, not admin-only). Preventive fix: configure Schedule Refresh to run automatically each morning.
- **Issue 2:** Running user is set to a Specified User (likely Carlos or a high-access user). Everyone sees that user's data. Fix: change to Logged-in User or Dynamic Dashboard so each viewer sees their own sharing-appropriate data.
- **Issue 3:** Dynamic Dashboards cannot use email subscriptions — personalized per-viewer data cannot be pre-generated as a single email snapshot. Fix: create a separate non-dynamic Dashboard with a specified running user configured for the VP's perspective, then set up a subscription on that Dashboard.

### Q7 · Bucket Field vs Formula Field (Objective 3)

Both Colleague A (formula field on object) and Colleague B (Bucket Field in report) produce a Small/Medium/Large tier column based on Amount.

**Answer:**
- **Colleague A is right when:** the tier needs to appear on record pages, drive automation, be consistent across multiple reports, or appear in list views and filters. The field has a permanent role beyond this one report.
- **Colleague B is right when:** the tier is needed for this one report or analysis only. No schema change needed, no other system needs this categorization.
- **Scenario where BOTH are needed:** the formula field serves standard tier definitions for automation and record display. The Bucket Field in a specific report uses different tier boundaries for a one-off executive analysis. Same concept, different definitions, serving different purposes simultaneously.

### Q8 · Sharing and Report Visibility (Objective 4)

Setup: Opportunity OWD Private. Report in Unfiled Public Reports. Built by VP (View All on Opportunities). Diego (rep) sees 47 Opps. Carmen (manager) sees 89 Opps. Marcus (contact center agent) sees 12 Opps.

**Answer:**
- **Diego vs Carmen:** OWD Private means each sees records based on their own sharing access. Carmen is above Diego in the Role Hierarchy — she sees his records plus all reps rolling up to her. Role Hierarchy flows ownership visibility upward.
- **Marcus seeing anything:** Unfiled Public Reports gives any internal user the ability to run the report — but what they see is still filtered by their own sharing access. Marcus can run it (folder access) but sees only 12 Opportunities his sharing access allows. Folder access and record-level access are independent.
- **Security risk and fix:** the report is in Unfiled Public Reports — any internal user can run it, potentially exposing pipeline data to users who have some Opportunity access they shouldn't. Fix: move the report to a properly permissioned folder immediately. Audit Unfiled Public Reports regularly as admin hygiene.
- **VP as specified running user on Dashboard:** Diego sees everything the VP can see — all Opportunities, bypassing his own Private OWD constraints. The specified running user's access completely replaces Diego's own sharing context for Dashboard viewing. This is the exact risk of using a high-access specified running user without considering who views the Dashboard.

---

## The 15 Most Exam-Likely Facts

1. **Data Import Wizard: 50K limit, specific objects only, no install, no scheduling**
2. **Data Loader: 5M records, ALL objects, must install, supports Hard Delete and Export All**
3. **Upsert = insert new + update existing using an External ID field to match**
4. **Mass Delete Records is a UI Setup tool — filter-driven, no CSV needed, specific objects**
5. **Validation Rules fire for EVERYONE — including admins — unless explicitly excluded in the formula**
6. **Validation Rule formula describes the ERROR condition — TRUE blocks the save**
7. **Matching Rule finds duplicates; Duplicate Rule decides what to do — both required; saves despite alerts create Duplicate Record Sets**
8. **Bucket Fields produce a column in the report without touching the object schema**
9. **Summary Formula = group/aggregate level; Row-Level Formula = individual record level — both produce report columns only**
10. **Tabular reports cannot drive most Dashboard components — need Summary or Matrix**
11. **Sharing affects report results — the running user's access is the invisible first filter before any explicit filters run**
12. **Unfiled Public Reports is accessible to ALL internal users — folder access and record-level access are independent**
13. **Dynamic Dashboards respect sharing 100% and add drill-through — but cannot be scheduled for email delivery and have an edition cap**
14. **Schedule Refresh (when data updates) and Subscriptions (when emails send) are independent settings**
15. **Missing field in a report: check FLS first, then Custom Report Type field layout, then relationship chain depth**
