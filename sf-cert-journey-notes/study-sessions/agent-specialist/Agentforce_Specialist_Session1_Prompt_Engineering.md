# Session 1 — Prompt Engineering

> **Salesforce Agentforce Specialist Cert (AI-201) — Study Guide Series**
> Captured deep-dive session: concept explanations, mental-model clarifications from Q&A, and a worked quiz with full answer rationale. Read the concepts, then test yourself against the quiz before reading the graded answers.

---

## Exam Weight: 20%

### Objectives Covered
- Given business requirements, identify when it's appropriate to use Prompt Builder
- Identify access controls governing prompt templates
- Identify the considerations for using a prompt template type such as field generation and flex types
- Given a scenario, identify the appropriate grounding technique
- Explain the process for creating, activating, and executing prompt templates
- Explain how to implement best practices for writing effective prompts
- Identify the security and privacy features of the Trust Layer
- Explain how to manage and prevent specific models from being accessed

---

## The Core Mental Model

Before any mechanics, lock in this definition:

> **Prompt Builder is a tool for creating structured, reusable, templated AI prompts that humans or automated processes trigger to produce a defined output using Salesforce data.**

Every word in that sentence matters:

- **Structured and reusable** — not a one-off interaction. A template used repeatedly.
- **Humans or automated processes trigger it** — a button click, a Flow action, a page component. Nothing autonomous.
- **Defined output** — you know what you want back before you build the template.
- **Using Salesforce data** — grounded in your org's actual records.

### The Core Decision Triangle

The exam tests three tools against each other in scenario questions. Know their lanes:

| Tool | When to Use |
|---|---|
| **Prompt Builder** | Repeatable, record-based, single output, triggered by a user or process |
| **Agent** | Autonomous, multi-step, reasoning-based, conversational |
| **Flow** | Structured automation, deterministic logic, no AI generation needed |

These are not mutually exclusive. A Flow can *call* a Prompt Builder template as an action. An Agent can *use* a Prompt Builder template inside a custom action. But each has a primary lane.

**The exam line:** If the AI needs to *decide what to do next* → Agent. If it needs to *generate a defined output from structured data* → Prompt Builder. If the logic is deterministic and requires no AI → Flow.

---

## Objective 1: When to Use Prompt Builder

### Signal Words in Correct Scenarios
Repeatable · Record-based · Single output · Triggered by user or process

### Classic Prompt Builder Use Cases

**Case Resolution Summary** — After each call, agents complete a "Next Steps" field on the Case record. AI suggests the content based on case subject, description, and priority. User clicks a button, reviews the suggestion, saves. → Prompt Builder. Repeatable, record-based, single field output, user-triggered.

**Field Auto-Population via Flow** — Product descriptions auto-populated on Product records when created via a Flow. → Prompt Builder. Repeatable, automated, defined field output.

**Knowledge Article Draft** — A knowledge article draft generated from a closed case's details for the knowledge base team. → Prompt Builder. Structured, record-based, single output.

### What Prompt Builder is NOT For

**Multi-step customer interaction** — A customer wants to talk to a bot that checks account status, looks up orders, escalates if needed, and sends a follow-up email. → Agent. Multi-step, autonomous, conversational.

**Cross-system routing logic** — Automatic case routing based on sentiment analysis checking multiple systems. → Agent or Flow.

**The exam trap:** Scenarios that sound simple but require decision-making across steps. If the AI needs to decide what to do next rather than just generate an output, it is not Prompt Builder.

---

## Objective 2: Access Controls

Access to Prompt Builder is layered. The exam tests all four layers independently and in combination.

### Layer 1 — Feature-Level Permission Sets

> **Terminology note:** "Feature-level" is a general Salesforce concept, not Agentforce-specific. These are standard Salesforce-delivered permission sets you assign like any other — similar to "Marketing User" or "Knowledge User." They exist in the org once Prompt Builder is enabled.

Two permission sets control Prompt Builder access:

| Permission Set | What It Grants |
|---|---|
| **Prompt Template Manager** | Create, edit, activate, deactivate templates — the builder role |
| **Prompt Template User** | Execute (run) prompt templates — the end-user role |

