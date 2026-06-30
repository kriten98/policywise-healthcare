# PolicyWise AI

> **AI-powered Health Insurance Intelligence Platform**
> Helping Indian policyholders understand what their policy actually covers — before they need to use it.

🔗 [Live Prototype](https://policywise-ai.pages.dev/) &nbsp;|&nbsp; 📊 [Eval Results (V0)](https://docs.google.com/spreadsheets/d/1kK7Gq4IvonJQ_AiVhOXV7-NOdt5EjBIkqyTefpRmk58/)

---

## The Problem

Most Indian health insurance policyholders only discover the fine print during a claim — when it's too late. Room rent caps trigger proportionate deductions that can slash payouts by 30–40%. Co-pay clauses, sub-limits, and cashless intimation deadlines are buried in 40-page policy PDFs written in legal language.

The result: patients end up paying far more out-of-pocket than they expected, not because they weren't covered, but because they didn't understand how their coverage worked.

PolicyWise AI solves this by turning a policy PDF into a structured intelligence dashboard — surfacing every clause that could reduce a claim, before hospitalization happens.

---

## What It Does

**For a new user:**
1. Upload a health insurance policy PDF
2. The AI pipeline extracts all claim-critical information within ~10 seconds
3. A structured dashboard renders — showing benefits, sub-limits, claim risks, and a policy health score

**For a returning user:**
- Dashboard loads instantly from stored policy data
- A chat interface lets users ask natural language questions about their specific policy
- The AI agent answers using only the user's actual policy data — no hallucinated coverage

---

## Key Features

| Feature | Description |
|---|---|
| Policy Dashboard | Insurer, sum insured, expiry, and coverage overview |
| Claim Reduction Intelligence | Flags clauses that can reduce payout — room rent caps, proportionate deductions, co-pay |
| AI Insurance Assistant | RAG-powered chat grounded exclusively in the user's policy JSON |
| Policy Health Score | AI-assessed score across coverage quality, claim friendliness, and risk exposure |
| Analytics | Coverage distribution, risk factor breakdown, benefit category view |

---

## Architecture

Two decoupled flows, orchestrated via n8n (self-hosted on Docker):

**Flow 1 — Upload & Extract**
```
PDF Upload → Gemini API (structured extraction) → JavaScript (schema validation) → Supabase (storage)
```

**Flow 2 — Query & Respond**
```
User Query → Supabase (policy JSON retrieval) → Gemini 2.5 Flash AI Agent (RAG) → Response
```

The frontend (Next.js 15) is intentionally decoupled from the backend pipeline. The current prototype runs on validated mock data while the pipeline is evaluated against reliability thresholds.

→ See [`/docs/architecture.md`](./docs/architecture.md) for full design detail.

---

## Evaluation (V0)

Tested across 6 synthetic policy documents. Metrics designed for the domain — insurance literacy errors have real financial consequences.

| Metric | Score |
|---|---|
| Coverage | 87% |
| Accuracy | 89% |
| Hallucination Rate | 12% |
| Consistency | 88% |

Policy A showed an outlier hallucination rate of 31%, flagged and under investigation. All other policies ranged 4–14%.

→ See [`/docs/eval-framework.md`](./docs/eval-framework.md) for methodology and per-policy breakdown.

---

## Tech Stack

| Layer | Tool | Rationale |
|---|---|---|
| AI Extraction | Google Gemini API | Token efficiency, structured output determinism |
| LLM Model | Gemini 2.5 Flash | Lower thought token usage, more deterministic JSON schema compliance |
| Orchestration | n8n (self-hosted, Docker) | Rapid pipeline iteration for V0 |
| Database | Supabase | AI-ready storage, native JSON support, SQL querying |
| Frontend | Next.js 15 + TypeScript | App Router, component-driven, production-deployable |
| Styling | TailwindCSS + Framer Motion | Utility-first with animation support |
| Charts | Recharts | Lightweight, composable data visualization |
| Prototyping | Claude (Anthropic) | Rapid UI iteration and component scaffolding |

→ See [`/docs/product-decisions.md`](./docs/product-decisions.md) for the reasoning behind each choice.

---

## Repository Structure

```
policywise-ai/
├── README.md
├── docs/
│   ├── architecture.md          # System design and flow detail
│   ├── data-schema.md           # Supabase schema + JSON output schema
│   ├── ai-design.md             # LLM design decisions and prompt rationale
│   ├── eval-framework.md        # Evaluation methodology and results
│   ├── product-decisions.md     # Key PM decisions and tradeoffs
│   └── limitations-and-roadmap.md
├── prompts/
│   ├── extraction-prompt.md     # Gemini system prompt for Upload flow
│   └── rag-agent-prompt.md      # AI agent system prompt for Retrieval flow
└── samples/
    └── sample-json-output.md    # Example structured output from the pipeline
```

---

## Current Status

**V0 — Evaluation Phase**

- ✅ Backend pipeline built and tested (n8n + Gemini + Supabase)
- ✅ Frontend prototype deployed
- ✅ Eval framework designed and run across 6 synthetic policies
- ⏳ FE–BE integration pending post-eval metric threshold
- ⏳ Real policy document testing in progress

The frontend and backend are deliberately kept decoupled at this stage. Integration will happen once the pipeline clears reliability thresholds across real-world policy documents.

---

## About

Built by an APM exploring the intersection of AI and financial literacy in Indian healthcare. This project is part of a portfolio demonstrating end-to-end AI product thinking — from problem scoping through architecture, evaluation, and iteration.
