# Citation accuracy

A common bug: the model produces an answer with citations, the citations look right, the user trusts the system, but when you check, the cited chunk does not actually support the claim it is attached to.

This is sometimes called "citation laundering". It is worse than no citations at all, because the citations create false confidence.

## What to check

For each (claim, citation) pair in an answer:

1. **Does the cited chunk exist?** Some systems hallucinate chunk IDs entirely. This is rare with current models but check anyway.
2. **Does the cited chunk contain the claim?** The chunk should directly state, or unambiguously imply, what the claim says.
3. **Is the citation the best chunk for this claim?** Sometimes the chunk is technically related but a different retrieved chunk would have been a better citation. This is a softer failure but worth tracking.

## Scoring

For a sample of generated answers, compute:

- **Citation existence rate**: percent of cited chunk IDs that exist in the retrieved set
- **Citation support rate**: percent of cited chunks that actually contain the claim
- **Citation density**: average citations per claim

Targets:
- Existence rate: 100% (anything less is a system bug)
- Support rate: 90% plus for production
- Citation density: at least 0.5 (one citation per two claims) for high-stakes domains

## Common patterns of bad citations

### Wrong chunk, right topic

The model cites a chunk that discusses the right topic but a different aspect.

Example: question asks about Pro plan pricing. The cited chunk discusses the Pro plan features. The claim "Pro is $19" is correct but the cited chunk does not contain pricing.

This usually means the model picked the first chunk that mentioned the topic and cited it without checking.

### Generic chunk for specific claim

The model cites a high-level overview chunk for a specific factual claim.

Example: claim "the timeout is 30 seconds" cited to a chunk about "API limits in general" that does not contain the 30-second figure.

### Stale citations after editing

The model generates an answer, then revises a sentence. The citation does not update.

Example: original draft says "Pro costs $15" cited to chunk 7 (which actually says $19). Model corrects to $19 but keeps the chunk 7 citation. Now correct, still citing the right chunk, no problem. But the reverse can happen: the model corrects a claim away from what the cited chunk says.

### Aggregate citation for compound claim

A single citation attached to a paragraph that contains 4 distinct claims, only 2 of which are in the cited chunk.

Better practice: cite per claim, not per paragraph.

## How to test

Build a small set of question + retrieved-chunks + answer triplets where you have hand-labeled which chunks support which claims.

Then for each generated answer, score:
1. List every claim and its citations
2. For each, manually verify the cited chunk supports the claim
3. Count failures

50 question samples is enough to get a baseline. Re-run after any prompt change that touches citation behavior.

## Why automated scoring is hard here

LLM-as-judge tends to be too lenient. If a chunk discusses related material, the judge often accepts it as supporting evidence even when it does not contain the specific claim.

To reduce judge leniency:
- Phrase the judge prompt as "does the chunk contain enough information to verify this exact claim without other sources?"
- Calibrate against hand-labeled examples
- Run human spot-checks on a 10% sample

## Fixing citation accuracy

1. **Force citation in the system prompt**: "Every factual claim must be followed by a citation to a chunk ID. If you cannot find a supporting chunk, say so instead of inventing or guessing."

2. **Reduce chunk size**: smaller chunks make it easier for the model to find the specific supporting fragment.

3. **Use a citation-aware model**: some models (Anthropic's citation API, structured citation modes in others) handle this natively and outperform free-form citation prompting.

4. **Two-pass generation**: first pass writes the answer, second pass verifies and adjusts citations. Slower and more expensive but the support rate improves significantly.

5. **Drop low-confidence claims**: if the model cannot find a supporting chunk for a claim, the safer behavior is to omit the claim rather than cite something weakly related.

## What to never do

Do not display citations in your UI until you have measured the support rate. Citations create trust. If the support rate is 70%, you are misleading 30% of users every time they trust a citation. Better to ship without citations than to ship with broken ones.
