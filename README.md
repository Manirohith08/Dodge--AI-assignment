# O2C Graph Intelligence Platform

A graph-based data modeling and natural language query system for SAP Order-to-Cash (O2C) business process data.

**Live Demo**: Open `src/index.html` in any modern browser — no server required.

---

## What It Does

This application ingests a full SAP O2C dataset (13 tables, 17,756 records) and builds:

1. **An interactive graph visualization** — entities as nodes, business relationships as edges, rendered with a force-directed physics layout
2. **A conversational query interface** powered by Groq's LLaMA-3.3-70b model — ask anything about orders, customers, deliveries, invoices, payments, and products in plain English

---

## Architecture

### Frontend-Only, Zero Backend
The entire application is a single `index.html` file (~750KB). All data is embedded as JavaScript objects. No server, no database, no build step.

### Data Layer
All 13 SAP tables are loaded at startup:

| Table | Records | Purpose |
|---|---|---|
| `sales_order_headers` | 100 | Core O2C starting point |
| `sales_order_items` | 167 | Line items with materials |
| `outbound_delivery_headers` | 86 | Delivery execution |
| `billing_document_cancellations` | 80 | Invoices / billing docs |
| `journal_entry_items_accounts_receivable` | 123 | Accounting entries |
| `payments_accounts_receivable` | 120 | Payment clearing |
| `business_partners` | 8 | Customer master data |
| `customer_company_assignments` | 8 | Company-level config |
| `customer_sales_area_assignments` | 28 | Sales area config |
| `plants` | 44 | Plant/location master |
| `product_descriptions` | 69 | Product catalog |
| `product_plants` | 200 | Product-plant assignments |
| `product_storage_summary` | 69 | Storage location summary |

Pre-built lookup indices (`IDX`) enable fast cross-table joins in generated queries.

### Graph Model

**Nodes (9 types)**:
- `Customer` — business partners and sold-to parties
- `SalesOrder` — order headers
- `SalesOrderItem` — line items
- `Product` — materials with descriptions
- `Delivery` — outbound delivery documents
- `BillingDoc` — invoices
- `JournalEntry` — accounting documents
- `Payment` — clearing entries
- `Plant` — shipping/production locations

**Edges (typed relationships)**:
- `PLACED_ORDER` — Customer → SalesOrder
- `HAS_ITEM` — SalesOrder → SalesOrderItem
- `FOR_PRODUCT` — SalesOrderItem → Product
- `SHIPPED_FROM` — Delivery → Plant
- `BILLED_TO` — Customer → BillingDoc
- `GENERATES_JE` — BillingDoc → JournalEntry
- `MADE_PAYMENT` — Customer → Payment
- `PAYS_FOR` — Payment → SalesOrder

### LLM Query Engine
- **Model**: Groq `llama-3.3-70b-versatile` (free tier)
- **Strategy**: The LLM receives the full schema + field descriptions, then generates a JavaScript expression that operates on the `DATA` object directly
- **Execution**: Generated code runs in a sandboxed `new Function('DATA', 'IDX', code)` call — no eval, no server
- **Guardrails**: System prompt strictly limits responses to dataset questions; off-topic queries return a rejection message
- **Context**: Last 12 messages maintained for multi-turn conversations

### LLM Prompting Strategy

The system prompt includes:
1. Full schema with all field names and their meanings
2. Status code legend (A/B/C delivery/billing statuses)
3. Join key relationships (which fields connect which tables)
4. Available index shortcuts for performance
5. The O2C flow diagram in text form
6. Strict output format instructions (JSON only, no markdown)
7. Explicit guardrail instruction to reject off-topic queries

The LLM generates JavaScript instead of SQL because:
- No SQL engine is available in the browser
- JavaScript array methods (`filter`, `map`, `reduce`) handle all required query patterns
- Generated code can use pre-built indices for complex joins
- Results are immediately available as JavaScript objects for rendering

### Guardrails
- Off-topic questions (general knowledge, coding help, creative writing, math) are rejected with a clear message
- The system prompt explicitly forbids answering anything not about the O2C dataset
- LLM temperature set to 0.1 for deterministic query generation

---

## Setup & Running

### Requirements
- A modern browser (Chrome, Firefox, Safari, Edge)
- A free [Groq API key](https://console.groq.com) (takes 30 seconds to get)

### Steps
```bash
# Clone the repo
git clone <your-repo-url>
cd o2c-graph-intelligence

# Open directly in browser
open src/index.html
# or double-click src/index.html
```

No npm install, no Python, no docker. Just open the file.

### Getting a Groq API Key
1. Go to [console.groq.com](https://console.groq.com)
2. Sign up (free, no credit card)
3. Create an API key
4. Paste it into the field at the top of the app

---

## Example Queries

### Data Queries
- "Which products appear in the most billing documents?"
- "Top 5 customers by total sales order value"
- "What is the total revenue across all sales orders?"
- "Show orders created in March 2025"
- "Average order value by sales organization"

### O2C Flow Analysis
- "Trace complete O2C flow for billing document 90504274"
- "Find all sales orders with no delivery (not yet shipped)"
- "Find orders that are delivered but not billed"
- "Find billed orders with no payment recorded"
- "Show all orders with incomplete O2C flow"

### Entity Exploration
- "List all customers with their names and order counts"
- "Show all products with descriptions and quantity ordered"
- "Which plants handle the most shipments?"
- "Customers with blocked billing"

---

## Database / Storage Decision

**Why no database?** The dataset is small enough (~450KB of JSON) to embed directly in the HTML. This gives:
- Zero infrastructure cost
- Instant startup (no network round-trips for data)
- Fully offline capability
- Single-file deployment

For a production system with millions of records, the right choice would be PostgreSQL (relational joins) or Neo4j (native graph traversal). The graph model built here maps directly to Neo4j's Cypher query language.

---

## Files

```
/
├── src/
│   └── index.html          # Complete application (self-contained)
├── sessions/
│   └── claude_session.md   # AI coding session log
├── README.md
```

---

## Trade-offs & Known Limitations

- **Dataset size**: `product_storage_locations` (16,723 records) is summarized to avoid bloating the file — all other tables are loaded in full
- **LLM accuracy**: Complex multi-table joins may occasionally produce incorrect JS; the interface shows the generated code for transparency
- **Graph layout**: Force simulation runs for 300 steps then freezes — use filter chips to focus on relevant entity types
- **No persistence**: Conversation history resets on page refresh

---

## Bonus Features Implemented

- ✅ Natural language to JavaScript query translation (equivalent to NL→SQL)
- ✅ Node highlighting on query results — nodes matching result rows glow on the graph
- ✅ Graph node search with text filter
- ✅ Conversation memory (last 12 turns)
- ✅ Pre-built lookup indices for fast cross-table queries
- ✅ Three query modes: Query Data / Explore Graph / Trace Flow
- ✅ Type-based graph filtering with live physics re-layout