These are **distinct and non-overlapping by default.** A Manager can build and activate but cannot necessarily execute. A User can run templates but cannot see or edit them in Prompt Builder.

**Special case:** Users need **Data Cloud Admin** permissions to access the View This Model link in Template Settings and modify model configurations in AI Models. Model management is gated behind a separate elevated permission — not covered by Manager or User permission sets alone.

### Layer 2 — Object and Field-Level Security (FLS)

This is where most study materials stop short. When a prompt template executes, it runs **in the context of the user executing it** — not the admin who built it. This means:

- If the template references Account fields and the running user doesn't have Read access via FLS, those fields return blank or the template errors.
- If the template references a related list of Cases and the user's profile doesn't grant Case Read, the related list returns nothing.
- Sharing rules apply. A user cannot pull data from records they don't have access to, even inside a prompt template.

**Exam scenario signal:** A template works for some users but not others → the answer is almost always FLS or sharing, not a problem with the template itself.

### Layer 3 — Template Activation State

A template must be **Active** before any user can execute it. An inactive template is invisible to end users regardless of their permissions. Activation requires Prompt Template Manager permission.

**Key distinction:** Activation and deployment are separate steps. You can activate without deploying (template is active but not on any page — users can't reach it). You can deploy without activating (template is on a page but users get an error trying to run it). **Both must be true for a user to successfully execute a template.**

### Layer 4 — Template Versioning and Immutability

**Activating a template version makes it immutable.** Even if deactivated afterward, any version that was ever active remains immutable. To make changes after activation, you must save as a new version.

**Exam trap:** Questions asking how to modify a deployed template. The answer is always "create a new version" — never "edit the existing one."

---

## Objective 3: Prompt Template Types

There are **three template types** in Prompt Builder. The exam outline mentions two by name but all three appear in scenarios.

### Field Generation Templates

**What it does:** Returns an AI-generated response directly into a specific field on a specific record.

**Architecture:**
- Bound to **one object** and **one target field** — both chosen during template creation
- Input comes only from that object and its related data — cannot pull from multiple unrelated objects
- Triggered by an **Einstein button** placed on the record's Lightning page via Lightning App Builder
- The Einstein button is simply a star icon that appears next to the target field — it's how the user triggers the template from the record
- Output goes directly into the target field — user reviews, edits if needed, saves

**BPO Example:**
You're running a collections BPO. After every call, agents must fill in a "Case Resolution Summary" field on the Case record. You build a Field Generation template bound to Case → Case Resolution Summary field. Merge fields pull Case Subject, Description, and Priority. When the agent clicks the Einstein button next to the summary field, AI generates a professional, consistent summary directly into it.

**Use case anchor:** Any time you want to auto-populate a specific field using AI.

### Flex Templates

**What it does:** Returns AI-generated text output for the user or process to use however needed — not bound to any field.

**Architecture:**
- Not bound to a specific field or a single object
- You define your own inputs during template creation — up to **five inputs** (records, free text fields, or data model objects)
- Output is returned as text — the user decides what to do with it
- As of 2026, Flex is the dominant template type by volume — every custom Agentforce action you build will in practice use a Flex template

**BPO Example:**
Same BPO. QA team needs coaching notes for agents that reference the Case details, the Account history, and the agent's Performance record — three different objects. A Field Generation template can't do this (single object bound). A Flex template takes all three as inputs, generates a coaching note, returns it as text for the QA analyst to copy or send directly.

**Use case anchor:** Multiple objects, free text inputs, flexible output, or any case that doesn't fit the single-object/single-field pattern.

### Standard Templates

**What it does:** Pre-built by Salesforce for specific product features — use case-specific responses within a defined scope.

**Architecture:**
- Already exist in the org for features like Work Summary, Sales Email, Sales Pitch Coaching
- You can configure and adjust prompt instructions inside them but cannot change the template architecture
- Not customizable in structure — only content adjustments allowed

**BPO Example:**
You're using Salesforce's native Work Summary feature for Service Cloud. After a service interaction, Salesforce auto-generates a work summary using a Standard template Salesforce pre-built for that exact purpose. You didn't build it — you configure it.

**Use case anchor:** When the use case matches a standard Salesforce product feature. If a Standard template already does what you need, use it rather than building a custom one.

### Template Type Decision Framework

| Question | Answer |
|---|---|
| Does the output go into a specific field on a specific object? | Field Generation |
| Does the output serve a standard Salesforce product feature (Work Summary, Sales Email)? | Standard |
| Everything else — multiple objects, free text inputs, flexible output? | Flex |

---

## Objective 4: Grounding Techniques

**Grounding is always the goal** — giving the LLM accurate, current, contextual data so it doesn't generate generic or hallucinated output. Without grounding, you're asking the LLM to answer with no knowledge of your business, your customers, or your data.

> **Mental model:** Grounding = the goal. The five techniques = the delivery methods. The technique you choose depends entirely on where the needed data lives.

### Technique 1 — Static Text (System Instructions)

Text hardcoded directly into the prompt template that doesn't change at runtime. Sets persona, tone, constraints, and organizational context.

**Example:** *"You are a professional customer service representative for Acme Corp. Always respond in a polite, concise tone. Never make promises about refunds without manager approval."*

- Doesn't pull any Salesforce data — just sets the frame
- Always present alongside dynamic grounding in a well-built template
- Rarely the sole correct answer on an exam — always combined with another technique
- **Use when:** You need consistent instructions, persona definition, or guardrails that don't change per record

### Technique 2 — Merge Fields (Dynamic Grounding)

The most common grounding technique. Pull live Salesforce field values into the prompt at runtime using merge fields from the resource picker.

**Example:** *"The customer's name is {!Contact.Name}. Their case subject is {!Case.Subject}. Their account tier is {!Account.Type}."*

At execution, merge fields are replaced with actual values from the record the user is viewing. Every execution is personalized to that specific record.

**Key nuances:**
- Available merge fields are determined by the template's object binding and the running user's FLS
- Inserted via the **resource picker** in Prompt Builder — not typed manually
- If FLS blocks a field, it comes back blank in the prompt — the template doesn't error, it just has no data for that field
- **Use when:** The needed context is a field value on the current Salesforce record

### Technique 3 — Related List Grounding

An extension of merge fields for collections of related records rather than single field values.

**Example:** Including the last five Case Comments on a Case prompt so the LLM understands full interaction history before generating a resolution summary.

**Key nuances:**
- Related lists are passed to the LLM as **JSON** — the LLM parses the structure automatically
- Very large related lists can exceed token limits — a consideration for complex implementations
- Available related lists come from the page layout of the parent object
- **Use when:** The scenario requires historical or contextual data from connected records, not just fields on the primary record

### Technique 4 — Apex Grounding

Advanced grounding for when standard merge fields and related lists aren't enough. A custom Apex class handles data retrieval and formatting before the prompt executes.

**Apex grounding can:**
- Run complex SOQL queries joining multiple objects declaratively unavailable in the resource picker
- Call external APIs to pull real-time data (inventory levels, shipping status, third-party CRM data)
- Transform and format data in a specific structure before it reaches the LLM

**Key nuances:**
- Requires a developer to write and deploy the Apex class first — not declarative
- The class appears in the resource picker as "Apex: ClassName" once deployed
- **Use when:** The scenario describes complex data requirements, external system data, or custom formatting logic that standard tools can't provide

> **Note:** Apex grounding in Prompt Builder is one use of Apex in Agentforce — not the only one. Apex also appears heavily in the AI Agents section (Session 3) for building custom Agent Actions via @InvocableMethod.

### Technique 5 — Retrieval Augmented Generation (RAG)

Grounds the prompt in **unstructured data** — documents, knowledge articles, PDFs, and other content stored in the Agentforce Data Library. Instead of pulling structured field values, RAG searches a content corpus for the most relevant chunks and injects them into the prompt as context.

**Example:** A service agent prompt that needs to reference your company's product documentation before generating a troubleshooting recommendation. The template searches the knowledge base, retrieves the most relevant article chunks, and injects them.

**Key distinction:**
- RAG is for **unstructured knowledge content** — documents and articles, not record fields
- If the data lives in a document or knowledge article rather than an object field, RAG is the answer
- RAG is the primary mechanism behind the Agentforce Data Library (covered in depth in Session 2 — Data 360 Fundamentals)

> **Note:** RAG appears here in the Prompt Builder context and expands significantly in Session 2. What you've learned here is the foundation for that section — the full mechanics of chunking, indexing, and retrievers are Session 2 territory.

### Grounding Technique Decision Framework

| Scenario Signal | Correct Technique |
|---|---|
| Need to set persona, tone, or org-specific rules | Static text |
| Need field values from the current record | Merge fields |
| Need history or context from related records | Related list grounding |
| Need complex SOQL, external system data, or custom data transformation | Apex grounding |
| Need to search unstructured documents or knowledge content | RAG |

---

## Objective 5: Creating, Activating, and Executing Prompt Templates

The full lifecycle has six stages. The exam tests the sequence and the gate conditions between stages.

### Stage 1 — Enable Prompt Builder

Admin must enable Prompt Builder in Setup before anyone can build. Org-wide prerequisite. Rarely the direct answer to exam questions but appears in troubleshooting scenarios — "users can't access Prompt Builder" → check if it's enabled first.

### Stage 2 — Build

In Prompt Builder: select template type, bind to object and field (Field Generation only), write the prompt with merge fields and static instructions, select grounding resources via the resource picker, and select the LLM configuration (which model the template will use).

### Stage 3 — Preview and Test

Prompt Builder has a built-in preview mode. Select a sample record from your org, run the template against it, review the output before activating. This is a documented step in the process — **testing happens before activation, not after.** The exam will test this sequence.

### Stage 4 — Activate

- Requires **Prompt Template Manager** permission
- Activation locks that version — **immutable from this point forward**
- To make changes after activation: create a new version
- Only the **active version** is used when the template executes
- Switching active versions: deactivate current, activate the new version

### Stage 5 — Deploy

Making the template available to users. Method depends on template type:

| Template Type | Deployment Method |
|---|---|
| Field Generation | Einstein button on Lightning record page via Lightning App Builder |
| Flex (user-triggered) | Component on Lightning page or custom button |
| Flex (automated) | Flow action |
| Flex (agent action) | Custom Agentforce agent action |

**Deployment is independent of activation.** A template can be active but not deployed (can't reach it), or deployed but not active (errors when run). **Both must be true.**

### Stage 6 — Execute

A user or automated process runs the template. Requires **Prompt Template User** permission. The Trust Layer processes the prompt before it reaches the LLM. Output is returned and surfaced to the user or injected into the target field.

### Why Can't This User Run the Template? — The Three Most Common Exam Answers

1. Template is not Active
2. User lacks Prompt Template User permission set
3. User lacks FLS access to a field the template references

---

## Objective 6: Best Practices for Writing Effective Prompts

The exam tests this conceptually — you identify the better approach in a scenario, not write a prompt. Understand the *why* behind each principle well enough to reason about it.

### Establish Persona and Role Before the Task

The LLM needs to know who it's supposed to be before it can behave consistently. Skipping this produces generic output that doesn't reflect your brand or context.

**Strong:** *"You are a professional support agent for a B2B SaaS company serving enterprise clients."*
**Weak:** No persona instruction — LLM defaults to generic assistant behavior.

### Provide Context Before the Ask

Structure is: who you are → what context you have → what you need. Don't lead with the task. The LLM uses earlier content in the prompt to interpret later instructions. Ground before you ask.

### Use Affirmative Instructions, Not Negations

LLMs respond to what you tell them to do, not what you tell them to avoid. Negations create ambiguity.

**Strong:** *"Respond in a professional, empathetic tone."*
**Weak:** *"Don't be unprofessional."*

### Specify Output Format Explicitly

*"Write a two-paragraph response"* or *"Respond in three bullet points starting with an action verb"* produces consistent, usable output. Unformatted prompts produce inconsistently structured responses that vary across executions.

### One Task Per Template

If you need a case summary AND a follow-up email draft, build two templates — not one. Multi-task prompts produce diluted output as the LLM splits attention across objectives.

> **Framing note:** The correct mental model is "one task per template" — you're building the template, so the template gets one task. Not "one template per task."

**BPO Example:**
A collections BPO builds one Flex template called "Collections Outreach" with instructions to generate both a customer-facing payment reminder email AND an internal risk assessment for the agent. Output quality is inconsistent — sometimes the email is good but the risk summary is vague, sometimes the reverse.

**Root cause:** The LLM is splitting attention between two tasks with different tones, audiences, and formats. Customer-facing email requires a professional tone. Internal risk summary requires an analytical tone. One prompt can't reliably serve both.

**Fix:** Two templates — "Collections Email Draft" (customer-facing) and "Agent Risk Summary" (internal analytical). Each produces consistent output because it has one job.

### Define Fallback Instructions for Missing Data

Your template will run against records with blank fields. Include fallback instructions:

*"If the Account Tier field is blank, treat the customer as a standard tier account."*

Without this, blank merge fields produce unexpected or inconsistent output.

### Build-Stage vs. Runtime

All best practices above except "One Task Per Template" are build-stage considerations — decisions made when designing the template. "One Task Per Template" is an architectural decision that affects how you scope templates, but it also shapes runtime consistency.

---

## Objective 7: The Einstein Trust Layer — Security and Privacy

The Trust Layer is Salesforce's security and compliance wrapper around **every single LLM interaction** — whether triggered by Prompt Builder, an Agent, or the Models API. Nothing bypasses it.

> **One-sentence mental model:** Data goes out masked and defended, nothing is retained externally, output comes back scanned and demasked, everything is logged internally.

The Trust Layer has two distinct journeys.

### The Prompt Journey (Outbound — Data Leaving Salesforce)

**Step 1 — Secure Data Retrieval**
Salesforce fetches the data needed to ground the prompt. This retrieval **respects the running user's permissions** — sharing rules, FLS, and object permissions all apply. The Trust Layer cannot be used to bypass Salesforce security.

**Step 2 — Dynamic Grounding**
The retrieved data is merged into the prompt template, personalizing it to the specific record and user context.

**Step 3 — Data Masking**
Before the prompt leaves Salesforce, the Trust Layer scans for PII and sensitive data using pattern-based detection (regular expressions). Detected PII — names, SSNs, email addresses, phone numbers, dates of birth — is replaced with masked placeholders. The LLM never sees the actual sensitive values.

> **Critical exam flag: Data masking for LLMs is currently disabled for agents.** Data masking applies to Prompt Builder-triggered prompts but NOT to agent interactions. This is a tested distinction. If an exam question asks about data security in an agent context, masking is not currently a protection — this is a known limitation.

**Step 4 — Prompt Defense**
System-level guardrails applied to limit hallucinations and prevent prompt injection attacks — where malicious content embedded in a record field tries to override the prompt instructions.

**Step 5 — Zero Data Retention**
Salesforce has a contractual zero data retention policy with all external LLM providers (OpenAI, Anthropic, etc.). Data sent to the LLM is not retained, not stored, and not used for model training after the response is returned. The LLM provider deletes it immediately after processing.

### The Response Journey (Inbound — Data Returning to Salesforce)

**Step 1 — Toxicity Detection**
The LLM's response is scanned before it reaches the user. Content flagged as harmful, biased, offensive, or inappropriate is blocked. Applies to both the input prompt content and the generated response.

**Step 2 — Data Demasking**
Masked placeholders in the response are replaced with the original actual values. This restores personalization — the LLM worked with masked data, but the user sees the real data in the final response.

**Step 3 — Feedback Framework**
Users can provide thumbs up/down feedback on AI responses. This data stays within Salesforce for internal quality monitoring — it does not go back to the LLM provider.

**Step 4 — Audit Trail**
Every prompt interaction is logged. The log captures what was sent, what was returned, who triggered it, when it ran, and the toxicity score. This is the compliance and accountability mechanism. Available for review and analysis within Salesforce.

### Trust Layer Quick Reference

| Feature | Direction | What It Does |
|---|---|---|
| Secure Data Retrieval | Outbound | Fetches grounding data respecting user permissions |
| Dynamic Grounding | Outbound | Merges Salesforce data into the prompt |
| Data Masking | Outbound | Replaces PII with placeholders before LLM sees it |
| Prompt Defense | Outbound | Guards against hallucinations and prompt injection |
| Zero Data Retention | Outbound | LLM provider cannot store or train on your data |
| Toxicity Detection | Inbound | Scans response for harmful content before user sees it |
| Data Demasking | Inbound | Restores real values from masked placeholders |
| Feedback Framework | Inbound | Captures user feedback, stays within Salesforce |
| Audit Trail | Inbound | Logs every interaction for compliance |

---

## Objective 8: Managing and Preventing Model Access

This is managed through **AI Models** (formerly Einstein Studio) — the admin console for configuring which LLMs are available in your org.

### The Model Landscape

**Salesforce Default (Managed Mix):** A Salesforce-managed mix of trusted models currently including GPT-4o, optimized for accuracy, trust, and performance. This is the default for Agentforce. Salesforce manages the mix and can update it.

**AWS-Hosted Option:** Anthropic Claude Sonnet 4 on Amazon Bedrock. Available as an alternative — not the default.

> **Exam note:** Do not assume Claude is the default model. GPT-4o is currently in the default managed mix. Claude on Bedrock is the AWS-hosted alternative. This distinction is testable.

**BYOLLM (Bring Your Own LLM):** Organizations can connect external LLM providers using their own accounts via the LLM Open Connector API specification. The model is saved to the Model Library in AI Models as a foundation model that can be configured and tested in Model Playground before production use.

### Hiding a Model to Prevent Access

An LLM configuration can be **hidden** in AI Models. Hiding a model:

- Prevents it from being selected for new or edited prompt templates
- **Does NOT break existing active templates** — templates already using a hidden model continue to run
- If you open a template that uses a hidden model, Prompt Builder requires you to select a different model before you can save

**To fully remove a model from use:** Hide it in AI Models AND update or deactivate any templates currently referencing it.

**Exam scenario signal:** "Prevent agents from using a newly added third-party LLM without breaking existing templates" → Hide the model in AI Models and update any existing templates that reference it.

### Permission Layer for Model Management

Users need **Data Cloud Admin permissions** to access model configuration in AI Models. Prompt Template Manager permission alone does not grant this. Template managers can *select* from available models during template creation but cannot *modify* model configurations.

### BYOLLM Deployment Gotcha

When deploying prompt templates that reference a BYOLLM from sandbox to production, the LLM configuration must exist in the target org with the **exact same name and identifier**. If the configuration is missing or named differently in production, the deployment fails. This is a tested deployment scenario.

---

## Section Summary — Five Things That Win Exam Questions

**1. Prompt Builder produces single, templated, triggered outputs.** If the scenario requires autonomy or multiple steps, it is not Prompt Builder.

**2. Access is layered.** Feature permissions, object/FLS permissions, activation state, and versioning all act as independent gates. A failure at any layer blocks execution.

**3. Template type selection follows one rule.** Does the output go into a specific field? → Field Generation. Does it serve a standard Salesforce feature? → Standard. Does everything else apply? → Flex.

**4. Grounding technique selection follows where the data lives.** Record fields → Merge fields. Related records → Related list. Complex logic or external data → Apex. Unstructured documents → RAG.

**5. The Trust Layer processes every LLM interaction in two directions.** Data masking applies to Prompt Builder but NOT to agents — this is a tested distinction. ZDR means nothing is retained by external LLMs. Everything is logged via audit trail.

---

## Quiz — Session 1

> Read each question fully. Write your answer and reasoning before checking the graded section.

---

**Question 1**

A BPO managing a telecom client wants to improve consistency in how agents close cases. After each call, agents must complete a "Next Steps" field on the Case record. The BPO wants AI to suggest the next steps automatically based on the case subject, description, and priority. The suggested content should appear directly in the field for the agent to review and save.

Which solution should the administrator configure?

- A) Build a Flex template in Prompt Builder with Case as the primary input and deploy it via a Flow action
- B) Build a Field Generation template in Prompt Builder bound to the Case object and the Next Steps field, deployed via Lightning App Builder
- C) Build an Agentforce agent that reads the case details and populates the Next Steps field autonomously
- D) Build a Standard template in Prompt Builder for case resolution and assign it to the Case page layout

