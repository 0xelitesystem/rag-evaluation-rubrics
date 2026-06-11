# rag-evaluation-rubrics

Practical evaluation rubrics for RAG (retrieval-augmented generation) systems. Concrete checks, not vague metrics.

Most RAG eval frameworks tell you to measure "faithfulness" or "relevance" without saying how. This repo defines what those mean in checkable terms, with example pass and fail cases, and points to which failure modes each check actually catches.

## Contents

| File | Covers |
|---|---|
| [retrieval-precision-recall.md](rubrics/retrieval-precision-recall.md) | How to build a labeled eval set and compute precision at k, recall at k, and MRR without a research budget |
| [generation-faithfulness.md](rubrics/generation-faithfulness.md) | Distinguishing hallucination from valid inference, with a 5-rule checklist |
| [citation-accuracy.md](rubrics/citation-accuracy.md) | Whether the citations actually support the cited claim, not just look plausible |
| [latency-budgets.md](rubrics/latency-budgets.md) | P50, P95, and tail budgets per pipeline stage, with realistic targets |
| [retrieval-failure-modes.md](rubrics/retrieval-failure-modes.md) | The 8 recurring ways retrieval breaks, how to spot each in your eval data |
| [generation-failure-modes.md](rubrics/generation-failure-modes.md) | Ungrounded inference, citation laundering, context overflow truncation, instruction collapse |
| [eval-set-construction.md](rubrics/eval-set-construction.md) | Building 50-200 labeled questions that actually catch regressions |
| [scoring-templates.md](rubrics/scoring-templates.md) | Copy-paste rubric prompts for LLM-as-judge with calibration notes |

## Who this is for

Engineers shipping RAG to production who already know the basics and need a defensible eval plan that catches regressions on real user traffic, not a paper benchmark.

If you have not built a RAG system yet, this will not teach you how. Start with a tutorial first, then come back.

## How to use this

1. Read [eval-set-construction.md](rubrics/eval-set-construction.md) and build your eval set. This is the hard part. Plan a half-day, minimum.
2. Pick 2 or 3 rubrics from the list above based on the failure modes you are most worried about.
3. Score your current system on those rubrics to get baselines.
4. Run rubric checks on every meaningful change to the prompt, retriever, or chunking strategy.
5. Re-score every 2 to 4 weeks with fresh real-traffic questions added to the eval set.

## What this is not

- Not a benchmark leaderboard. There is no comparison across vendors.
- Not a library. Nothing to install, nothing to import. Read it, apply it.
- Not exhaustive. Rubrics cover the common 80%. Domain-specific systems (legal, medical, financial) need additional checks beyond these.
- Not a substitute for human review. LLM-as-judge has known biases; the scoring templates include guardrails but you still need human spot-checks.

## Contributing

Add a new rubric only if it is:
- Concrete and verifiable (not "measure quality")
- Tied to a real failure mode you have hit in production
- Has at least one pass example and one fail example

See [CONTRIBUTING.md](CONTRIBUTING.md).

## License

MIT.

## Related

- [rag-chunk-visualizer](https://github.com/0xelitesystem/rag-chunk-visualizer) - visualize how chunking strategies affect retrieval inputs
- [embedding-cost-estimator](https://github.com/0xelitesystem/embedding-cost-estimator) - cost side of the retrieval pipeline
- [llm-eval-datasets-starter](https://github.com/0xelitesystem/llm-eval-datasets-starter) - public eval datasets to seed your own
- [agent-failure-modes](https://github.com/0xelitesystem/agent-failure-modes) - failure modes specific to agentic loops on top of RAG
- [prompt-cost-calculator](https://github.com/0xelitesystem/prompt-cost-calculator) - cost side of the generation stage
- [agentic-workflow-patterns](https://github.com/0xelitesystem/agentic-workflow-patterns) - for RAG agents specifically
