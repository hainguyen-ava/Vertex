# Functional Design Document (FDD)
## Vertex IP Correspondence Automation Agent

| | |
|---|---|
| **Client** | Vertex Pharmaceuticals |
| **Project** | IP Correspondence Automation |
| **Solution Platform** | Microsoft Power Platform (Power Automate, Outlook, Dataverse/SharePoint) |
| **Document Type** | Functional Design Document (FDD) |
| **Version** | 1.0 Draft |
| **Prepared By** | Hai Nguyen |
| **Date** | October 2026 |

---

## Revision History and Approvals

### Revision History

| Version | Date | Author | Description |
|---|---|---|---|
| 1.0 Draft | October 2026 | Hai Nguyen | Initial draft issued for internal and stakeholder review |

### Approvals

This document requires the following approvals before Draft status is removed and the design is baselined.

| Name | Role | Decision | Date |
|---|---|---|---|
| | Vertex IP Operations Lead | Pending | |
| | Avanade Engagement Lead | Pending | |

---

## Table of Contents

1. [Document Purpose](#1-document-purpose)
2. [Business Overview](#2-business-overview)
3. [Solution Overview](#3-solution-overview)
4. [Functional Scope](#4-functional-scope)
5. [Functional Requirements](#5-functional-requirements)
6. [Reference Identification](#6-reference-identification)
7. [Reference Normalization](#7-reference-normalization)
8. [Folder Structure](#8-folder-structure)
9. [Automated Filing Rules](#9-automated-filing-rules)
10. [Exception Rules](#10-exception-rules)
11. [Folder Creation Process](#11-folder-creation-process)
12. [Audit Logging](#12-audit-logging)
13. [Daily Report Functional Design](#13-daily-report-functional-design)
14. [Non-Functional Requirements](#14-non-functional-requirements)
15. [Future Phase Enhancements](#15-future-phase-enhancements)
- [Appendix – Proposed Business Decisions](#appendix--proposed-business-decisions)

---

## 1. Document Purpose

This Functional Design Document defines the end-to-end functional behavior of the Vertex IP Correspondence Automation solution. The solution automates the classification, organization, and filing of intellectual property correspondence received via Outlook email.

The system identifies patent family references from incoming correspondence, files emails into the appropriate folder structure, creates folders when required, manages exceptions, and generates operational reporting.

## 2. Business Overview

Today, Vertex IP Operations manually:

- Review incoming emails
- Identify patent family references
- Locate family folders
- Create folders when necessary
- Move emails to proper locations
- Track filing exceptions

This process consumes significant operational effort and relies heavily on user knowledge of folder structures and reference formats.

The proposed solution automates these activities while preserving business oversight for ambiguous scenarios.

## 3. Solution Overview

### High-Level Process

> **Note:** The technical implementation architecture (Outlook, Power Automate, Azure OpenAI via Foundry, Dataverse) is documented separately in the **Vertex IP Technical Design Document (TDD)**.

## 4. Functional Scope

### Included

**Email Processing**
- New emails
- Replies
- Acknowledgements

**Reference Extraction**
- Subject
- Current message body
- Quoted message history

**Folder Management**
- Folder discovery
- Folder creation
- Folder validation

**Filing**
- Move email to family folder
- Audit processed activity

**Reporting**
- Daily operational report
- Exception reporting

### Excluded (Phase 1)

- Attachment OCR
- Attachment content extraction
- iManage integration
- Patent docketing activities
- Patent workflow management
- Legal review activities

## 5. Functional Requirements

This section consolidates the solution's functional requirements into nine numbered areas (FR-1 through FR-9) for client review and sign-off. Each requirement traces to the detailed functional design in the sections that follow and to the solution architecture diagram.

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

## 6. Reference Identification

### Supported Family References

Examples include:

1. VPI-18-114
2. ALP-25-001
3. EXO-22-115
4. MOD-24-220
5. SEM-26-788
6. VIA-05-201
7. VMN-19-114

Based on current samples, the solution shall support the matter prefixes:

- VPI
- ALP
- EXO
- MOD
- SEM
- VIA
- VMN

> **Note:** this prefix list reflects current samples. It should be confirmed with Vertex IP Operations as complete prior to go-live; any new prefixes will require a change request.

### Reference Search Order

The system searches in the following sequence:

**Priority 1** — Email Subject

> *Example:* Notice of Allowance | Your Ref: SEM-26-788

**Priority 2** — Current Email Body

> *Example:* Client Reference: SEM-26-788

**Priority 3** — Quoted Email Chain

If no reference is detected in:
- Subject
- Current body

the system shall inspect the most recent quoted correspondence.

## 7. Reference Normalization

The system shall standardize references before matching.

### Examples

| Detected Value | Normalized Value |
|---|---|
| VPI/19-117 | VPI-19-117 |
| VPI 19 117 | VPI-19-117 |
| vpi-19-117 | VPI-19-117 |
| VPI-18-114 US PRV1 | VPI-18-114 |
| SEM-25-786 US | SEM-25-786 |

### Normalization Rules

- Convert to uppercase
- Convert "/" to "-"
- Convert spaces to "-"
- Remove country codes
- Remove subcase suffixes
- Trim extra spaces

## 8. Folder Structure

### Expected Folder Architecture

```
Inbox
├─ VPI-18-114
├─ VPI-19-117
├─ SEM-26-788
├─ ALP-21-005
└─ VIA-05-201
```

Family folders are the filing destination.

Country-specific folders are ignored for routing purposes.

> *Example:* VPI-18-114 US PRV1
> Routes to: **VPI-18-114**

## 9. Automated Filing Rules

### Rule 1 – Single Valid Family
- **Condition:** One valid family detected
- **Action:** File automatically

### Rule 2 – Folder Exists
- **Condition:** Folder located
- **Action:** Move email to folder

### Rule 3 – Folder Missing
- **Condition:** Valid family detected + Folder not found
- **Action:** Create family folder, move email, record activity

## 10. Exception Rules

### Exception 1 – No Reference Found
- **Condition:** No valid reference detected
- **Action:**
  - Leave email in Inbox
  - Add to exception queue
  - Include in daily report

### Exception 2 – Multiple Family References
- **Example:** VPI-17-127, VPI-17-128
- **Action:**
  - No auto-filing
  - Route for manual review

### Exception 3 – Subject/Body Conflict
- **Example:**
  - Subject: VPI-19-117
  - Body: VPI-20-015
- **Action:**
  - Do not determine priority automatically
  - Route for manual review

### Exception 4 – Conflicting References in Quoted History
- **Example:** Latest quoted: VPI-19-117 / Older quoted: VPI-20-015
- **Action:**
  - Create exception
  - Require manual review

### Exception 5 – Folder Rule Validation Failure (Invalid Format)
- **Condition:** A reference is detected but fails Folder Rule Validation — a Vertex reference without exactly 2 segments, or a non-Vertex reference without a prefix and 2 segments.
- **Example:** VPI-ABC-123
- **Action:**
  - Leave email untouched in Inbox (do not create or file to a folder)
  - Add exception record (Exception Type: Invalid Format), noting which validation rule failed
  - Route for manual review
  - Include in daily report

### Exception 6 – Attachment Only
- **Example:** Reference appears solely within a PDF attachment.
- **Action:**
  - Do not process in Phase 1
  - Create exception

## 11. Folder Creation Process

When a family folder does not exist:

1. **Step 1** — Create folder *(Example: SEM-26-788)*
2. **Step 2** — Record creation activity
3. **Step 3** — Move email into folder
4. **Step 4** — Capture audit details

## 12. Audit Logging

Every processed email shall generate an audit entry.

### Audit Fields

| Field | Description |
|---|---|
| Message ID | Unique identifier of the source email, assigned by the mail system |
| Sender | Email address of the message sender |
| Subject | Original email subject line, unmodified |
| Reference | Normalized matter/family reference detected (e.g., VPI-18-114); blank if none found |
| Folder | Destination family folder the email was filed to, or the existing folder it was matched against |
| Processing Result | Filed / Created Folder and Filed / Exception |
| Timestamp | Date and time the email was processed (UTC) |
| Exception Type | Populated only when Processing Result = Exception (e.g., No Reference Found, Multiple References, Invalid Format) |

*Each field is captured per processed email; descriptions to be finalized during build.*

## 13. Daily Report Functional Design

### Report Schedule

Configurable — *Example: 5:00 PM Eastern*

The send time is configured via the Power Automate recurrence trigger for the daily report flow, and can be adjusted by an administrator without code changes.

### Executive Summary

| Metric | Value |
|---|---|
| Emails Processed | |
| Auto-Filed | |
| Folders Created | |
| Exceptions | |
| Automation Success Rate | |

*Structural template only — populated at runtime from the day's processed email activity; shown empty here by design.*

### New Folders Created

| Folder Name | Family Reference | Created Timestamp |
|---|---|---|
| | | |

*Structural template only — populated at runtime from the day's processed email activity; shown empty here by design.*

### Filing Activity

| Message ID | Sender | Reference | Folder | Result |
|---|---|---|---|---|
| | | | | |

*Structural template only — populated at runtime from the day's processed email activity; shown empty here by design.*

### Exceptions

| Message ID | Sender | Exception Type | Reason | Recommended Action |
|---|---|---|---|---|
| | | | | |

*Structural template only — populated at runtime from the day's processed email activity; shown empty here by design.*

### Normalization Activity

| Original Value | Normalized Value | Source Field |
|---|---|---|
| | | |

*Structural template only — populated at runtime from the day's processed email activity; shown empty here by design.*

## 14. Non-Functional Requirements

**Performance**
- Process emails within minutes of receipt

**Accuracy**
- Target >95% filing accuracy

**Security**
- Operate within Vertex-approved Microsoft 365 environment

**Auditability**
- Every action logged

**Scalability**
- Support additional IP teams and family types

## 15. Future Phase Enhancements

> **Note:** AI-assisted reference extraction via Azure OpenAI (Foundry) is now part of the **Phase 1** technical implementation (see the Vertex IP Technical Design Document), not a future enhancement.

### Attachment Processing
- OCR
- PDF parsing
- Reference extraction from attachments

### Copilot Studio Agent

Business users can ask:
- Show all correspondence for SEM-26-788
- Why wasn't this email filed?
- What folders were created today?
- List all open exceptions.

### Power BI Dashboard

Operational KPIs including:
- Processing volume
- Automation success rate
- Open exceptions
- Top active patent families
- Trend analysis

## Appendix – Proposed Business Decisions

| Scenario | Proposed Action |
|---|---|
| One valid family | Auto-file |
| Folder exists | Move email |
| Folder missing | Create folder and move |
| No reference | Leave in Inbox |
| Multiple families | Manual review |
| Subject/body conflict | Manual review |
| Invalid format | Manual review |
| Reference only in attachment | Manual review |
| Slash format | Normalize and process |
| Country suffixes | Remove for routing |
| Duplicate processing | Prevent re-processing |
| Folder rename | Do not rename automatically |

---

This design provides a complete Phase 1 MVP blueprint that can be implemented using Outlook, Power Automate, and Dataverse/SharePoint while establishing a clear roadmap toward Copilot Studio and AI-enabled IP operations.
