# Specification: blocked-invoice-release-workflow

> **Guidelines**: Read [../../guidelines.md](../../guidelines.md) and [../../guidelines-n8n-workflow.md](../../guidelines-n8n-workflow.md) before executing ANY tasks below.

Asset folder: `assets/workflows/blocked-invoice-release-workflow/`
Asset type: `n8nworkflow`
Workflow file: `blocked-invoice-release.n8n.json`

---

## Pre-generation research (MANDATORY — do before writing any node)

- [ ] Read skill references: `n8n-sap-mcp-client.md`, `n8n-sap-agent.md`, `n8n-sap-task-center.md`, `n8n-trigger-nodes-webhook.md`, `n8n-expressions.md`, `n8n-loops.md`, `n8n-loops-loop-over-items.md`
- [ ] Call `search-nodes-catalog` with all planned node keywords: `webhook`, `split in batches`, `sap mcp client`, `sap agent`, `sap task center`, `switch`, `aggregate`, `respond to webhook`, `sticky note`
- [ ] Record the exact `name` (→ `type`) and `version` (→ `typeVersion`) for every node from catalog results
- [ ] Confirm all planned nodes were returned by the catalog; if any are missing, find a supported alternative

## Solution Setup

- [ ] Invoke the `setup-solution` skill for a single n8n workflow asset named `blocked-invoice-release-workflow` under `assets/workflows/blocked-invoice-release-workflow/`. Pass node types `CUSTOM.sapMcpClient` and `CUSTOM.sapAgent` so the skill declares the correct SAP dependencies. Follow **Example 1: Multi-Asset Solution (Agent + n8n Workflows)** in `references/setup-solution.md`.
- [ ] Verify `solution.yaml` has a `ref:` entry pointing to `./assets/workflows/blocked-invoice-release-workflow/asset.yaml`
- [ ] Verify `assets/workflows/blocked-invoice-release-workflow/asset.yaml` exists with `metadata.type: n8nworkflow` and `sourceRoot: "."`

## Workflow Design

### Trigger

- [ ] Add a Webhook trigger node named **"Blocked Invoice Received"**
  - `responseMode: "responseNode"` (because a Respond to Webhook node is used at the end)
  - `httpMethod: POST`
  - Path will be generated/overwritten by `generate-workflow-asset.js` — do not hand-set it

### Step 1 — Fetch Blocked Supplier Invoices

- [ ] Add a `CUSTOM.sapMcpClient` node named **"Get Blocked Invoices (S/4HANA)"**
  - Tool selection: use the supplier invoice API tool to list all supplier invoices that have a payment block indicator
  - This is a read-only call; no write-back
  - Connect from: Webhook trigger node
  - Follow all rules in `n8n-sap-mcp-client.md`

### Step 2 — Loop Over Each Blocked Invoice

- [ ] Add a `n8n-nodes-base.splitInBatches` node (Loop Over Items) named **"Loop Over Blocked Invoices"**
  - `batchSize: 1` — process one invoice at a time through the agent and HITL steps
  - Connect from: "Get Blocked Invoices (S/4HANA)"
  - Follow rules in `n8n-loops-loop-over-items.md`

### Step 3 — Analyse Block Reason (SAP Agent)

- [ ] Add a `CUSTOM.sapAgent` node named **"Analyse Invoice Block Reason"**
  - The agent must: (a) identify the invoice block type, (b) fetch the referenced Purchase Order data via SAP MCP Client tools, (c) fetch Goods Receipt status via SAP MCP Client tools, (d) fetch Supplier Contract data if referenced via SAP MCP Client tools, (e) synthesise a structured finding: block reason, root cause, and recommended action
  - System prompt must instruct the agent to return a structured JSON object with fields: `invoiceId`, `blockReason`, `rootCause`, `recommendation`
  - Connect from: Loop Over Blocked Invoices "loop" output (the per-item output, not the "done" output)
  - Follow all rules in `n8n-sap-agent.md`; populate `agents` parameter correctly (may be empty list `[]` if no other agents exist in solution)

### Step 4 — Present Findings to AP Clerk (SAP Task Center HITL)

- [ ] Add a `CUSTOM.sapTaskCenter` node named **"AP Clerk Approval"**
  - Task title: `"Invoice Block Review: {{ $json.invoiceId }}"` (derive invoiceId from agent output)
  - Task description: surface the agent's `blockReason`, `rootCause`, and `recommendation` fields in readable form
  - `recipients`: configure at least one recipient (AP Clerk role/group); use a placeholder recipient value if the exact role is unknown — mark with a sticky note
  - Decision options: `Approve` (release invoice) and `Reject` (skip invoice)
  - Connect from: "Analyse Invoice Block Reason"
  - Follow all rules in `n8n-sap-task-center.md`