---

**Question 2**

A Prompt Template Manager at a financial services BPO has built and activated a Field Generation template that generates account risk summaries. After activation, the compliance team requests a small change to the prompt wording. The admin opens the template in Prompt Builder to edit it but finds the wording cannot be changed.

What is the correct explanation for this behavior and the correct resolution?

- A) The template is locked because it has been deployed to a Lightning page. The admin must remove it from the page before editing.
- B) Activated template versions are immutable. The admin must create a new version of the template, make the changes, and activate the new version.
- C) Only users with the Data Cloud Admin permission can edit activated templates. The admin should request elevated permissions.
- D) The template is locked because it is currently being used in production. The admin must deactivate it, make changes, then reactivate.

---

**Question 3**

A collections BPO wants to use Prompt Builder to generate personalized payment reminder emails for customers. Each email should reference the customer's account name, their outstanding balance, and a list of their last three invoices. The balance is a field on the Account object. The invoices are child records on the Account.

Which grounding approach should the template use?

- A) Static text grounding only, with the account name and balance hardcoded into the template instructions
- B) Merge fields for the account name and balance, and related list grounding for the last three invoices
- C) RAG grounding using the Agentforce Data Library to retrieve the invoice history
- D) Apex grounding to retrieve both the account balance and invoice records through a SOQL query

