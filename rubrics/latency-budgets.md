# Latency budgets

RAG latency is the sum of several stages. A budget breaks the user-facing target into per-stage targets so you know which stage is over budget when things slow down.

## Stages

A typical RAG request:

1. **Query embedding** (10 to 100ms): the user question goes through an embedding model
2. **Vector search** (5 to 200ms): top k similar chunks fetched from the vector store
3. **Reranking** (50 to 400ms, optional): a cross-encoder or LLM rescores the top n
4. **Context assembly** (1 to 20ms): chunks formatted into the prompt
5. **Generation TTFT** (300 to 2000ms): time to first generation token
6. **Generation streaming** (10 to 50ms per token): rest of the answer

Total user-facing latency = stages 1 through 5 to first character on screen.

## Realistic budgets

### Interactive chat (perceived as fast)

| Stage | P50 | P95 |
|---|---|---|
| Query embedding | 30ms | 80ms |
| Vector search | 30ms | 150ms |
| Rerank (if used) | 100ms | 300ms |
| Context assembly | 5ms | 20ms |
| TTFT | 600ms | 1500ms |
| **Total to first token** | **800ms** | **2000ms** |

Anything above 2 seconds to first token feels sluggish in chat. Above 4 seconds, users start to wonder if it broke.

### Bulk processing or background

P95 of 10 to 30 seconds is acceptable. Optimize for throughput and cost, not latency.

### Voice or sub-second use cases

Drop reranking. Use smaller models. Pre-warm everything. Target P95 under 1.2 seconds to first audio token. Hard.

## How to measure

Instrument each stage with start/end timestamps. Log per-request:

```
{
  "request_id": "...",
  "stage_times_ms": {
    "embed": 28,
    "search": 45,
    "rerank": 0,
    "assemble": 3,
    "ttft": 720,
    "total_to_first_token": 796
  },
  "tokens_in": 1240,
  "tokens_out": 380
}
```

Aggregate to compute P50, P95, P99 per stage daily. Set alerts on P95 regression.

## What blows up latency in production

| Cause | Stage affected | Fix |
|---|---|---|
| Cold start of embedding model | Embedding | Keep warm with a heartbeat request |
| Vector store paged to disk | Search | Right-size memory; check index residency |
| Network hop to embedding API | Embedding | Self-host if you can afford it |
| Long context bloating TTFT | Generation | Smaller context, smaller chunks, fewer chunks |
| Reranker timeout | Rerank | Set a budget; fall back to non-reranked on timeout |
| Slow LLM at peak hours | Generation | Provisioned throughput or fallback model |
| Streaming buffer flushing | Generation | Check your proxy or load balancer buffering |

## Tail latency matters most

P50 is what most teams optimize. P95 is what users feel.

A system with 500ms P50 and 8 second P95 feels broken even though half the responses are fast. The 1-in-20 slow response is what users remember.

Always report P95 alongside P50. P99 if you have a high-stakes product or large traffic.

## When latency matters less than you think

For complex answers where the user expects a real answer (not a snippet), they will wait 3 to 5 seconds. Show a clear loading state. Stream the answer as it generates.

For factoid lookups ("what is X"), users expect 500ms. Cache them aggressively.

For agent flows with multiple steps, individual step latency matters less; total time-to-answer is what counts.

Tune budgets to the perceived task complexity, not a single global target.

## Cost vs latency tradeoffs

Most latency optimizations cost money:
- Faster vector store: more memory or more shards
- Better LLM endpoint: provisioned throughput is expensive
- Self-hosted embedding: infra cost
- Caching: storage cost
- Smaller models: usually lower quality

Track latency and cost together. A 2-second improvement that doubles per-request cost is not always worth it.

## What this rubric does not cover

- Connection setup, TLS, DNS (network-layer latency)
- Client-side rendering time
- Auth check latency (often slow in real systems, often missed)
- Streaming render delays in the client

Those are real but outside the scope of RAG-specific optimization. Measure them once to know the floor; otherwise focus on the stages above.
