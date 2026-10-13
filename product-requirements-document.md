# Product Requirements Document (PRD)

**Title:** Blocked Supplier Invoice Release Workflow
**Date:** 2026-10-13
**Solution Category:** n8n Workflow

## Product Purpose & Value Proposition

**Elevator Pitch:**
AP teams waste significant time manually investigating why supplier invoices are blocked, cross-referencing purchase orders, goods receipts, and supplier contracts. This workflow automates that investigation using an AI agent, surfaces the findings to the AP Clerk for a single-click decision in SAP Task Center, and releases approved invoices directly in S/4HANA — all triggered from Joule chat.

**Business Need:**
Blocked invoices slow down payment cycles and require AP Clerks to manually look up data across multiple systems. This solution eliminates the manual investigation step and reduces the time from "invoice blocked" to "invoice released".

**Expected Value:**
Faster invoice resolution, reduced AP workload for exception handling, and a consistent, auditable decision process for every blocked invoice.

**Product Objectives:**
1. Automate investigation of blocked supplier invoices across PO, GR, and Supplier Contract data.
2. Route AI-generated findings to the AP Clerk for a single-approval decision via SAP Task Center.
3. Release approved invoices in SAP S/4HANA Cloud without manual data entry.

## Requirements

### Must-Have Requirements

**R1: Webhook Trigger by Joule Agent**
- **User Story:** As a Joule Chat user, I need to trigger the blocked invoice resolution workflow from SAP Joule chat so that I can initiate the process without switching applications.
- **Acceptance Criteria:** Given a Joule Agent sends an HTTP POST to the workflow webhook, the workflow starts, processes all blocked invoices, and responds with a summary.
- **Priority Rank:** 1

**R2: Read All Blocked Supplier Invoices from S/4HANA**
- **User Story:** As the workflow, I need to retrieve all currently blocked supplier invoices from SAP S/4HANA Cloud so that every blocked invoice is included in the resolution run.
- **Acceptance Criteria:** Given the workflow starts, the SAP MCP Client fetches all invoices with a block indicator from S/4HANA; the list is available for iteration.
- **Priority Rank:** 2

**R3: Per-Invoice Block Reason Analysis by SAP Agent**
- **User Story:** As an AP Clerk, I need the system to automatically identify the root cause of each invoice block across Purchase Order, Goods Receipt, and Supplier Contract data so that I receive a clear explanation without manual lookup.
- **Acceptance Criteria:** Given a blocked invoice, the SAP Agent queries PO, GR, and Supplier Contract data and returns a structured analysis of the block reason for that invoice.
- **Priority Rank:** 3

**R4: AP Clerk Approval via SAP Task Center**
- **User Story:** As an AP Clerk, I need to review the agent's findings and approve or reject release of each invoice via SAP Task Center so that I retain control over the final decision.
- **Acceptance Criteria:** Given the agent's findings for an invoice, a task is created in SAP Task Center assigned to the AP Clerk; the workflow waits for Approve or Reject; the decision is captured and drives the next step.
- **Priority Rank:** 4

**R5: Release Approved Invoice in S/4HANA**
- **User Story:** As an AP Clerk, I need approved invoices to be automatically released in SAP S/4HANA Cloud so that no additional manual steps are required after my approval.
- **Acceptance Criteria:** Given the AP Clerk approves an invoice in Task Center, the SAP MCP Client performs the invoice release action in S/4HANA; rejected invoices are skipped.
- **Priority Rank:** 5

**R6: Webhook Response Summary**
- **User Story:** As a Joule Chat user, I need to receive a response confirming how many invoices were processed, approved, and rejected so that I know the outcome of the workflow run.
- **Acceptance Criteria:** Given all invoices have been processed, the workflow responds to the original webhook with a structured summary of results.
- **Priority Rank:** 6

## Solution Architecture

**Architecture Overview:**
An n8n workflow hosted on SAP-managed n8n, triggered via webhook by a Joule Agent. The workflow iterates over each blocked invoice, invokes an SAP Agent for AI-based analysis, routes the findings through SAP Task Center for AP Clerk approval, and calls back to S/4HANA to release approved invoices.

**Key Components:**
- **Webhook Trigger**: Receives the invocation from the Joule Agent; responds with a summary upon completion.
- **SAP MCP Client (Read)**: Fetches all blocked supplier invoices from SAP S/4HANA Cloud.
- **Loop (Split in Batches)**: Iterates over each blocked invoice individually.
- **SAP Agent**: Analyses the block reason for a single invoice by querying PO, GR, and Supplier Contract data via SAP MCP Client tools.
- **SAP Task Center (HITL)**: Presents the agent's findings to the AP Clerk and waits for Approve or Reject.
- **Switch Node**: Routes based on the AP Clerk's decision.
- **SAP MCP Client (Release)**: Releases the approved invoice in S/4HANA.
- **Respond to Webhook**: Returns the outcome summary to the Joule Agent.

**Integration Points:**
- SAP S/4HANA Cloud: read blocked invoices, read PO/GR/Contract data, release invoice — via SAP MCP Client node.
- SAP Task Center: create and resolve approval tasks — via SAP Task Center node.
- Joule Agent: triggers via HTTP POST to the workflow webhook endpoint.

### Automation & Agent Behaviour

**Automation Level:** Hybrid — AI-assisted analysis with human approval gate.

**Actions the system performs without human approval:**
- Fetching blocked supplier invoices from S/4HANA.
- Running the SAP Agent to analyse block reasons for each invoice.
- Creating approval tasks in SAP Task Center.
- Releasing invoice in S/4HANA after AP Clerk approves.

**Actions that require human review or approval:**
- AP Clerk must explicitly Approve or Reject each invoice release decision in SAP Task Center.

**Model or engine used:** SAP Agent node (SAP AI Core / SAP Generative AI Hub).

**Knowledge & data sources accessed:**
- SAP S/4HANA Cloud: Supplier Invoice data, Purchase Order data, Goods Receipt data, Supplier Contract data — via SAP MCP Client.

**Tools or connectors invoked:**
- SAP MCP Client (read blocked invoices): read-only call to S/4HANA Supplier Invoice API.
- SAP MCP Client (read PO/GR/Contract for agent): read-only, used internally by the SAP Agent during analysis.
- SAP Task Center: creates a task per invoice; waits for AP Clerk decision.
- SAP MCP Client (release invoice): write action, triggered only on AP Clerk Approve.

**Guardrails & fail-safes:**
- Invoice release only occurs after an explicit AP Clerk Approve in SAP Task Center — never autonomously.
- Rejected invoices are skipped without any write-back to S/4HANA.
- If the SAP Agent fails to analyse an invoice, the task presented to the AP Clerk includes a note that analysis was unavailable; the clerk can still decide.