---

**Question 4**

A healthcare BPO has deployed Prompt Builder templates to help agents generate patient follow-up summaries. The compliance officer raises a concern — patient names, dates of birth, and social security numbers are being referenced in the case records used to ground the prompts. She wants to confirm that this sensitive data is not being sent to the external LLM in clear text.

Which Trust Layer feature directly addresses this concern?

- A) Zero Data Retention, which ensures the LLM deletes all data immediately after generating a response
- B) Toxicity Detection, which scans the prompt for sensitive data before it reaches the LLM
- C) Data Masking, which detects and replaces PII with masked placeholders before the prompt is sent to the LLM
- D) Prompt Defense, which prevents sensitive fields from being added to prompt templates via the resource picker

---

**Question 5**

A Salesforce administrator at a BPO has built a Flex template to generate coaching notes for QA analysts. The template has been activated. A QA analyst reports they cannot see any option to run the template on the Case record page.

After investigation, the administrator confirms the template is active. What are the TWO most likely causes of this issue?

- A) The template has not been deployed to the Case Lightning record page
- B) The QA analyst does not have the Prompt Template User permission set assigned
- C) The QA analyst does not have the Prompt Template Manager permission set assigned
- D) The template is a Flex type and cannot be deployed to Lightning record pages
- E) The Case object does not support Prompt Builder templates

