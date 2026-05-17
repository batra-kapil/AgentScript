# AgentScript — IT Support Agent with Salesforce Agent Script

> Built by [Kapil Batra](https://www.linkedin.com/in/kapilbatra) | [SF Bolt YouTube Channel](https://www.youtube.com/@sfbolt)

This repository contains the complete source code for the **IT Support Agent** built using **Salesforce Agent Script** — demonstrated in the SF Bolt video series on Agentforce development.

---

## What Is This?

This project demonstrates how to build a predictable, enterprise-grade Agentforce agent using **Agent Script** — Salesforce's pro-code scripting language for AI agents announced at Dreamforce 2025 (public beta November 2025).

Agent Script preserves the conversational intelligence of LLM-based agents and adds the predictability of programmatic execution. You define exactly which parts of your workflow execute deterministically and which parts leverage LLM reasoning. The result is **hybrid reasoning** — strict business logic where you need it, intelligent flexibility where you want it.

### What the Agent Does

The IT Support Agent handles three support scenarios for Acme Corp employees:

| Scenario | Behaviour |
|---|---|
| **Password Reset** | Verifies identity → checks MFA status → resets only after explicit YES confirmation |
| **Software Access** | Verifies identity → routes automatically for Engineering, requires manager approval for all other departments |
| **Hardware Replacement** | Verifies identity → checks warranty → discloses $250 cost if out of warranty → proceeds only after YES confirmation |

All three scenarios enforce **identity verification first** — deterministically, in code, every single time.

---

## What Agent Script Adds

According to the official Salesforce documentation, Agent Script lets you define:

- **Specific areas where the LLM is free to reason** — intent classification, natural language understanding
- **Specific areas that execute deterministically** — flows called in guaranteed sequence via `->` (arrow)
- **Variables that reliably store state** — persists across every conversation turn, not relying on LLM memory
- **Conditional expressions controlling execution paths** — branching driven by data, not LLM judgment
- **Deterministic subagent transitions** — routing in code, not as an LLM suggestion

The two core symbols:

```
->   Arrow = deterministic. Code runs exactly as written, every time.
|    Pipe  = prompt. This text goes to the LLM to reason about.
```

---

## Repository Structure

```
AgentScript/
├── force-app/
│   └── main/
│       └── default/
│           ├── aiAuthoringBundles/
│           │   └── IT_Support_Agent/
│           │       └── IT_Support_Agent.agent        # Agent Script file
│           ├── flows/
│           │   ├── Verify_Identity.flow-meta.xml
│           │   ├── Check_MFA_Status.flow-meta.xml
│           │   ├── Reset_Password.flow-meta.xml
│           │   └── Raise_Support_Ticket.flow-meta.xml
│           └── objects/
│               ├── Contact/
│               │   └── fields/
│               │       ├── EmployeeNumber__c.field-meta.xml
│               │       ├── Has_MFA__c.field-meta.xml
│               │       ├── Under_Warranty__c.field-meta.xml
│               │       └── Password_Reset_Required__c.field-meta.xml
│               └── IT_Support_Request__c/
│                   ├── IT_Support_Request__c.object-meta.xml
│                   └── fields/
│                       ├── Contact__c.field-meta.xml
│                       ├── Issue_Type__c.field-meta.xml
│                       ├── Status__c.field-meta.xml
│                       ├── Notes__c.field-meta.xml
│                       ├── Requires_Manager_Approval__c.field-meta.xml
│                       └── Chargeble__c.field-meta.xml
├── scripts/
│   └── apex/
│       └── CreateTestContacts.apex
├── manifest/
│   └── package.xml
└── README.md
```

---

## Prerequisites

Before deploying this project you need:

- Salesforce Developer Edition org with **Einstein and Agentforce enabled**
- **Salesforce CLI v2** installed — verify with `sf -v`
- **VS Code** with the Salesforce Extension Pack installed
- **Agentforce DX VS Code Extension** installed
- `Prompt Template Manager` permission set assigned to your user
- An authorized SFDX project connected to your org

---

## Custom Object: IT_Support_Request__c

Created to log all IT support requests raised by the agent.

| Field Label | API Name | Type | Notes |
|---|---|---|---|
| Request Number | Name | Auto Number | Format: REQ-{0000} |
| Contact | Contact__c | Lookup (Contact) | Links ticket to employee |
| Issue Type | Issue_Type__c | Picklist | password_reset, software_access, hardware |
| Status | Status__c | Picklist | Open, Pending Approval, Escalated, Resolved |
| Notes | Notes__c | Long Text Area | Optional notes |
| Requires Manager Approval | Requires_Manager_Approval__c | Checkbox | Default: false |
| Chargeable | Chargeble__c | Checkbox | Default: false |

> Note: `Chargeble__c` has a typo in the field label — this matches what was deployed to the org. Do not rename without updating the Raise_Support_Ticket flow accordingly.

---

## Custom Fields on Contact

| Field Label | API Name | Type | Default | Purpose |
|---|---|---|---|---|
| Employee Number | EmployeeNumber__c | Text (20) | — | Unique employee identifier used for identity verification |
| Has MFA | Has_MFA__c | Checkbox | false | Whether employee has MFA enabled — gates password reset |
| Under Warranty | Under_Warranty__c | Checkbox | true | Whether employee device is under warranty |
| Password Reset Required | Password_Reset_Required__c | Checkbox | false | Updated to false when Reset_Password flow runs |

---

## Flows

### 1. Verify_Identity
**Purpose:** Looks up a Contact record by EmployeeNumber__c and returns identity confirmation, Contact Id, and department.

| Variable | Direction | Type | Description |
|---|---|---|---|
| employee_id_input | Input | Text | Employee ID provided by the user |
| verified | Output | Boolean | True if a matching Contact was found |
| contact_id | Output | Text | Salesforce Contact Id |
| department | Output | Text | Contact.Department field value |

**Logic:** Get Records on Contact where `EmployeeNumber__c = employee_id_input`. On success — set verified = true, map Id and Department. On fault — set verified = false.

---

### 2. Check_MFA_Status
**Purpose:** Checks the Has_MFA__c field on the verified Contact record.

| Variable | Direction | Type | Description |
|---|---|---|---|
| contact_id | Input | Text | Salesforce Contact Id |
| has_mfa | Output | Boolean | Value of Has_MFA__c on the Contact |

**Logic:** Get Records on Contact where `Id = contact_id`. Map Has_MFA__c to output. On fault — set has_mfa = false.

---

### 3. Reset_Password
**Purpose:** Simulates a password reset by updating Password_Reset_Required__c to false on the Contact record.

| Variable | Direction | Type | Description |
|---|---|---|---|
| contact_id | Input | Text | Salesforce Contact Id |
| reset_successful | Output | Boolean | True if record update succeeded |

**Logic:** Update Contact record — set Password_Reset_Required__c = false. On success — set reset_successful = true. On fault — set reset_successful = false.

---

### 4. Raise_Support_Ticket
**Purpose:** Creates an IT_Support_Request__c record with routing logic based on issue type and department.

| Variable | Direction | Type | Description |
|---|---|---|---|
| contact_id | Input | Text | Salesforce Contact Id |
| issue_type | Input | Text | One of: password_reset, software_access, hardware |
| department | Input | Text | Employee department for routing |
| ticket_number | Output | Text | Created record Id |
| requires_approval | Output | Boolean | True if manager approval required |
| chargeable | Output | Boolean | True if hardware replacement incurs cost |

**Routing logic:**

| Issue Type | Condition | Status | Requires Approval | Chargeable |
|---|---|---|---|---|
| password_reset | Any | Escalated | false | false |
| software_access | Department = Engineering | Open | false | false |
| software_access | Department ≠ Engineering | Pending Approval | true | false |
| hardware | Under_Warranty__c = true | Open | false | false |
| hardware | Under_Warranty__c = false | Open | false | true |

---

## Agent Script — IT_Support_Agent.agent

### Block Structure (in order)

| Block | Purpose |
|---|---|
| system | Global agent personality, welcome and error messages |
| config | Agent metadata — API name, type, role, company context |
| variables | Six global variables persisting across all conversation turns |
| language | Supported locales |
| start_agent | Entry point — routes to correct subagent based on verified state |
| subagent | Core IT support logic — actions, reasoning instructions, tools |

### Variables

| Name | Type | Default | Description |
|---|---|---|---|
| contact_id | mutable string | "" | Salesforce Contact Id of the verified employee |
| verified | mutable boolean | False | Whether identity has been confirmed this session |
| has_mfa | mutable boolean | False | Whether the employee has MFA enabled |
| department | mutable string | "" | Employee department for routing decisions |
| requested_action | mutable string | "" | What the employee explicitly asked for |
| action_confirmed | mutable boolean | False | Whether employee confirmed an irreversible action |

### Reasoning Flow

```
Every message → start_agent
   → if not verified → prompt for employee ID
   → if verified     → transition to IT_Support_Handler

IT_Support_Handler
   → DETERMINISTIC: run Verify_Identity flow
   → LLM: classify intent → set requested_action
   → DETERMINISTIC: branch on requested_action value
      → "question"        → LLM answers, no action taken
      → "password_reset"  → confirmation gate → check MFA → reset
      → "software_access" → confirmation gate → route by department
      → "hardware"        → warranty check → cost disclosure → confirm
```

---

## How to Deploy

### Step 1 — Clone the repository

```bash
git clone https://github.com/batra-kapil/AgentScript.git
cd AgentScript
```

### Step 2 — Authorize your org

```bash
sf org login web --alias your-org-alias
```

### Step 3 — Deploy custom fields first

```bash
sf project deploy start \
  --source-dir force-app/main/default/objects \
  --target-org your-org-alias
```

### Step 4 — Deploy flows

```bash
sf project deploy start \
  --source-dir force-app/main/default/flows \
  --target-org your-org-alias
```

> All four flows must deploy and activate successfully before the agent can reference them.

### Step 5 — Deploy the Agent Script

```bash
sf project deploy start \
  --source-dir force-app/main/default/aiAuthoringBundles \
  --target-org your-org-alias
```

### Step 6 — Create test Contact records

Run in Developer Console → Execute Anonymous:

```apex
List<Contact> contacts = new List<Contact>{
    new Contact(
        FirstName = 'Arjun',
        LastName = 'Kapoor',
        Department = 'Engineering',
        EmployeeNumber__c = 'EMP001',
        Has_MFA__c = true,
        Under_Warranty__c = true,
        Password_Reset_Required__c = true
    ),
    new Contact(
        FirstName = 'Sneha',
        LastName = 'Iyer',
        Department = 'Finance',
        EmployeeNumber__c = 'EMP002',
        Has_MFA__c = false,
        Under_Warranty__c = true,
        Password_Reset_Required__c = false
    ),
    new Contact(
        FirstName = 'Vikram',
        LastName = 'Nair',
        Department = 'HR',
        EmployeeNumber__c = 'EMP003',
        Has_MFA__c = true,
        Under_Warranty__c = false,
        Password_Reset_Required__c = false
    )
};
insert contacts;
System.debug('Test contacts created successfully');
```

### Step 7 — Activate the agent

Go to **Setup → Agentforce Agents → IT Support Agent → Activate**

### Step 8 — Test in Agentforce Builder

Open Agentforce Builder → click **Preview** and run these conversations:

**Test 1 — Password reset happy path (EMP001 — has MFA)**
```
My employee ID is EMP001. I need a password reset.
→ YES
```
Expected: Identity verified → MFA confirmed → password reset after YES.

**Test 2 — Software access, Finance department (EMP002)**
```
My employee ID is EMP002. I need access to the Finance reporting tool.
→ YES
```
Expected: Manager approval required — Sneha is in Finance.

**Test 3 — Hardware, out of warranty (EMP003)**
```
My employee ID is EMP003. My laptop is broken.
→ YES
```
Expected: $250 cost disclosed before confirmation — Vikram's device is out of warranty.

**Test 4 — Question only, no action taken**
```
My employee ID is EMP001. How long does a password reset usually take?
```
Expected: Agent answers the question. No reset fires.

---

## Debugging

### Interaction Summary — Agentforce Builder

Open Agentforce Builder → run a test conversation → check the **Interaction Summary** panel. Shows each step in sequence: Input, Reasoning, Topic Transitions, Actions, Output Evaluation.

### Trace Tab — Agentforce Builder

Expand the **Trace tab** at the bottom of Agentforce Builder for millisecond-level timing and execution detail.

### Agent Tracer — VS Code

Open the **Agent Tracer tab** in the Agentforce DX VS Code extension. Shows full execution plan: User Input, Topic Selected, Variable Updates, Tools Enabled, Reasoning steps, Topic Transitions. Click any conversation turn to view the complete raw JSON execution trace.

### Event Logs — Production Monitoring

Go to **Setup → Agentforce Agents → IT Support Agent → Edit → Enable "Enrich event logs with conversation data"**

Then query via SOQL:
```sql
SELECT ConversationDefinitionId, EventDateTime, EventLabel, 
       EventTarget, EventTargetType, StepType
FROM ConversationDefinitionEventLog
WHERE CreatedDate = TODAY
```

---

## Known Issues

- `Chargeble__c` has a typo in the field label — this is intentional to match the deployed org field name. Do not rename without updating the `Raise_Support_Ticket` flow.
- `EmployeeNumber__c` is a custom field — the standard Contact object `EmployeeNumber` field was not available in this org configuration.
- Deploy order matters — objects before flows before agent. Deploying out of order will cause atomic rollback failures.

---

## Resources

- [Official Agent Script Documentation](https://developer.salesforce.com/docs/ai/agentforce/guide/agent-script.html)
- [Agent Script Reference: Variables](https://developer.salesforce.com/docs/ai/agentforce/guide/ascript-ref-variables.html)
- [Agent Script Reference: Reasoning Instructions](https://developer.salesforce.com/docs/ai/agentforce/guide/ascript-ref-instructions.html)
- [Agent Script Reference: Tools](https://developer.salesforce.com/docs/ai/agentforce/guide/ascript-ref-tools.html)
- [Agent Script Reference: Utils](https://developer.salesforce.com/docs/ai/agentforce/guide/ascript-ref-utils.html)
- [How to Debug Agent Script](https://developer.salesforce.com/blogs/2026/03/agent-script-decoded-how-to-debug-agent-script)
- [Agentforce DX Extension for VS Code](https://marketplace.visualstudio.com/items?itemName=salesforce.salesforcedx-vscode-agentforce)

---

## About SF Bolt

SF Bolt is a YouTube channel and LinkedIn brand focused on Salesforce and AI content — built by Kapil Batra, Salesforce MVP and Certified Application Architect.

- YouTube: [SF Bolt](https://www.youtube.com/@sfbolt)
- LinkedIn: [Kapil Batra](https://www.linkedin.com/in/kapilbatra)
- GitHub: [batra-kapil](https://github.com/batra-kapil)

---

*If this helped you, star the repo and subscribe to SF Bolt for more Agentforce content every week.*