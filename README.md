# SHL Assessment Recommender

A conversational FastAPI agent that recommends SHL Individual Test Solutions through natural dialogue.

## Architecture

```
POST /chat ──► Claude claude-sonnet-4-20250514 (system prompt with full catalog)
                    │
                    ├── Clarifies vague queries
                    ├── Recommends 1-10 catalog assessments
                    ├── Refines on constraint changes
                    ├── Compares assessments using catalog data
                    └── Refuses off-topic / injection attempts

GET /health ──► {"status": "ok"}
```

## Stack

| Layer | Choice | Reason |
|---|---|---|
| API framework | FastAPI + Pydantic v2 | Fast, schema-validated, auto-docs |
| LLM | Claude claude-sonnet-4-20250514 via Anthropic SDK | Strong instruction-following, no extra infra |
| Catalog storage | JSON file (catalog.json) | 106 assessments — no vector DB needed at this scale |
| Retrieval | Full catalog injected in system prompt | Eliminates retrieval latency & hallucination risk |
| Deployment | Render (Docker/Python runtime) | Free tier, fast cold starts, env var support |

## Design Decisions

### Why full-catalog-in-prompt instead of RAG?
The catalog is 106 items × ~200 tokens each ≈ 21K tokens — well within Claude's context window. Injecting the full catalog eliminates retrieval errors entirely and guarantees the model can answer comparison questions accurately. For catalogs >500 items, a vector store (Chroma/FAISS) would be appropriate.

### Hallucination guard
Every returned recommendation is validated against the catalog's name and URL sets at the API layer. Hallucinated items are silently dropped before the response leaves the server.

### Stateless by design
The `/chat` endpoint accepts the full conversation history on every call. No per-session state is stored server-side, matching the evaluator's expectations.

### Behavior probes handled
- **Vague query → clarify**: System prompt explicitly instructs the model to ask one focused question before recommending
- **Off-topic → refuse**: Explicit refusal instruction for non-assessment topics
- **Refinement**: Conversation history carries all prior context; model updates shortlist
- **Comparison**: Full catalog text enables grounded comparison answers
- **Turn cap**: System prompt reminds model to commit by turn 8

## Local Setup

```bash
git clone <repo>
cd shl-recommender
pip install -r requirements.txt
export ANTHROPIC_API_KEY=sk-ant-...
uvicorn app.main:app --reload
```

API docs available at `http://localhost:8000/docs`

## Running Tests

```bash
# Unit/schema tests (no API key needed for catalog integrity tests)
pytest tests/ -v

# Integration tests (require real ANTHROPIC_API_KEY)
ANTHROPIC_API_KEY=sk-ant-... pytest tests/ -v
```

## Deployment (Render)

1. Push this repo to GitHub
2. Create a new Web Service on [render.com](https://render.com)
3. Connect your repo; Render auto-detects `render.yaml`
4. Add `ANTHROPIC_API_KEY` as an environment variable
5. Deploy — health check at `/health`

## API Reference

### `POST /chat`

**Request:**
```json
{
  "messages": [
    {"role": "user", "content": "I am hiring a Java developer"},
    {"role": "assistant", "content": "What seniority level?"},
    {"role": "user", "content": "Mid-level, 4 years"}
  ]
}
```

**Response:**
```json
{
  "reply": "Here are 5 assessments for a mid-level Java developer.",
  "recommendations": [
    {"name": "Java 8 (New)", "url": "https://www.shl.com/...", "test_type": "K"},
    {"name": "OPQ32r", "url": "https://www.shl.com/...", "test_type": "P"}
  ],
  "end_of_conversation": false
}
```

### `GET /health`

```json
{"status": "ok"}
```

## Catalog

106 SHL Individual Test Solutions covering:
- **A** – Ability & Aptitude (Verify series, Graduate Reasoning)
- **B** – Biodata & Situational Judgement (SJT, Call Centre, Retail)
- **K** – Knowledge & Skills (Java, Python, SQL, AWS, …)
- **P** – Personality & Behavior (OPQ32r, OPQ32n, ADEPT-15, MQ)
- **S** – Simulations (Automata, Financial Modeling)
- **C** – Competencies (UCF)

## Evaluation Notes

- **Schema compliance**: Enforced by Pydantic models + catalog validation layer
- **Recall@10**: Full catalog in prompt enables high recall; ordering biased toward role-specific technical tests first
- **Behavior probes**: Covered by explicit system prompt rules and post-processing validation
