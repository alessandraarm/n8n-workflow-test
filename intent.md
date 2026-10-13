# Blocked Supplier Invoice Release Workflow - teste

n8n workflow triggered by Joule Agent to automatically process blocked supplier invoices end-to-end.

## Business Challenge

AP teams spend significant manual effort investigating why supplier invoices are blocked, cross-referencing purchase orders, goods receipts, and supplier contracts before deciding whether to release or reject. This workflow automates that investigation and routes the findings to the AP Clerk for a final decision via SAP Task Center, reducing resolution time and manual lookup effort.

## Business Architecture (RBA)

### End-to-End Process

Source to Pay (generic)

### Process Hierarchy

```
Source to Pay (generic)
└── Invoice to Pay (generic)
    └── Process supplier invoices (generic) [BPS-325]
        └── Receive and process supplier invoices
└── Procure to Receipt (generic)
    └── Purchase products and services (generic) [BPS-329]
        └── Manage purchase order
        └── Manage purchase scheduling agreement releases
```

### Summary

The business challenge maps to the Source to Pay E2E process. BPS-325 (Process supplier invoices) covers reading, analysing, and releasing blocked invoices with AP Clerk approval. BPS-329 (Purchase products and services) covers cross-referencing PO, goods receipt, and supplier contract block reasons.

## Fit Gap Analysis

| Requirement (business) | Standard asset(s) found | API ORD ID | MCP Server ORD ID | MCP Server Version | Webhook API ORD ID | Gap? | Notes / assumptions |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Read blocked supplier invoices from S/4HANA | SAP S/4HANA Cloud Public/Private Edition — Supplier Invoice Processing | `sap.s4:apiResource:API_SUPPLIERINVOICE_PROCESS_SRV:v1` | — | — | — | No | Access via SAP MCP Client node in n8n |
| Cross-reference PO details for block reason | SAP S/4HANA Cloud — Purchase Order Processing | `sap.s4:apiResource:CE_PURCHASEORDER_0001:v1` | — | — | — | No | Access via SAP MCP Client node in n8n |
| Cross-reference Goods Receipt for block reason | SAP S/4HANA Cloud — Goods Receipt | `sap.s4:apiResource:SSPOPENITMGOODSRECEIPT_0001:v1` | — | — | — | No | Access via SAP MCP Client node in n8n |
| AI analysis of block reasons across PO/GR/Contract | CUSTOM.sapAgent node (available) | — | — | — | — | No | SAP Agent node available on this n8n instance |
| AP Clerk approval via SAP Task Center | CUSTOM.sapTaskCenter node (available) | — | — | — | — | No | SAP Task Center node available; mandatory Switch pattern required post-HITL |
| Release invoice in S/4HANA after approval | SAP S/4HANA Cloud — Supplier Invoice Processing | `sap.s4:apiResource:API_SUPPLIERINVOICE_PROCESS_SRV:v1` | — | — | — | No | Write-back via SAP MCP Client node |
| Webhook trigger invokable by Joule Agent | n8n-nodes-base.webhook (available) | — | — | — | — | No | responseMode set to responseNode for Joule compatibility |
| Loop per blocked invoice for individual analysis | n8n-nodes-base.splitInBatches (available) | — | — | — | — | No | Iterate one invoice at a time through agent + HITL |

### Key Findings

-   SAP S/4HANA Cloud (Public and Private Edition) provides full standard coverage for supplier invoice processing (BPS-325) and purchase order processing (BPS-329).
    
-   All required n8n nodes are confirmed available on this instance: `CUSTOM.sapMcpClient`, `CUSTOM.sapAgent`, `CUSTOM.sapTaskCenter`, `webhook`, `switch`, `splitInBatches`, `respondToWebhook`.
    
-   No pre-built MCP server was found for the S/4HANA Supplier Invoice or Purchase Order APIs; all S/4HANA calls will be made via the SAP MCP Client n8n node.
    
-   The workflow follows the pattern: Webhook → SAP MCP Client (read) → Loop → SAP Agent (analyse) → SAP Task Center (HITL) → Switch (approve/reject) → SAP MCP Client (release).
    
-   Joule Agent acts as the entry point via a webhook call; the workflow responds via Respond to Webhook upon completion.
    
-   SAP Fieldglass and Ariba capabilities are excluded as the use case is focused on direct procurement invoice processing.

## Recommendations

### Automated Blocked Invoice Resolution Workflow

#### Executive Summary

n8n workflow with AI analysis, HITL approval, and S/4HANA write-back.

#### Recommended Solution

An n8n workflow triggered by a Joule Agent webhook. For each blocked supplier invoice retrieved from SAP S/4HANA Cloud via the SAP MCP Client, an SAP Agent analyses the block reason by cross-referencing Purchase Order, Goods Receipt, and Supplier Contract data. Findings are presented to the AP Clerk via SAP Task Center. Upon approval, the invoice is released in S/4HANA via the SAP MCP Client. Rejections are logged and the webhook responds with a summary.

#### Recommended solution category

n8n Workflow

#### Intent fit

95%