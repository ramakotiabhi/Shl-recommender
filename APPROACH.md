# Approach Document — SHL Assessment Recommender

## 1. Problem Decomposition

The task requires four capabilities:

1. **Clarification** — detect vague intent and ask focused follow-up questions
2. **Recommendation** — retrieve 1-10 catalog-grounded assessments
3. **Refinement** — update a shortlist mid-conversation without resetting context
4. **Comparison** — answer "what is the difference between X and Y?" accurately

The evaluator runs a stateless multi-turn harness, so the service must be fully stateless (full history on every call) and respond within 30 seconds.

---

## 2. Retrieval Strategy: Full-Catalog-in-Prompt

**Decision:** Inject the entire catalog (106 assessments × ~200 chars ≈ 21K tokens) into the system prompt instead of using a vector store.

**Rationale:**
- At 106 items, a RAG pipeline adds latency and retrieval errors without benefit
- Full context eliminates hallucination risk on names and URLs
- Enables accurate side-by-side comparisons without retrieval
- Claude claude-sonnet-4-20250514's 200K context window makes this trivially affordable

**Hallucination guard:** A post-generation validation layer checks every returned `name` and `url` against two Python sets built from the catalog. Invalid items are silently dropped before the response is serialized.

---

## 3. Agent Design

### State machine (implicit)

| Turn state | Condition | Action |
|---|---|---|
| Clarifying | First message is vague OR role/level unknown | Ask ONE focused question; return empty recommendations |
| Recommending | Role + level (or skill area) established | Return 1-10 assessments |
| Refining | User changes constraints | Update shortlist using full history |
| Comparing | User asks "difference between X and Y" | Answer from catalog data, no new shortlist needed |
| Refusing | Off-topic / injection / legal | Polite refusal, empty recommendations |
| Closing | User satisfied | `end_of_conversation: true` |

### Prompt engineering

The system prompt uses:
- **Explicit behavior rules** numbered 1-6 to reduce ambiguity
- **Catalog as structured text** (name | type | levels | duration | url) — compact enough to parse but rich enough to match
- **Mandatory JSON block** at the end of every response — parsed server-side to extract structured data
- **Test type legend** so the model correctly maps A/B/K/P/S to descriptions
- **Matching strategy** section guiding the model to mix cognitive + knowledge + personality tests

---

## 4. Tech Stack

| Component | Choice |
|---|---|
| API | FastAPI + Pydantic v2 |
| LLM | Claude claude-sonnet-4-20250514 (Anthropic SDK) |
| Storage | JSON file (no DB needed) |
| Deployment | Render (render.yaml auto-deploy) |

FastAPI was chosen for schema validation via Pydantic and automatic OpenAPI docs. Claude claude-sonnet-4-20250514 was chosen for strong instruction-following and long-context reliability.

---

## 5. What Didn't Work

**Attempt 1 — Structured output via tool calling:** I initially tried forcing structured output through Anthropic's tool_use feature. This worked but made it hard for the model to produce natural conversational replies alongside the structured data. Switched to embedded JSON block in the response.

**Attempt 2 — Semantic search (FAISS):** Tested a FAISS retrieval step before sending to the LLM. With only 106 items, it consistently retrieved irrelevant items (e.g. "Verify G+" for "general manager role"). Full-catalog injection outperformed retrieval on both precision and recall in informal testing.

---

## 6. Evaluation Approach

**Schema compliance**: Pydantic enforces the response shape; catalog validation prevents hallucinated items.

**Recall@10**: The model sees all 106 assessments and is instructed to diversify across test types. For Java developer queries, it reliably returns Java 8 (New), Core Java, Java Spring, Verify Numerical, OPQ32r — covering both technical and behavioral dimensions.

**Behavior probes tested manually:**
- Vague query → clarifies (no recommendations on turn 1) ✓
- Off-topic → polite refusal ✓
- Refinement → adds personality test when asked ✓
- Comparison → uses catalog descriptions, no hallucination ✓
- Prompt injection → treated as off-topic, refused ✓

**AI tools used:** Claude claude-sonnet-4-20250514 assisted with boilerplate code generation and test case drafting. All design decisions and prompt engineering were done manually and can be defended in interview.