---

**Question 6**

A BPO administrator is building a Prompt Builder template to generate escalation recommendations for complex support cases. The recommendation needs to reference the case details, the customer's full interaction history pulled from an external CRM system, and a calculated risk score that requires logic across five custom objects. No standard merge fields or related lists can provide this data in the required format.

Which grounding technique is most appropriate?

- A) Related list grounding using the Case's related Interaction records
- B) RAG grounding using the Agentforce Data Library to index the external CRM data
- C) Apex grounding using a custom Apex class to query the required objects, call the external system, and format the data
- D) Static text grounding with instructions telling the LLM to assume standard escalation criteria

---

**Question 7**

A Salesforce admin at a BPO has been asked to prevent agents from using a newly connected third-party LLM that was added by a developer last week. Existing prompt templates using the company's standard Salesforce-managed model should continue to run without interruption.

What is the correct approach?

- A) Delete the third-party LLM configuration from AI Models to prevent any templates from using it
- B) Hide the third-party LLM configuration in AI Models so it cannot be selected for new templates, and update any existing templates that reference it
- C) Deactivate all prompt templates that use the third-party LLM and reassign them to the Salesforce default model
- D) Remove the Prompt Template Manager permission from all users so they cannot select the third-party model when building templates

---

**Question 8**

A QA manager at a customer service BPO reviews AI-generated coaching notes and notices the output quality is inconsistent. Some responses include a professional email draft to the agent AND an internal risk assessment, while others only include one or the other. The manager wants reliable, consistent output every time.

