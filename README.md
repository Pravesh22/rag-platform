# RAG Internal Platform (portfolio prototype)

A small, production-shaped RAG assistant for internal company policies. The point of the project is not the chat
answer. It is everything around it: **grounding and guardrails, async job processing, end-to-end tracing, a human
review loop, and a rigorous evaluation harness.**

```
                       X-Request-ID                     W3C trace context (OpenTelemetry)
 client ──POST /v1/query──▶ FastAPI ──enqueue(job_id, question, request_id)──▶ Redis ──▶ Celery worker
    ▲                        │  job: QUEUED                                                  │
    │                        ▼                                                               ▼
    └──GET /v1/jobs/{id}── Redis job store ◀── RUNNING / RETRYING / SUCCEEDED / FAILED ── RAG pipeline
                                                                                             │
        BM25 + dense ─▶ RRF fusion ─▶ rerank ─▶ CONFIDENCE GATE ─▶ Claude (grounded, cited) ─▶ citation check
                                                     │ below threshold                         ─▶ groundedness judge
                                                     ▼                                              │ fails
                                              refuse + review queue ◀───────────────────────────────┘
                                                     │
              PENDING_REVIEW ─▶ APPROVED | REJECTED | NEEDS_EDIT ─▶ export ─▶ golden-set candidates ─▶ eval harness
```

## Quick start (no API key needed)

```bash
pip install -r requirements-dev.txt
make test        # 36 tests
make calibrate   # picks the confidence threshold from the DEV split
make eval        # metrics on the held-out TEST split
make gate        # CI regression gate against eval/baseline.json
```

Without `ANTHROPIC_API_KEY` the app uses a deterministic **offline stub** in place of the LLM so everything runs
anywhere. The stub is for tests and CI only. Its numbers prove the harness works; they say nothing about answer
quality. See "Using real Claude" below.

### Run the full stack

```bash
docker compose up -d                       # Redis + Jaeger (UI: http://localhost:16686)
cp .env.example .env && set -a && . ./.env && set +a
make worker                                # terminal 1  (celery, solo pool)
make api                                   # terminal 2  (uvicorn on :8000)

curl -si -X POST localhost:8000/v1/query -H 'content-type: application/json' \
     -H 'X-Request-ID: demo-1' -d '{"question":"How many unused vacation days can I carry over?"}'
curl -s localhost:8000/v1/jobs/<job_id>
curl -s 'localhost:8000/v1/reviews?status=PENDING_REVIEW'
curl -s localhost:8000/metrics | grep rag_
```

Search `demo-1` in the logs, or the `trace_id` from the response header in Jaeger, to see the whole request across
the API, the queue and the worker.

### Using real Claude

```bash
export ANTHROPIC_API_KEY=...   # LLM_PROVIDER=auto picks it up. Model: ANTHROPIC_MODEL (default claude-sonnet-5-5)
make eval
```

Then re-run `make calibrate` and `python -m eval.run_eval --split test --update-baseline`, because the offline
baseline in `eval/baseline.json` was produced by the stub.

## What is where

