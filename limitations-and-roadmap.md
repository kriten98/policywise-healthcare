# Limitations & Roadmap

## Current Limitations (V0)

### Pipeline

**No real policy document testing yet**
V0 evaluation was conducted on synthetic, AI-generated policy documents. Real policies introduce variables — scan quality, OCR errors, non-standard formatting, regional language inserts — that synthetic documents don't capture. Performance on real-world documents may differ from V0 eval results.

**Policy A hallucination outlier**
Policy A produced a 31% hallucination rate during V0 evaluation, compared to 4–14% across other policies. The suspected cause is non-standard document structure disrupting the extraction prompt's priority ordering. This is unresolved in V0 and is a target for prompt hardening in V1.

**Proportionate deduction calculation accuracy**
The financial impact calculation for proportionate deduction (applying the room breach ratio across all line items) is complex. The RAG agent sometimes applies this calculation incorrectly. Flagged under the Accuracy metric; a calculation verification step is planned.

**Waiting period disambiguation**
Policies that define overlapping waiting period categories (e.g. a specific disease that is also a pre-existing condition) sometimes produce conflicting waiting period values in the agent's response. A disambiguation rule is planned for V1.

**No metric thresholds cleared yet**
All four V0 eval metrics are within 3 percentage points of target but none have crossed their threshold. FE–BE integration is gated on meeting all four targets.

### Frontend

**Mock data only**
The live prototype runs on a validated mock policy (Star Health Family Floater). It is not connected to the backend pipeline. This is intentional — see product decisions doc for rationale.

**No authentication or multi-user support**
V0 is a single-user prototype. User accounts, policy management (storing multiple policies per user), and session handling are not implemented.

**No mobile-native upload flow**
The PDF upload is functional on desktop. Mobile upload experience (particularly from camera/files on iOS and Android) has not been tested or optimised.

---

## Roadmap

### V1 — Pipeline Reliability

- [ ] Harden extraction prompt for non-standard document structures (targeting Policy A regression)
- [ ] Add proportionate deduction calculation verification step in the JavaScript node
- [ ] Expand eval set to 15+ synthetic policies with deliberately varied structures
- [ ] Introduce real policy documents (anonymised) to the eval set
- [ ] Add per-field-type extraction breakdown to eval framework
- [ ] Clear all four metric thresholds

### V1.5 — Integration

- [ ] Replace n8n with FastAPI Python backend
- [ ] Connect frontend to live pipeline
- [ ] End-to-end upload → dashboard flow with real policy documents
- [ ] Add user authentication (Supabase Auth)
- [ ] Multi-policy support per user

### V2 — Product Depth

- [ ] Pre-hospitalisation checklist — actionable steps before admission based on the user's specific policy
- [ ] Claim risk simulator — "if I choose this room, here's the financial impact"
- [ ] Policy comparison — side-by-side comparison of two policies across key dimensions
- [ ] Renewal intelligence — flag deteriorating terms on renewal documents vs. previous year
- [ ] Notification layer — alert users when cashless intimation deadlines are approaching during an active hospitalisation

### V3 — Scale

- [ ] Support for group health policies (employer-provided)
- [ ] Insurer-specific extraction rules (some insurers use non-standard clause structures)
- [ ] Hindi and regional language policy document support
- [ ] API layer for potential integration with insurance aggregators or hospital platforms