### Step 5 — Route on AP Clerk Decision (Switch)

- [ ] Add a `n8n-nodes-base.switch` node named **"Route on AP Clerk Decision"**
  - This is a **mandatory** post-Task Center Switch node as required by `n8n-sap-task-center.md`
  - Rules mode; two outputs: output 0 = Approved, output 1 = Rejected/other
  - Match on the Task Center decision field (exact field name per `n8n-sap-task-center.md`)
  - Add a fallback output for unmatched decisions (required by Rule 9)
  - Connect from: "AP Clerk Approval"

### Step 6 — Release Invoice if Approved

- [ ] Add a `CUSTOM.sapMcpClient` node named **"Release Invoice (S/4HANA)"**
  - Tool selection: use the supplier invoice API tool to remove the payment block / release the invoice by its ID
  - Connect from: Switch output 0 (Approved branch)
  - Follow all rules in `n8n-sap-mcp-client.md`

### Step 7 — Feed Results Back into Loop

- [ ] Both the Approved branch (after "Release Invoice (S/4HANA)") and the Rejected branch (Switch output 1) connect directly back to the **"Loop Over Blocked Invoices"** node's loop-feedback input (exclusive branches — no Merge required per Rule 8 Case 1)

### Step 8 — Aggregate Results and Respond

- [ ] After the Loop's **"done"** output, add a `n8n-nodes-base.aggregate` node named **"Aggregate Results"** to collect all processed invoice outcomes into a summary array
- [ ] Add a `n8n-nodes-base.respondToWebhook` node named **"Respond to Joule"**
  - Response body: structured JSON summary — total invoices processed, count approved, count rejected, list of invoice IDs with their decisions
  - `respondWith: "json"`
  - Connect from: "Aggregate Results"

## Sticky Notes

- [ ] Add a sticky note above any node that contains user-configurable values (e.g. Task Center recipient, agent system prompt placeholder values)
  - Use the standard content: "This configuration is hardcoded — update the values in this node directly when they change."

## Asset YAML & ORD ID Generation

- [ ] Run the `generate-workflow-asset.js` script to generate `workflow:` and `provides.apis[]` entries for `asset.yaml`:
  ```bash
  node .agent/skills/n8n-workflow/scripts/generate-workflow-asset.js \
    assets/workflows/blocked-invoice-release-workflow/blocked-invoice-release.n8n.json \
    blocked-invoice-release-workflow \
    <solution-name>
  ```
  Replace `<solution-name>` with the actual value from `solution.yaml` `metadata.name`.
- [ ] Paste the script output verbatim into `assets/workflows/blocked-invoice-release-workflow/asset.yaml`
- [ ] Write a one-sentence description for each `kind: rest` entry: "Webhook endpoint that receives a trigger from the Joule Agent to start processing blocked supplier invoices."

## OpenAPI Sidecar Refinement

- [ ] Refine `assets/workflows/blocked-invoice-release-workflow/workflow-schema.openapi.json` (generated by the script):
  - Replace all `<...>` placeholders with real values
  - Set `requestBody` to an empty object (the Joule Agent trigger sends no required body fields)
  - Set `200` response schema to: `{ type: object, properties: { processed: { type: integer }, approved: { type: integer }, rejected: { type: integer }, invoices: { type: array, items: { type: object, properties: { invoiceId: { type: string }, decision: { type: string, enum: ["Approved", "Rejected"] } } } } } }`
  - Verify no `<...>` placeholder or all-`string` fields remain

## pinData

- [ ] Call `pin-data-schemas` MCP tool with all node types used in the workflow (deduplicated, using exact `typeVersion` from catalog)
- [ ] For every entry in `schemas`, synthesise one sample item and add it under `pinData[<node.name>]` as a single-element array
- [ ] Merge the resulting object into the workflow's top-level `pinData` field

## Validation

- [ ] Call `validate-n8n-workflow` MCP tool with the full workflow JSON (as string) and `asset.yaml` content
- [ ] Fix any errors reported (except expected credential MISSING errors)
- [ ] Confirm all node connections are by `name`, not `id`
- [ ] Confirm no `"credentials"` block exists in any node
- [ ] Confirm `responseMode: "responseNode"` on the webhook trigger since a `respondToWebhook` node is present
- [ ] Confirm `splitInBatches` (Loop Over Items) `batchSize` is 1
- [ ] Confirm both loop branches (Approved and Rejected) connect back to the loop feedback input — no Merge node between them and the loop

## Write

- [ ] Write the completed, validated workflow JSON to `assets/workflows/blocked-invoice-release-workflow/blocked-invoice-release.n8n.json` in a single Write call (Rule 15)
