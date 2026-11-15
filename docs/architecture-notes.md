# Architecture Notes — RAG + MCP Customer Support Assistant

> This document provides detailed technical notes for each component in the architecture diagram.
> It is intended as a companion to the README.md for deeper implementation guidance.

---

## 1. System Overview

The architecture is a **stateless, orchestration-first** design that combines:

- **RAG (Retrieval-Augmented Generation)** — to ground LLM answers in verified support content.
- **MCP (Model Context Protocol)** — to expose external support operations as well-typed, reusable tools.

No persistent agent loop or memory store is included. Each request is handled end-to-end in a single orchestration cycle.

---

## 2. Component Responsibilities

### A. Customer / User

| Attribute        | Detail                                             |
|------------------|----------------------------------------------------|
| **Role**         | External actor initiating the support interaction  |
| **Input**        | Natural-language question or support request       |
| **Output**       | Final generated answer returned by API Gateway     |
| **Protocol**     | HTTPS REST or WebSocket                            |

**Notes:** The customer never interacts directly with the LLM or any internal service. All traffic enters and exits through the API Gateway.

---

### B. API Gateway

| Attribute        | Detail                                             |
|------------------|----------------------------------------------------|
| **Role**         | Single entry/exit point for all external traffic   |
| **Responsibilities** | Auth/token validation, rate limiting, routing, response serialisation |
| **Inbound**      | HTTP POST with JSON payload (`{ "query": "..." }`) |
| **Outbound**     | JSON response (`{ "answer": "...", "sources": [] }`) |

**Design rationale:** Centralises cross-cutting concerns (auth, logging, TLS termination) so the LLM Orchestrator can remain stateless and focused on generation logic.

---

### C. LLM Orchestrator

| Attribute        | Detail                                              |
|------------------|-----------------------------------------------------|
| **Role**         | Core decision engine — routes, retrieves, generates |
| **Model options**| GPT-4o, Gemini 1.5 Flash, Claude 3.5 Sonnet         |
| **Decision logic** | Intent classification → conditional RAG → conditional MCP → answer synthesis |
| **Input**        | User query + (optionally) retrieved context + (optionally) tool results |
| **Output**       | Final natural-language answer                       |

**Conditional routing:**
1. If the query requires factual support knowledge → trigger RAG Retriever.
2. If the query requires a system action (ticket, order check) → trigger MCP tool call.
3. Both paths can be active in one request (e.g., retrieve FAQ *and* create a ticket).

**System prompt responsibilities:**
- Define persona and tone.
- Specify how to format retrieved context.
- Define how to invoke MCP tools (function-calling schema).

---

### D. RAG Retriever

| Attribute        | Detail                                               |
|------------------|------------------------------------------------------|
| **Role**         | Semantic search over the support knowledge corpus    |
| **Input**        | User query string                                    |
| **Process**      | Embed query → ANN vector search → rerank (optional) → return top-k chunks |
| **Output**       | List of `{ chunk_text, source_doc, score }` objects  |
| **Libraries**    | LangChain retriever, LlamaIndex, or custom           |

**Retrieval is conditional:** The LLM Orchestrator only calls the retriever when it determines the available context is insufficient to answer the question directly (e.g., the LLM lacks real-time or domain-specific knowledge).

**Chunk strategy:**
- Chunk size: 300–500 tokens with 50-token overlap.
- Embedding model: `text-embedding-3-small` (OpenAI) or `embedding-001` (Google).
- Top-k: typically 3–5 chunks per query.

---

### E. Vector Database

| Attribute        | Detail                                              |
|------------------|-----------------------------------------------------|
| **Role**         | Persistent store for support content embeddings     |
| **Content**      | FAQs, troubleshooting guides, product documentation |
| **Technology**   | Pinecone, Weaviate, Qdrant, pgvector, or Chroma     |
| **Index type**   | HNSW (Hierarchical Navigable Small World) for ANN   |
| **Populated by** | Offline ingestion pipeline from Knowledge Source    |

**Ingestion pipeline (offline, not in real-time request path):**
```
Knowledge Source → Chunker → Embedding Model → Vector DB
```
The ingestion pipeline runs periodically or on document update events, not on every user request.

---

### F. MCP Server

| Attribute        | Detail                                               |
|------------------|------------------------------------------------------|
| **Role**         | Standardised gateway exposing external operations as typed tools |
| **Protocol**     | Model Context Protocol (MCP) — JSON-RPC-based        |
| **Tools exposed**| `create_support_ticket`, `check_order_status`        |
| **Calls**        | External Support API via HTTPS                       |
| **Returns**      | Structured JSON result to LLM Orchestrator           |

**Defined Tools:**

```json
{
  "tools": [
    {
      "name": "create_support_ticket",
      "description": "Creates a new support ticket for a customer issue.",
      "parameters": {
        "customer_id": { "type": "string", "required": true },
        "issue_description": { "type": "string", "required": true }
      }
    },
    {
      "name": "check_order_status",
      "description": "Returns the current status of a customer order.",
      "parameters": {
        "order_id": { "type": "string", "required": true }
      }
    }
  ]
}
```

**Why MCP instead of direct API calls?**
MCP decouples tool definitions from the LLM client. New tools can be added server-side without changing LLM prompt engineering. Enables tool reuse across multiple LLM applications.