| Area | Files | Notes |
|---|---|---|
| Chunking | `app/ingestion.py` | Heading-aware; markdown tables are never split |
| Hybrid retrieval | `app/retrieval.py` | BM25 + dense, fused with Reciprocal Rank Fusion, then reranked. Offline defaults need no downloads; `sbert` + `cross-encoder` are one env var away |
| Hallucination defenses | `app/guardrails.py`, `app/rag.py` | Grounding prompt, per-sentence citations, **confidence gate before the LLM call**, fabricated-citation detection with one strict retry, LLM-judge groundedness check, unparseable verdict = not trusted |
| Request correlation | `app/context.py`, `app/main.py` | ASGI middleware reuses a sanitised `X-Request-ID` or generates one, stores it in a `ContextVar`, echoes it back with `X-Trace-ID` |
| Tracing and logs | `app/telemetry.py`, `app/logging_setup.py`, `app/metrics.py` | OpenTelemetry (FastAPI, Celery, Redis auto-instrumentation + a span per RAG stage). Every JSON log line carries `request_id`, `trace_id`, `span_id` |
| Metrics | `app/metrics.py` | Prometheus: per-stage latency histograms (p50/p95/p99 via `histogram_quantile`), tokens, errors by type, outcomes, top rerank score, queue depth |
| Async jobs | `app/worker.py`, `app/jobs.py` | QUEUED, RUNNING, RETRYING, SUCCEEDED or FAILED. Bounded retries with exponential backoff only for transient LLM errors. Clean user-facing failure message; details only in logs |
| Human-in-the-loop | `app/reviews.py` | State machine with enforced transitions and reason codes, review and feedback APIs, rejection stats, export of approved items as golden-set candidates |
| Evaluation | `eval/` | 40-query golden set (22% unanswerable) with dev/test split, retrieval + generation + refusal metrics, slices, CI gate, threshold calibration, judge calibration |

### Golden-set labeling schema

```json
{"id": "q011", "split": "test", "query": "...", "answerable": true,
 "evidence": ["| manager | $2,000 | director |"],      // text that must appear in a relevant chunk
 "reference_answer": "...", "key_facts": ["$2,000", "director"],
 "query_type": "table", "category": "finance", "difficulty": "hard", "labeled_by": "author"}
```

Evidence is labeled as *text*, not chunk IDs, so labels survive a change of chunking strategy.

### Metrics

- **Retrieval:** Recall@5 (share of gold evidence found, so multi-hop is handled), MRR, NDCG@5
- **Generation:** key-fact coverage, groundedness (judge), answer relevance (judge), citation precision and recall
- **Refusal:** correct-refusal rate on unanswerable queries, false-refusal rate on answerable ones, hallucinated-answer rate
- **Ops:** latency p50/p95, tokens per query, review rate

## Interview talking points, mapped to the feedback

| Feedback | Where to point | One-liner |
|---|---|---|
| Hallucination mitigation | `guardrails.py`, `rag.py`, `tests/test_pipeline.py` | "Layered: grounded prompt, mandatory citations, a gate on the reranker score before generation, citation validation, then a judge. A failed check withholds the answer and queues it for review." |
| Request correlation | `main.py` middleware, `worker.py`, `tests/test_api.py` | "ID at the edge in a ContextVar, passed into the task, with OTel propagating trace context through Celery headers. One trace ID spans API and worker log lines; I verified it across real processes." |
| Evaluation rigor | `eval/` | "Stratified golden set with unanswerables, evidence-text labels, held-out split, per-slice reporting, a threshold calibrated on dev and checked on test, a calibrated judge, and a CI gate." |
| Human-in-the-loop model | `reviews.py` | "Explicit states, enforced transitions, reason codes, an API for decisions, and approved corrections exported back into the golden set." |

## Honest limitations

- **Offline stub, not Claude.** The real-Claude path (`_anthropic` in `app/llm.py`) was written against the SDK but has
  not been exercised end-to-end here, because there was no API key in the build environment. `LLM_TEMPERATURE` is
  optional and only sent if set.
- **Offline retrieval is lexical.** It falsely refuses "doctor's note" because the policy says "medical certificate".
  `make eval` lists this under "Misses". That is the case for switching to `sbert` + `cross-encoder` and re-calibrating.
- **Small data.** Six documents, 40 queries, 12 judge-calibration examples. Real use needs the 200–500 query set and
  ~50 human-labeled judge examples described in the notes above.
- **The stub judge has blind spots**, for example a number that appears elsewhere in the context. `eval.judge_calibration`
  shows this. It is the reason you calibrate a judge before trusting it.
- Not built: authentication, rate limiting, multi-tenant document ACLs, vector DB persistence (the index is in memory),
  and a UI. `docker-compose.yml` was not run in the build environment. Redis and the worker/API processes were.
# rag-platform