A review of the template shows the prompt instructs the LLM to "generate a coaching email for the agent and provide an internal risk assessment of the case."

What is the most likely cause and the recommended fix?

- A) The template is using the wrong LLM model. Switch to the AWS-Hosted Claude Sonnet option for better consistency.
- B) The template lacks sufficient static grounding instructions. Add more detailed persona and tone instructions to stabilize the output.
- C) The template is attempting two distinct tasks in a single prompt. Split into two separate templates — one for the coaching email and one for the risk assessment.
- D) The Flex template needs additional inputs. Add the agent's performance record as a fifth input to give the LLM more context for both outputs.

---

## Quiz — Graded Answers

**Q1 — B ✅**
Repeatable, single field target, one object (Case), user-triggered. All four Prompt Builder signals present. A is wrong because Flex doesn't bind to a specific field — Field Generation is required when the output must go into a defined field. C is wrong because an agent is autonomous and multi-step; this is a simple, repeatable, single-output task. D is wrong because Standard templates exist for specific Salesforce product features — there is no Standard template for generic case next steps.

**Q2 — B ✅**
Activated template versions are immutable — this is Access Layer 4. Deactivating does not unlock the template for editing (eliminating D). Deployment to a Lightning page has no effect on editability (eliminating A). Data Cloud Admin permissions govern model configuration access, not template editing (eliminating C). The only path forward is creating a new version.

