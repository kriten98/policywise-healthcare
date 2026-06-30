# System Architecture

## Overview

PolicyWise AI uses two decoupled n8n flows to handle policy ingestion and query resolution. The frontend is intentionally separated from the backend pipeline during the V0 evaluation phase.

---

## Flow 1 — Upload & Extract

**Purpose:** Convert a raw policy PDF into a structured, claim-critical JSON object stored in Supabase.

```
User uploads PDF
       ↓
Webhook (n8n trigger)
       ↓
Gemini API — Document Analysis
  - Full PDF passed as input
  - System prompt enforces strict JSON-only output
  - Extraction priority: room rent → proportionate deduction → co-pay → deductibles → waiting periods → sub-limits
       ↓
JavaScript Node — Schema Validation & Cleaning
  - Enforces output schema
  - Handles null values for missing fields
  - Strips any non-JSON artefacts
       ↓
Supabase — Insert Row
  - One row per policy
  - policy_rules_json column stores the full structured object
  - Indexed by user/policy identifier
```

**Target latency:** ~10 seconds end-to-end from upload to dashboard render.

---

## Flow 2 — Query & Respond (RAG)

**Purpose:** Answer natural language questions about a user's specific policy, grounded strictly in their stored policy JSON.

```
User submits query
       ↓
Webhook (n8n trigger)
       ↓
Supabase — Get Row
  - Retrieves stored policy_rules_json for the user
       ↓
Gemini 2.5 Flash AI Agent
  - Policy JSON injected directly into system prompt context
  - Agent instructed to use only provided data — no external knowledge
  - Structured verdict format: ✅ Covered / ⚠️ Limited / ❌ Not Covered / ℹ️ Insufficient Data
       ↓
Webhook Response
  - Returns verdict + answer + policy evidence + financial impact
```

This is a direct-context RAG pattern — the full policy JSON is passed in the system prompt rather than retrieved via vector search. This works well given the bounded size of a structured policy JSON object.

---

## Frontend

Built with Next.js 15 (App Router), deployed on Cloudflare Pages.

Currently decoupled from the backend pipeline. The prototype renders a validated mock policy (Star Health Family Floater) to demonstrate the full dashboard experience while the pipeline is evaluated against reliability thresholds.

**Components:**
- `overview-cards.tsx` — Insurer, expiry, sum insured, risk level
- `claim-reduction-section.tsx` — Ranked list of claim-reducing clauses
- `policy-health-score.tsx` — Radial gauge (coverage quality, claim friendliness, risk exposure)
- `analytics-section.tsx` — Coverage distribution, risk factor bars, benefit categories
- `ai-chat.tsx` — Chat interface connected to retrieval flow
- `upload-modal.tsx` — Drag-and-drop PDF upload with processing state

---

## Infrastructure

| Component | Tool | Hosting |
|---|---|---|
| Workflow orchestration | n8n | Self-hosted, Docker |
| Database | Supabase | Managed (Supabase Cloud) |
| Frontend | Next.js 15 | Cloudflare Pages |
| AI Model | Gemini 2.5 Flash | Google AI API |

---

## Design Decisions

**Why direct-context RAG over vector search?**
A structured policy JSON is small enough (~2–5KB) to fit comfortably in a Gemini Flash context window. Vector search adds latency and retrieval error risk for no meaningful gain at this document size. Passing the full JSON ensures the agent always has complete context.

**Why decouple FE and BE at V0?**
Connecting the frontend to a pipeline that hasn't cleared evaluation thresholds would risk shipping unreliable outputs to users. Decoupling lets the pipeline be validated independently. Integration happens post-eval.

**Why n8n for orchestration?**
n8n allows rapid iteration on workflow logic without writing orchestration code. The plan is to replace it with a Python-based API layer once the pipeline logic is stable and metrics are met.

---

## Planned Architecture (V1)

```
FastAPI (Python)
  ├── /upload  → PDF parsing → Gemini extraction → Supabase upsert
  └── /query   → Supabase retrieval → Gemini agent → structured response

Next.js Frontend
  └── API calls to FastAPI endpoints
```

n8n will be retired in favour of a fully API-integrated Python backend for reliability, testability, and deployment flexibility.
