# Scoring templates

LLM-as-judge prompts that actually work for RAG eval. Use these as starting points, then calibrate.

## Why these need calibration

Judge models have biases. They prefer longer answers. They give higher scores when the answer matches their own writing style. They are lenient on ungrounded claims that sound plausible.

Before trusting any judge prompt, calibrate it: run on 20 hand-labeled examples, compare judge scores to your scores, adjust the prompt until they agree on 18 of 20.

## Faithfulness scoring template

```
You will check whether a generated answer is faithful to source chunks.

Faithful means: every factual claim in the answer is either directly stated in
the chunks, or follows from the chunks by simple logical inference (combining
two clearly stated facts, basic arithmetic, restatement in different words).

Unfaithful means: claims that go beyond what the chunks support, including:
- Adding details not in the chunks
- Extrapolating from specific cases to general claims
- Inserting information from background knowledge
- Plausible-sounding but unsupported claims

Source chunks:
[INSERT CHUNKS HERE]

Generated answer:
[INSERT ANSWER HERE]

Step 1: List every distinct factual claim in the answer. Aim for atomic
statements, not full sentences.

Step 2: For each claim, decide if it is:
(A) directly stated in the chunks
(B) validly inferred from the chunks
(C) unsupported by the chunks

Step 3: Return a JSON object:
{
  "claims": [
    {"text": "...", "verdict": "A" | "B" | "C", "supporting_chunk": "..."},
    ...
  ],
  "faithfulness_score": <fraction of claims with verdict A or B>
}

Be strict. When uncertain between B (valid inference) and C (unsupported),
choose C. The cost of a false positive (calling unfaithful answers faithful)
is higher than the cost of being too strict.
```

### Calibration notes

- Without "Be strict" instruction, judges run about 8% over-lenient
- The atomic-claim decomposition step matters; without it, judges score whole answers as faithful when only some parts are
- "Supporting chunk" field forces grounding; do not omit it

## Citation accuracy scoring template

```
You will check whether citations in a generated answer actually support the
cited claims.

The answer contains claims followed by citations like [chunk_3] or [doc_2].
Each citation points to a source chunk.

For each (claim, citation) pair:
- Read the cited chunk
- Decide if the chunk contains enough information to verify the exact claim
  WITHOUT needing other sources

Do not accept partial support. A chunk that mentions the topic but does not
contain the specific claim being cited is NOT support.

Source chunks (with IDs):
[INSERT CHUNKS WITH IDs HERE]

Generated answer (with inline citations):
[INSERT ANSWER HERE]

For each citation, return:
{
  "claim": "...",
  "cited_chunk_id": "...",
  "chunk_supports_claim": true | false,
  "reason": "brief explanation"
}

Then return the overall citation support rate (fraction of true).
```

### Calibration notes

- The phrase "WITHOUT needing other sources" is critical; without it judges accept very weak support
- Asking for a reason field reduces drift; judges that have to justify their answer are stricter
- Judges sometimes over-strictly reject citations that paraphrase rather than quote; balance with calibration

## Answer relevance scoring template

For checking that the answer actually addresses the question (not faithfulness, just whether it answers what was asked).

```
Question:
[INSERT QUESTION HERE]

Generated answer:
[INSERT ANSWER HERE]

Score 1-5:
5 = directly answers the question completely
4 = answers the question with some omission of detail
3 = partially answers; misses an important aspect
2 = tangentially related; does not really answer
1 = does not address the question

Return: {"score": <1-5>, "justification": "..."}
```

### Calibration notes

- 5-point scales are noisier than binary; if you need clean numbers, use binary (answers / does not answer)
- Justification field is essential for spot-checking judge errors
- Judges often score 4 when humans would score 3; calibrate per domain

## What to never put in a judge prompt

- "You are an expert evaluator" or similar role priming. Adds no signal, may bias toward generous scoring.
- "Be thorough" or "carefully consider". Wastes tokens, does not actually improve quality.
- The expected answer. Once the judge sees the expected answer, it scores against that rather than against the question. Useful for direct-match scoring, terrible for open-ended scoring.
- Few-shot examples without confirming they help. Test with and without on your calibration set.

## Model selection for judging

- For binary or numeric scoring: a smaller, cheaper model is usually fine if calibrated well
- For atomic-claim decomposition: a larger model is worth it; decomposition errors cascade
- For citation accuracy: a model with strong reading comprehension; smaller models miss subtleties

The judge does not need to be the same model as the generator. Often a different model is better because it has different biases.

## Human spot-check rate

LLM judges drift. Re-calibrate every 4 to 8 weeks by sampling 30 judge scores and checking by hand. If agreement drops below 85%, the judge prompt needs updating.

Track judge agreement over time. A drop is a leading indicator that something in your system (or in the model behind the judge) has changed.
