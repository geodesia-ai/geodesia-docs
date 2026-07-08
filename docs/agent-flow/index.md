# Agent Flow API

The Agent Flow API provides a structured multi-agent debugging and demonstration interface. It exposes pre-built pipeline types that show how Geodesia G-1 integrates into agentic AI workflows — and allows you to trace, replay, and compute explainability for each step.

This API is primarily intended for:

- **Integration demonstrations** — showing how Geodesia works in a multi-agent pipeline
- **Developer debugging** — tracing what happened in a complex multi-step pipeline
- **Research and evaluation** — computing PSS attribution across a full agent run

---

## Endpoints

All agent flow endpoints are prefixed with `/v1/agent-flow/`.

---

### GET /v1/agent-flow/demos

Returns the list of available demo pipeline types.

```bash
curl http://localhost:8199/v1/agent-flow/demos
```

```json
{
  "demos": [
    {
      "type": "rag_qa",
      "name": "RAG Question Answering",
      "description": "A multi-step pipeline that retrieves context from a knowledge base and answers a question with hallucination detection.",
      "steps": ["retrieve", "generate", "evaluate", "explain"]
    },
    {
      "type": "safety_filter",
      "name": "Safety Filter Pipeline",
      "description": "Demonstrates prompt screening, generation, and answer safety evaluation.",
      "steps": ["screen_prompt", "generate", "screen_answer"]
    },
    {
      "type": "closed_book_qa",
      "name": "Closed-Book QA",
      "description": "A single-step pipeline for evaluating factual claims without grounding context.",
      "steps": ["generate", "evaluate_closedbook"]
    }
  ]
}
```

---

### GET /v1/agent-flow/trace/{type}

Returns the full trace structure for a pipeline type — what steps it runs, what each step produces, and what Geodesia G-1 evaluates at each point.

```bash
curl http://localhost:8199/v1/agent-flow/trace/rag_qa
```

This returns a static schema describing the pipeline — useful for building visualizations or planning integration.

---

### POST /v1/agent-flow/run/{type}

Execute a complete agent pipeline of the given type and return the full trace with detection scores at each step.

```bash
curl -X POST http://localhost:8199/v1/agent-flow/run/rag_qa \
  -H "Content-Type: application/json" \
  -d '{
    "prompt": "What is the capital of France?",
    "collection_id": "my-knowledge-base",
    "session_id": "demo-session-001"
  }'
```

#### Request Fields

| Field | Type | Description |
|---|---|---|
| `prompt` | `string` | The user's question or instruction |
| `collection_id` | `string` | RAG collection to retrieve from (if the pipeline includes retrieval) |
| `session_id` | `string` | Optional session identifier for audit linking |
| `generation_config` | `object` | Generation parameters (see [Evaluate Endpoint](../product-api/evaluate.md)) |
| `explain` | `boolean` | Whether to include XAI attribution in the trace |
| `threshold_overrides` | `object` | Per-axis threshold overrides |

#### Response

```json
{
  "run_id": "run_abc123",
  "pipeline_type": "rag_qa",
  "session_id": "demo-session-001",
  "steps": [
    {
      "step": "retrieve",
      "input": {"query": "capital of France"},
      "output": {"chunks": [{"text": "Paris is the capital of France.", "score": 0.97}]},
      "duration_ms": 42
    },
    {
      "step": "generate",
      "input": {"prompt": "...", "context": "Paris is the capital of France."},
      "output": {"response": "The capital of France is Paris."},
      "duration_ms": 318
    },
    {
      "step": "evaluate",
      "input": {"response": "The capital of France is Paris.", "context": "..."},
      "output": {
        "halluc_context": {"score": 0.04, "triggered": false},
        "answer_safety": {"score": 0.02, "triggered": false}
      },
      "duration_ms": 89
    }
  ],
  "final_response": "The capital of France is Paris.",
  "blocked": false,
  "total_duration_ms": 449
}
```

---

### POST /v1/agent-flow/compute_pss/{type}

Run a PSS (Positional Semantic Stability) attribution pass over a complete agent pipeline. PSS is computed per pipeline step, allowing you to trace which parts of the input affected which parts of each step's output.

```bash
curl -X POST http://localhost:8199/v1/agent-flow/compute_pss/rag_qa \
  -H "Content-Type: application/json" \
  -d '{
    "prompt": "What is the capital of France?",
    "collection_id": "my-knowledge-base",
    "pss_n_samples": 8,
    "pss_temperature": 0.7
  }'
```

The response includes a `pss_attribution` block for each pipeline step, showing which input tokens most influenced the step's output.

---

## Use in Integration Testing

The agent flow API is particularly useful for validating that Geodesia G-1 is correctly wired into a multi-step pipeline:

```python
import httpx

# Run a full pipeline and check that hallucination detection is active
response = httpx.post(
    "http://localhost:8199/v1/agent-flow/run/rag_qa",
    json={
        "prompt": "What is the boiling point of water?",
        "collection_id": "science-kb"
    }
)
trace = response.json()

# Find the evaluate step
eval_step = next(s for s in trace["steps"] if s["step"] == "evaluate")
assert "halluc_context" in eval_step["output"], "Hallucination axis not running"
assert not trace["blocked"], "Correct answer was blocked"
print("Integration test passed.")
```
