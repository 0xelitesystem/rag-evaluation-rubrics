# Generation failure modes

Retrieval can be perfect and generation can still fail. Here are the 6 recurring patterns.

## 1. Ungrounded inference

The model takes a single statement and extrapolates a generality, or combines two facts into a conclusion that does not follow.

**Looks like**: a confident answer where you cannot point to a specific chunk that says the claim.

**Example**: chunks say "Pro plan has 1000 API calls daily" and "Team plan has 5000 API calls daily". Model answers "the Enterprise plan likely has 10000 or more" with no chunk supporting that.

**Fix**: explicit prompt instruction. "Do not make claims about anything not present in the context. If the user asks about something not in the context, say so."

## 2. Citation laundering

The model produces citations that look authoritative but do not actually support the cited claim.

Covered in [citation-accuracy.md](citation-accuracy.md). The pattern is severe enough to track separately.

## 3. Context overflow truncation

The retrieved context exceeds the model's context window. The model silently drops chunks or truncates the prompt. The answer is based on whatever survived.

**Looks like**: high-quality retrieval (the right chunks are there in the retrieval log) but the answer misses information from chunks that came after the cutoff.

**Fix**: enforce a context budget below the model's limit. Token-count chunks before assembly. Drop lowest-ranked chunks when over budget rather than truncating mid-chunk.

## 4. Instruction collapse

A long retrieved context overwhelms the system prompt instructions. The model stops following its guidelines around halfway through and reverts to default behavior.

**Looks like**: the first half of the answer follows instructions (cites correctly, refuses out-of-scope, sticks to context). The second half drifts (no citations, includes unsupported facts, ignores the system prompt's formatting requirements).

**Fix**: shorter context. Repeat critical instructions at the end of the prompt (right before the model generates). Use models with better instruction adherence on long contexts.

## 5. Confidence mismatch

The model phrases uncertain claims as confident facts (or, less commonly, hedges on facts that are well-supported).

**Looks like**: "The integration definitely supports X" when chunks say "support for X is planned". Or "X might be possible" when chunks clearly state X is supported.

**Fix**: prompt the model to mirror the certainty language of the source. "If the context uses hedged language ('may', 'planned', 'beta'), preserve that hedge. Do not present uncertain claims as definite."

## 6. Multi-hop reasoning failure

The question requires combining information from 2 or 3 chunks. The model uses one chunk and ignores the others, or fails to make the connection.

**Looks like**: answer is partially correct but misses the part that required cross-referencing chunks.

**Example**: question "if I upgrade mid-cycle, what happens to my unused credits". One chunk explains upgrade prorating, another explains credit policy. Model answers about prorating only.

**Fix**: chain-of-thought prompting before the answer. Or two-stage: first pass identifies all relevant chunks and what they say; second pass synthesizes. Slower and more expensive but the success rate improves.

## How to spot which mode is hurting you

When generation faithfulness is below your target, do not just retrain prompts blindly. Sample 30 failed answers and tag each with one of the 6 modes above. The distribution tells you where to focus.

A common pattern in production:
- 40% ungrounded inference (fix: tighter prompt)
- 20% context overflow (fix: chunk count or size)
- 15% instruction collapse (fix: prompt structure)
- 15% confidence mismatch (fix: prompt instruction)
- 10% multi-hop failure (fix: chain-of-thought or two-stage)

If your distribution looks different, the problem is somewhere else. Different model? Different domain? Eval set bias?

## A note on model selection

Smaller models hallucinate more in RAG. They are particularly bad at modes 1, 4, and 6.

If the system needs to run cheap, expect lower faithfulness and build accordingly: stricter refusal thresholds, fewer chunks (less context to drift in), shorter answers.

Larger models cost more but get out of these failure modes more reliably. The economics depend on the cost of a wrong answer in your domain.

## A note on chain-of-thought

CoT before answering helps with modes 1 and 6 (the reasoning modes). It does not help with modes 3, 4, 5 (the prompt-following modes) and can even make them worse by lengthening the prompt.

Use CoT selectively. Test with and without; do not adopt it as a default.