**Q3 — B ✅**
The data is entirely within Salesforce structured records — no unstructured documents involved (eliminating C/RAG). The balance is a single field value on Account → merge field. The invoices are child records on Account → related list grounding. Apex is unnecessary because the data is accessible through standard declarative tools (eliminating D). Static text cannot pull live data (eliminating A).

**Q4 — C ✅**
Data Masking is specifically designed to detect and replace PII before the prompt reaches the LLM — this is the direct answer. Zero Data Retention (A) addresses what happens *after* the LLM receives the data, not before. Toxicity Detection (B) scans for harmful content in responses, not PII in prompts. Prompt Defense (D) guards against prompt injection, not sensitive field exposure.

**Important addition:** This question assumes Prompt Builder context. Data masking is currently disabled for agents — if this were an agent scenario, masking would not apply and the answer would change.

**Q5 — A and B ✅**
Two independent gates must both be open for a user to execute a template: it must be deployed to the page they're on (A), and they must have the Prompt Template User permission set (B). C is wrong because Manager permission is for building, not executing. D is wrong because Flex templates can be deployed to Lightning pages. E is wrong because Case is a fully supported object.

**Q6 — C ✅**
Three requirements disqualify the other options: complex multi-object logic, external CRM data via API call, and custom formatting. RAG (B) is for unstructured content like documents — not structured external CRM data queried in real time. Related list grounding (A) only works with native Salesforce related records — not external systems. Static text (D) cannot pull any live data. Apex is the only technique that handles complex SOQL, external API calls, and custom data formatting in combination.

**Q7 — B ✅**
Hiding prevents new templates from selecting the model without breaking existing active templates (which continue running). Deleting (A) would break any existing templates already referencing that model. Deactivating all templates (C) is disruptive and unnecessary — the goal is prevention of new use, not breaking current deployments. Removing Manager permission (D) prevents users from building any templates at all — far broader than required.

**Q8 — C ✅**
One task per template. The prompt is trying to produce two outputs with different tones, audiences, and formats simultaneously — a customer-facing email and an internal analytical note. The LLM splits attention and produces inconsistent results. The fix is architectural: two separate templates, each with one job. Model selection (A) doesn't solve a prompt architecture problem. More persona instructions (B) don't fix the fundamental multi-task conflict. Adding more inputs (D) gives more data but doesn't resolve the dual-task output problem.

---

*Session 2 — Data 360 Fundamentals (20%) →*
