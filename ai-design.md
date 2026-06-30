# AI & LLM Design Decisions

## Model Choice — Gemini 2.5 Flash

**Why Gemini over GPT:**
- Better token efficiency for long PDF documents — Indian health insurance policies regularly run 40–80 pages
- Faster TAT (turnaround time) for structured extraction tasks, which matters for the ~10s target latency in the upload flow
- Native support for document inputs, reducing the need for a separate PDF parsing layer

**Why Flash over Pro:**
- Flash 2.5 produces more deterministic JSON schema compliance — fewer thought tokens spent on reasoning, more consistent output structure
- Lower hallucination variance on structured extraction tasks compared to more generative model configurations
- Cost-efficient at the extraction volume needed for V0 testing

---

## Prompt Architecture

Two distinct system prompts govern the two flows. They are intentionally designed with different objectives.

### Extraction Prompt (Flow 1)

**Primary goal:** Convert a PDF into a clean, structured JSON object with zero fabrication.

Key design decisions:

**Strict JSON-only output**
The prompt explicitly prohibits markdown, explanations, code blocks, and any text before or after the JSON. This is enforced because n8n's downstream JavaScript node expects raw JSON — any wrapper causes parsing failure.

**Null over guess**
The model is instructed to use `null` for unavailable fields rather than inferring or estimating. In an insurance context, a fabricated sub-limit or waiting period is more harmful than an empty field — it creates false confidence.

**Extraction priority order**
Fields are extracted in a deliberate sequence reflecting financial risk:
1. Room rent limits — highest claim impact
2. Proportionate deduction clauses — can reduce entire claim, not just room rent
3. Co-pay obligations
4. Deductibles
5. Waiting periods
6. Disease/procedure-specific sub-limits
7. Claim reduction conditions

This ordering ensures that if the model hits context or attention limits on a very long document, the most financially consequential fields are extracted first.

**Claim-critical filter**
The prompt explicitly excludes wellness programs, loyalty rewards, concierge services, and promotional content. These create noise and can distract the model from extracting the fields that matter.

**Waiting period extraction rule**
For waiting periods with multiple durations (common in Indian policies), the model extracts only the shortest applicable value. This surfaces the earliest claim eligibility date — the number a user most needs to act on.

→ Full prompt: [`/prompts/extraction-prompt.md`](../prompts/extraction-prompt.md)

---

### RAG Agent Prompt (Flow 2)

**Primary goal:** Answer user questions accurately using only their policy's stored JSON — no external knowledge, no fabrication.

Key design decisions:

**Closed-world instruction**
The agent is explicitly told: "Use only POLICY DATA." It is prohibited from inventing coverage, exclusions, limits, waiting periods, penalties, timelines, or benefits. This is the single most important constraint — users will make real financial decisions based on these answers.

**Policy JSON injected into system prompt**
The user's `policy_rules_json` is stringified and embedded directly in the system prompt at query time. This is a direct-context RAG pattern — more reliable than vector retrieval for structured data of this size, and eliminates retrieval errors.

**Field priority hierarchy**
The agent is given an explicit field priority order to resolve conflicts between structured fields and summary fields:
1. `room_limits`
2. `copay`
3. `hidden_caps / sub_limits`
4. `waiting_periods`
5. `claim_penalties`
6. `claim_timelines`
7. `risk_flags`
8. `policy_summary.smart_summary`

If two fields conflict, the more specific structured field wins over the summary.

**Verdict framework**
Every response is required to open with a verdict label:
- ✅ **Covered** — coverage exists without a relevant restriction
- ⚠️ **Limited Coverage** — covered but subject to caps, waiting periods, co-pay, or proportionate deductions
- ❌ **Not Covered** — explicitly excluded
- ℹ️ **Insufficient Information** — policy data does not contain enough to answer

This forces the model into a classification-first response pattern, reducing ambiguous or hedged answers that leave users uncertain about their coverage.

**Financial impact field**
Every response includes a `FINANCIAL IMPACT` section when enough data exists. This translates policy rules into rupee terms — e.g. "if your room costs ₹8,000/day and your cap is ₹5,000, proportionate deduction reduces your total admissible claim by approximately 37%."

→ Full prompt: [`/prompts/rag-agent-prompt.md`](../prompts/rag-agent-prompt.md)

---

## Known LLM Risks

**Proportionate deduction calculation errors**
The financial impact calculation for proportionate deduction is complex (room breach ratio applied across all line items). The model sometimes applies the ratio incorrectly. This is flagged as a known limitation and is being monitored in the eval framework under the Accuracy metric.

**Policy A hallucination spike**
During V0 evaluation, Policy A produced a 31% hallucination rate — significantly higher than the 4–14% range seen across other policies. The likely cause is policy document structure: Policy A used non-standard section headers and nested clause references that disrupted the extraction prompt's priority ordering. This is documented in the eval results and is a target for prompt refinement in V1.

**Waiting period ambiguity**
Some policies define overlapping waiting period categories (e.g. a specific disease that is also a pre-existing condition). The extraction prompt resolves this by taking the shortest value, but the agent occasionally surfaces both values, creating confusion. A disambiguation rule is planned for V1.
