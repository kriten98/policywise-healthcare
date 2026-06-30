# Product Decisions Log

A record of the key decisions made during PolicyWise AI's V0 build — what was chosen, what was rejected, and why.

---

## 1. What to extract from a policy document

**Decision:** Extract only claim-critical information. Exclude wellness benefits, loyalty programs, concierge services, and promotional content.

**Rationale:** Indian health insurance policy documents are long and dense. Including every benefit creates noise that buries the information users most need — the clauses that reduce payouts or deny claims. A user facing hospitalisation needs to know their room rent cap and proportionate deduction clause. They do not need to know about their gym membership discount.

The extraction prompt enforces an explicit priority order: room rent → proportionate deduction → co-pay → deductibles → waiting periods → sub-limits → claim reduction factors. This ensures that if a document is unusually long or poorly structured, the highest-risk fields are captured first.

**What was rejected:** Extracting all policy benefits and letting the frontend filter by relevance. Rejected because LLM attention degrades on longer contexts — front-loading claim-critical extraction is more reliable than post-hoc filtering.

---

## 2. Null over guess

**Decision:** When a field is absent from the policy document, extract `null`. Never infer, estimate, or fill from general insurance knowledge.

**Rationale:** A fabricated sub-limit is more harmful than a missing one. If the model fills in a ₹40,000 cataract limit based on what's common in Indian policies, and the user's actual policy has no cataract sub-limit, the user enters surgery with incorrect expectations. Explicit nulls prompt the frontend to display "not specified in your policy" — which is the honest, safe output.

**What was rejected:** Allowing the model to use domain knowledge to fill in commonly standard values. Rejected because it erodes the reliability signal — users would have no way to distinguish extracted data from inferred data.

---

## 3. Model selection — Gemini 2.5 Flash

**Decision:** Use Gemini 2.5 Flash for both extraction and the RAG agent.

**Rationale:**
- Token efficiency: Indian health policy PDFs can run 40–80 pages. Flash handles long documents cost-effectively without sacrificing extraction quality.
- TAT: Flash produces faster responses than Pro, keeping the upload flow within the ~10 second target.
- Schema determinism: Flash 2.5 shows more consistent JSON schema compliance on structured extraction tasks — fewer thought tokens spent on reasoning, more reliable output shape.

**What was rejected:** GPT-4o was tested informally and showed comparable accuracy but higher latency and cost for this document type. Gemini's native PDF handling also removed a parsing preprocessing step that GPT-4o would have required.

---

## 4. Direct-context RAG over vector search

**Decision:** Inject the full policy JSON into the RAG agent's system prompt at query time, rather than using vector embeddings and similarity search.

**Rationale:** A structured policy JSON is small — typically 2–5KB. It fits comfortably within Gemini Flash's context window. Vector search adds retrieval latency, introduces the risk of retrieving the wrong chunk, and requires an embedding pipeline that adds infrastructure complexity. For a bounded, structured document of this size, direct-context injection is faster, simpler, and more reliable.

**What was rejected:** Building a vector store in Supabase (pgvector). Rejected at V0 because the document size doesn't justify the complexity. This decision will be revisited if policy documents grow significantly longer or if multi-document querying becomes a requirement.

---

## 5. n8n for pipeline orchestration

**Decision:** Use n8n (self-hosted via Docker) for workflow orchestration in V0.

**Rationale:** n8n allows rapid iteration on pipeline logic through a visual interface, without writing orchestration code. For a V0 where the pipeline steps are still being validated, this speeds up experimentation significantly. Self-hosting on Docker keeps data within a controlled environment.

**What was rejected:** Building a Python-based API pipeline from the start. Rejected for V0 because it would slow down iteration — orchestration code adds a development surface that isn't justified until the pipeline logic is stable.

**Planned transition:** Once the pipeline clears eval metric thresholds, n8n will be replaced with a FastAPI Python backend. This enables proper testing, CI/CD, and deployment flexibility that n8n cannot provide at scale.

---

## 6. Decouple frontend and backend at V0

**Decision:** Build and deploy the frontend prototype independently, using validated mock data, while the backend pipeline is evaluated separately.

**Rationale:** Connecting a frontend to a pipeline that hasn't cleared reliability thresholds creates a poor user experience and ships unvalidated outputs. Decoupling allows both layers to be developed and validated independently — the frontend can be user-tested for UX, the backend can be evaluated for accuracy, and integration happens only when both are ready.

**What was rejected:** Building a thin integration layer early and accepting some inaccuracy in the demo. Rejected because in a domain where wrong answers have real financial consequences, shipping a "good enough" pipeline is a product integrity risk, not just a quality issue.

---

## 7. Evaluation metrics selection

**Decision:** Evaluate on four domain-specific metrics — Coverage, Accuracy, Hallucination Rate, and Consistency.

**Rationale:**
- **Coverage:** Missing a claim-critical clause is not a minor recall issue — it means a user lacks information they need before hospitalisation.
- **Accuracy:** A wrong sub-limit or waiting period creates direct financial exposure.
- **Hallucination Rate:** Fabricated policy data is more dangerous than missing data — it creates false confidence.
- **Consistency:** Multiple users under the same policy must receive equivalent intelligence. Inconsistency means some users are unknowingly disadvantaged.

**What was rejected:** Standard NLP metrics (BLEU, ROUGE) — not relevant for structured JSON extraction. Generic LLM evals (helpfulness, coherence) — not calibrated to the financial consequences of errors in this domain.

---

## 8. Verdict framework in the RAG agent

**Decision:** Every RAG agent response must open with a structured verdict: ✅ Covered / ⚠️ Limited Coverage / ❌ Not Covered / ℹ️ Insufficient Information.

**Rationale:** Users asking "is knee replacement covered?" need a clear, immediate answer — not a paragraph they have to parse for the conclusion. The verdict framework forces the model into a classification-first pattern, reduces ambiguous hedging, and makes responses scannable. The detailed answer and policy evidence follow the verdict.

**What was rejected:** Free-form responses where the conclusion was embedded in prose. Rejected because informal testing showed users frequently missed the key information when it wasn't surfaced immediately.