---

### G. External Support API

| Attribute        | Detail                                              |
|------------------|-----------------------------------------------------|
| **Role**         | Third-party system performing actual operations     |
| **Ownership**    | External (not part of our system)                   |
| **Operations**   | Ticket management, order management                 |
| **Interface**    | REST API (called by MCP Server only)                |
| **Examples**     | Zendesk, Salesforce Service Cloud, Shopify, custom  |

**Security note:** Credentials for the External Support API are stored in the MCP Server's secrets manager, never exposed to the LLM or the customer.

---

### H. Knowledge Source

| Attribute        | Detail                                              |
|------------------|-----------------------------------------------------|
| **Role**         | Third-party repository of support documentation     |
| **Ownership**    | External (not part of our system)                   |
| **Content**      | FAQs, product manuals, troubleshooting articles     |
| **Interface**    | File export, S3 bucket, CMS API, or web scrape      |
| **Consumer**     | Offline ingestion pipeline → Vector DB              |

**Update cadence:** Triggered on document change events or run on a nightly schedule to keep the vector index fresh.

---

### I. Eval Collector

| Attribute        | Detail                                               |
|------------------|------------------------------------------------------|
| **Role**         | Passive monitoring and evaluation data aggregator    |
| **Inputs**       | Retrieved chunks (from RAG), LLM response (from Orchestrator), tool call + result (from MCP Server) |
| **Storage**      | Append-only evaluation log (e.g., JSONL, BigQuery, Postgres) |
| **Consumers**    | Offline evaluation scripts, dashboards              |

**Data collected per request:**

```jsonc
{
  "request_id": "uuid",
  "timestamp": "ISO-8601",
  "user_query": "...",
  "retrieved_chunks": [{ "text": "...", "score": 0.92 }],
  "llm_response": "...",
  "tool_calls": [{ "name": "create_support_ticket", "args": {}, "result": {} }],
  "latency_ms": { "retrieval": 180, "llm": 1200, "tool": 420 }
}
```

**Evaluation metrics powered by this data:**

| Metric                 | Description                                         |
|------------------------|-----------------------------------------------------|
| Retrieval Recall       | Were the relevant docs retrieved?                   |
| Answer Faithfulness    | Does the answer stay within retrieved context?      |
| Tool-Call Correctness  | Were the right tools called with valid arguments?   |
| End-to-End Latency     | Time from customer query to final response          |

**Important:** The Eval Collector is **write-only** in the request path — it never feeds data back into the LLM or affects the response. It is a passive observer.

---

## 3. Data Flow Summary

```
Customer
  │
  ▼  (HTTPS query)
API Gateway
  │
  ▼  (forward query)
LLM Orchestrator
  ├─ [if context needed] ──► RAG Retriever ──► Vector DB
  │                             ◄── top-k chunks ──┘
  │                          ◄── context ──┘
  │
  ├─ [if action needed] ──► MCP Server ──► External Support API
  │                           ◄── tool result ──┘
  │
  ▼  (generate final answer)
LLM Orchestrator
  │
  ▼  (return answer)
API Gateway
  │
  ▼
Customer

Monitoring (dashed, out-of-band):
  RAG Retriever ──► Eval Collector
  LLM Orchestrator ──► Eval Collector
  MCP Server ──► Eval Collector
```

---

## 4. System Boundary Summary

| Component           | Owner       | Notes                                   |
|---------------------|-------------|-----------------------------------------|
| API Gateway         | Our System  | We build, deploy, operate               |
| LLM Orchestrator    | Our System  | We prompt-engineer, host, monitor       |
| RAG Retriever       | Our System  | We index, tune retrieval parameters     |
| Vector Database     | Our System  | We manage index and ingestion           |
| MCP Server          | Our System  | We define and maintain tool contracts   |
| Eval Collector      | Our System  | We own logging and evaluation pipeline  |
| External Support API| Third-Party | We consume via MCP Server only          |
| Knowledge Source    | Third-Party | We ingest; we do not control content    |

---

## 5. Architecture Trade-offs

### RAG
- ✅ Improves factual grounding and reduces hallucination.
- ✅ Knowledge updates without LLM retraining (update the index).
- ⚠️ Adds retrieval latency (~100–300 ms).
- ⚠️ Requires embedding pipeline and vector database infrastructure.

### MCP
- ✅ Reusable, versioned tool contracts — LLM-agnostic.
- ✅ Isolates external API credentials from LLM context.
- ⚠️ Requires maintaining MCP Server and tool schemas.
- ⚠️ Tool call adds round-trip latency (~200–600 ms).

### No Autonomous Agent
An autonomous agent loop (plan → act → observe → repeat) is intentionally excluded because:
- This use case requires only **conditional single-step** retrieval and tool calls.
- Autonomous agents introduce non-determinism, harder debugging, and unnecessary complexity.
- The LLM Orchestrator already handles intent classification and conditional routing without a full agent loop.

### Eval Collector
- ✅ Enables continuous quality monitoring and regression detection.
- ⚠️ Adds logging overhead and storage costs.
- ⚠️ Must be async or fire-and-forget to avoid increasing response latency.

---

*Architecture Notes — RAG + MCP Customer Support Assistant*  
*Version 1.0 — 2026*
