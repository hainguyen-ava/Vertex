# Functional Requirements
## Vertex IP Correspondence Automation Agent

Consolidated from the architecture diagram (`vertex-email-rules.html`) and the Functional Design Document (`Vertex IP FDD - Avanade Formatted.docx`).

---

### FR-1 — Email Ingestion

| ID | Requirement |
|---|---|
| FR-1.1 | The agent shall monitor the mailbox and read every new incoming email, including new messages, replies, and acknowledgements. |
| FR-1.2 | The agent shall process each email within minutes of receipt. |

### FR-2 — Reference (Ref #) Identification

| ID | Requirement |
|---|---|
| FR-2.1 | The agent shall scan the email **title (subject) first**; if no reference is found, it shall scan the **message body**. |
| FR-2.2 | If no reference is found in the subject or current body, the agent shall inspect the **most recent quoted correspondence** in the thread. |
| FR-2.3 | The agent shall recognize references following common lead-in phrases, including "Client Reference:", "Your Ref", "Your Reference:", "Your Ref No", or similar. |
| FR-2.4 | The agent shall support the current known matter prefixes (VPI, ALP, EXO, MOD, SEM, VIA, VMN), subject to confirmation with Vertex IP Operations; new prefixes require a change request. |

### FR-3 — Reference Normalization

| ID | Requirement |
|---|---|
| FR-3.1 | The agent shall standardize detected references before matching by: converting to uppercase, converting "/" to "-", converting spaces to "-", removing country codes, removing subcase suffixes, and trimming extra spaces. |

### FR-4 — Folder Rule Validation

| ID | Requirement |
|---|---|
| FR-4.1 | If the reference is a **Vertex** reference, the agent shall validate it has exactly **2 segments** (e.g., `00-113`, `00-128`). |
| FR-4.2 | If the reference is **not** a Vertex reference, the agent shall validate it has a **prefix + 2 segments** (e.g., `SEM-24-783`, `SEM-25-786`). |
| FR-4.3 | Country-specific folder suffixes shall be ignored for routing purposes (e.g., `VPI-18-114 US PRV1` routes to `VPI-18-114`). |
| FR-4.4 | If a detected reference **fails** validation (FR-4.1/FR-4.2), the agent shall not file or move the email; it shall raise an Invalid Format exception per FR-7.5. |

### FR-5 — Folder Management

| ID | Requirement |
|---|---|
| FR-5.1 | The agent shall search for an existing family/matter folder matching the normalized reference. |
| FR-5.2 | **If a matching folder exists**, the agent shall move the email into that folder. |
| FR-5.3 | **If no matching folder exists**, the agent shall create a new folder named after the normalized reference, then move the email into it. |
| FR-5.4 | Every folder-creation event shall be recorded as an audit/activity entry. |

### FR-6 — Automated Filing Rules

| ID | Requirement |
|---|---|
| FR-6.1 | **Single valid family detected** → file the email automatically. |
| FR-6.2 | **Folder located** → move the email to that folder. |
| FR-6.3 | **Valid family detected + folder missing** → create the family folder, move the email, and record the activity. |

### FR-7 — Exception Handling

| ID | Requirement |
|---|---|
| FR-7.1 | **No reference found** → leave the email in the Inbox, add it to the exception queue, document the reason, and include it in the daily report. |
| FR-7.2 | **Multiple family references detected** → do not auto-file; route for manual review. |
| FR-7.3 | **Subject/body conflict** (different references in subject vs. body) → do not determine priority automatically; route for manual review. |
| FR-7.4 | **Conflicting references in quoted history** → create an exception and require manual review. |
| FR-7.5 | **Invalid reference format (Folder Rule Validation failure)** → leave the email untouched in Inbox, do not create/file to a folder, add an exception record (Exception Type: Invalid Format) noting which validation rule failed, route for manual review, and include in the daily report. |
| FR-7.6 | **Reference present only inside an attachment** → do not process in Phase 1; create an exception (attachment OCR/content extraction is out of scope for Phase 1). |

### FR-8 — Audit Logging

| ID | Requirement |
|---|---|
| FR-8.1 | The agent shall generate an audit entry for every processed email, capturing: Message ID, Sender, Subject, Reference, Folder, Processing Result (Filed / Created Folder and Filed / Exception), Timestamp (UTC), and Exception Type (when applicable). |

### FR-9 — Daily Reporting

| ID | Requirement |
|---|---|
| FR-9.1 | The agent shall send one daily report email to business owners, on a configurable schedule (e.g., 5:00 PM Eastern), configured via the Power Automate recurrence trigger. |
| FR-9.2 | The report shall include an **Executive Summary** (emails processed, auto-filed, folders created, exceptions, automation success rate). |
| FR-9.3 | The report shall list all **new folders created** that day, with family reference and creation timestamp. |
| FR-9.4 | The report shall list **filing activity** for the day (message, sender, reference, folder, result). |
| FR-9.5 | The report shall list all **exceptions/unprocessed emails** for the day, each with sender, exception type, reason, and recommended action. |
| FR-9.6 | The report shall list **normalization activity** (original value, normalized value, source field) for the day. |

---

### Traceability

| Requirement area | Architecture component(s) | Technical implementation component(s) | FDD section(s) | TDD section(s) |
|---|---|---|---|---|
| Email ingestion | Email Inbox → Vertex Email Agent | Outlook trigger ("When a new email arrives (V3)") → Power Automate | §1, §2, §4 | §2 |
| Reference identification | Ref # Scanner | Azure OpenAI (via Foundry) — Instructions + Skill, returns structured JSON | §6 | §2, §3 |
| Normalization | Ref # Scanner → Folder Rule Validator | Azure OpenAI structured JSON output (normalized_value field) | §7 | §3 |
| Folder rule validation | Folder Rule Validator | Deterministic Condition/Switch action in Power Automate (not a model call) | §4, §8 (folder-naming cards) | §3, §4 |
| Folder management & filing | Folder Manager → Mailbox Folders | Power Automate (Outlook connector actions) | §8, §9, §11 | §2 |
| Exceptions | Ref # Scanner / Folder Manager → Activity Log | Power Automate → Dataverse (Log Exception) | §10 | §2, §5 |
| Audit logging | Activity Log | Dataverse table (Activity & Exception Log) | §12 | §5 |
| Daily reporting | Activity Log → Daily Report Generator → Business Owners | Independent Power Automate flow (daily recurrence) → Dataverse query → Outlook send | §13 | §2 |

Note: consolidated FR-1 through FR-9 tables are also reproduced as a dedicated **Section 5 — Functional Requirements** in `Vertex IP FDD - Avanade Formatted.docx` / `Vertex IP FDD.md` for client sign-off. The technical implementation architecture (Outlook, Power Automate, Azure OpenAI via Foundry, Dataverse) is documented in the standalone **Vertex IP TDD** (`Vertex IP TDD.md` / `Vertex IP TDD.docx`) and the corresponding Archify workflow diagram (`.archify\workflow-vertex-technical-implementation-20261008-092901\vertex-technical-implementation.html`).

