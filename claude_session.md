# AI Coding Session Log — O2C Graph Intelligence Platform

**Tool**: Claude (claude.ai)  
**Date**: 2025  
**Task**: Forward Deployed Engineer Assignment — Graph-Based Data Modeling and Query System

---

## Session Overview

Built a complete SAP Order-to-Cash graph intelligence platform from scratch using AI-assisted development. The session covered data ingestion, graph modeling, visualization, LLM integration, and deployment packaging.

---

## Key Prompts & Workflows

### Phase 1: Dataset Exploration

**Prompt**: "Explore the dataset structure — list all tables, record counts, and field names for each entity"

**Outcome**: Identified 13 JSONL entity folders:
- Core O2C: sales_order_headers (100), sales_order_items (167), outbound_delivery_headers (86), billing_document_cancellations (80), journal_entry_items_accounts_receivable (123), payments_accounts_receivable (120)
- Master data: business_partners (8), customer_company_assignments (8), customer_sales_area_assignments (28), plants (44), product_descriptions (69), product_plants (200), product_storage_locations (16,723)

**Key insight**: `product_storage_locations` at 16,723 records would bloat the file — summarized to per-product aggregates instead.

---

### Phase 2: Graph Model Design

**Prompt**: "Design a graph schema for this O2C dataset — define node types, edge types, and the key join fields that connect entities across tables"

**Decisions made**:
- 9 node types: Customer, SalesOrder, SalesOrderItem, Product, Delivery, BillingDoc, JournalEntry, Payment, Plant
- Critical join path: `soldToParty` (SO → Customer), `salesOrder` (Items → SO), `material` (Items → Product), `referenceDocument` (JE → BillingDoc), `shippingPoint = productionPlant` (Delivery → Plant)
- O2C flow: Customer → SalesOrder → [Items → Products] → Delivery → BillingDoc → JournalEntry → Payment

**Key challenge**: Delivery headers don't directly reference sales orders in the dataset — they link via `shippingPoint` matching `productionPlant` in items. This required understanding the SAP data model.

---

### Phase 3: Architecture Decision

**Prompt**: "Should I use a Python Flask backend with SQLite, or go frontend-only with embedded data? The dataset is ~450KB and the target is a live demo link."

**Decision**: Frontend-only single HTML file.

**Reasoning**:
- Dataset is small enough to embed directly (~450KB JSON)
- No deployment complexity — any static host works (GitHub Pages, Netlify, etc.)
- Single file is easier to share as a demo
- LLM API calls go directly from browser to Groq — no proxy needed

**Trade-off accepted**: No persistent storage, no server-side processing. Acceptable for this dataset size.

---

### Phase 4: LLM Prompting Strategy

**Prompt**: "Design the system prompt for the LLM that will handle natural language queries. It needs to: understand all 13 tables, generate executable queries, reject off-topic questions, and handle complex O2C flow analysis"

**Final strategy**:
1. Full schema with all field names and semantic descriptions in the system prompt
2. LLM generates JavaScript (not SQL) — executes directly in browser via `new Function()`
3. Pre-built `IDX` lookup object for fast cross-table joins
4. Status code legend (A/B/C for delivery/billing statuses)
5. Explicit guardrail: off-topic → return `{"offTopic": true, "message": "..."}` JSON
6. Temperature 0.1 for deterministic code generation
7. Model: `llama-3.3-70b-versatile` on Groq (free tier)

**Why JavaScript instead of SQL?**
- Browser-native, no SQL engine needed
- Can use pre-built indices (`IDX.soByCustomer[id]`) for O(1) lookups
- Complex joins expressible with `filter`/`map`/`reduce`
- Generated code visible to user for transparency

---

### Phase 5: Graph Visualization

**Prompt**: "Build a force-directed graph renderer on HTML5 Canvas. Requirements: pan/zoom, hover tooltips with node metadata, node type filtering, highlight nodes matching query results, search by node label"

**Implementation choices**:
- Custom Canvas renderer (no library dependency)
- Force simulation: repulsion between all visible nodes, attraction along edges, center gravity
- 300 simulation steps then freezes (stable layout)
- Color-coded by entity type with consistent palette
- `new Function` sandbox for safe code execution

**Iteration**: Initial version had node crowding — fixed by increasing repulsion constant from 500 to 1200 and adding center gravity force.

---

### Phase 6: Groq API Integration

**Prompt**: "Switch from Anthropic API to Groq API. Use llama-3.3-70b-versatile, adjust headers accordingly"

**Key diff**:
```javascript
// Before (Anthropic)
fetch('https://api.anthropic.com/v1/messages', {
  headers: { 'x-api-key': key, 'anthropic-version': '2023-06-01' },
  body: JSON.stringify({ model: 'claude-sonnet-4-20250514', messages: [...] })
})

// After (Groq)
fetch('https://api.groq.com/openai/v1/chat/completions', {
  headers: { 'Authorization': `Bearer ${key}` },
  body: JSON.stringify({ model: 'llama-3.3-70b-versatile', messages: [{role:'system',...}, ...] })
})
```

Groq uses OpenAI-compatible API format — straightforward migration.

---

### Phase 7: Complete Dataset Coverage

**Prompt**: "Ensure ALL 13 tables are accessible to the LLM and included in the schema prompt. The previous version excluded product_storage_locations — replace it with a summarized version"

**Solution**: Built `product_storage_summary` — aggregate per product showing `plantCount`, `locationCount`, and top `plants` array. Reduces 16,723 records to 69 summary rows while preserving analytical value.

---

## Debugging Iterations

### Issue 1: JSON parse errors in LLM responses
**Problem**: LLM occasionally wrapped JSON in markdown code fences
**Fix**: Added regex to extract JSON: `raw.match(/\{[\s\S]*\}/)`

### Issue 2: Graph nodes overlapping on load
**Problem**: All nodes initialized at center
**Fix**: Type-based cluster initialization — each entity type gets a grid position, nodes spread within cluster radius

### Issue 3: Query code failing for null fields
**Problem**: LLM-generated code crashed on `null` values in fields like `actualGoodsMovementDate`
**Fix**: Added to system prompt: "Handle null/undefined safely with || operators"

### Issue 4: Groq API CORS
**Problem**: Direct browser→Groq calls — needed CORS handling
**Fix**: Groq's API supports browser CORS directly (unlike some providers), no proxy needed

---

## Prompting Patterns That Worked Well

1. **Schema-first context**: Giving the LLM the complete field list upfront dramatically improved query accuracy
2. **Output format contract**: Specifying exact JSON structure (`{jsCode, explanation, insight}`) eliminated parsing failures
3. **Status code legend**: Including `A=Not started, B=Partial, C=Complete` in the prompt enabled correct filtering on delivery/billing statuses
4. **Index hints**: Telling the LLM about `IDX` shortcuts improved complex join queries significantly
5. **Example query patterns**: Including examples of broken-flow detection in the prompt enabled the LLM to handle "find incomplete O2C flows" correctly

---

## What Would Be Different at Scale

- **Graph DB**: Neo4j with Cypher for native graph traversal
- **Backend**: FastAPI with pre-computed graph analytics
- **NL→Cypher**: Specialized fine-tuned model for graph query generation
- **Streaming**: Server-sent events for progressive response rendering
- **Auth**: JWT for multi-user session isolation
