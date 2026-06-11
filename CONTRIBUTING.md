# Contributing

## What makes a good rubric

Each rubric must:

1. Be specific and checkable, not aspirational ("measure quality" is rejected)
2. Tie to a real failure mode someone has hit in production
3. Include at least one pass example and one fail example
4. Cover the case where the rubric does not apply (negative space)

## Submission

1. Add or edit files in `rubrics/`
2. Update README.md if you added a new rubric
3. Open a PR

PR title format: `add: <rubric-name>` or `update: <rubric-name>: <what changed>`

## What gets rejected

- Generic eval advice not specific to RAG
- Rubrics requiring proprietary tooling
- Metrics without a documented failure mode they catch
- Long rewrites of widely-known frameworks (RAGAS, TruLens, etc.) without local insight

## What gets accepted

- New failure modes seen in production with a checkable detection method
- Domain-specific rubrics (legal RAG, medical RAG, code RAG) where the general rubrics fall short
- Calibration notes on existing judge prompts
- Corrections, clarifications, additional examples

## License

By contributing you agree your contribution is MIT-licensed.
