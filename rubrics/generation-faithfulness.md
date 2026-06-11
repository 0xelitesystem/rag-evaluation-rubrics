# Generation faithfulness

Faithfulness means: every factual claim in the answer is supported by the retrieved context. The model is not adding facts it could not have known from the chunks.

This is different from "correct". An answer can be faithful (every claim is in the chunks) and wrong (the chunks themselves were wrong). It can also be unfaithful (the model added facts) and accidentally correct.

You want both correct and faithful. Faithfulness is the one you can systematically check.

## The 5-rule checklist

For each claim in a generated answer, mark whether it is:

1. **Directly stated** in the retrieved chunks. The chunk literally contains this claim.
2. **Validly inferred** from the chunks. The claim follows from chunks by simple logical deduction (combining two clearly stated facts).
3. **Ungrounded inference** from the chunks. The claim sounds related but is not actually supported. Most common form: the model takes a single statement and extrapolates a generality.
4. **Hallucinated.** The claim has no relationship to the retrieved chunks.
5. **Common knowledge interjection.** The model added a fact from training data, not from the chunks. Sometimes correct, but the model should not be doing this for a RAG system; it bypasses your sources.

Faithful answers contain only types 1 and 2. Types 3, 4, and 5 are failure modes that need different fixes.

## Worked example

Retrieved chunks:
> The Pro plan costs $19 per month. It includes 1000 API calls per day. Overage is billed at $0.001 per call.

Question: "If I make 2000 calls in a day on the Pro plan, what does that cost?"

Generated answer: "On the Pro plan you get 1000 calls per day included. Calls beyond that are charged at $0.001 each. So 2000 calls means 1000 overage calls, which is $1.00 on top of the base $19 monthly fee. Note that calls are counted in UTC."

Claim-by-claim:

| Claim | Type | Note |
|---|---|---|
| 1000 calls per day included on Pro | 1 (stated) | Directly in chunk |
| Overage charged at $0.001 each | 1 (stated) | Directly in chunk |
| 2000 calls = 1000 overage = $1 | 2 (inferred) | Valid arithmetic |
| Base $19 monthly | 1 (stated) | Directly in chunk |
| Calls counted in UTC | 5 (interjection) | Not in chunks; might be true, might not |

The answer is mostly faithful but includes one common-knowledge interjection. In a customer-facing system, that interjection is dangerous: it might be wrong, and the user has no way to verify.

## How to score systematically

You cannot read every answer by hand. Use LLM-as-judge with a careful prompt.

For each (question, retrieved chunks, generated answer) triplet:
1. Extract every distinct factual claim in the answer (a list of atomic statements)
2. For each claim, ask the judge model: "Is this claim directly stated, validly inferred, or unsupported by the chunks?"
3. Score: percent of claims that are directly stated or validly inferred

A faithfulness score below 90% is concerning. Below 75% means the system is regularly making up facts.

See [scoring-templates.md](scoring-templates.md) for the LLM-judge prompts that work.

## Common false positives

The LLM judge will sometimes mark valid inference as ungrounded. The hard cases:

- **Arithmetic and unit conversion.** Some judges flag "1000 + 1000 = 2000" as ungrounded because the chunks did not literally state the result.
- **Restatement in different terms.** "Costs less than the Team plan" when the chunk says Pro is $19 and Team is $49.
- **Negation.** The chunk says "Pro plan has API access". The answer says "Pro plan does not require an enterprise contract for API access". This is correct but requires inferring what is not stated.

Calibrate your judge by running it on 20 hand-labeled examples first.

## Common false negatives

The judge missing real hallucinations:

- **Numerical hallucinations close to correct values.** The chunk says $19, the answer says $18.99. The judge often passes this; humans catch it.
- **Detail elaboration on stated facts.** Chunk says "supports JSON output". Answer says "supports JSON output with nested objects up to 10 levels deep". The first part is grounded; the elaboration is invented.
- **Plausible source attributions.** "According to our latest update..." when there is no source for "latest update" in the chunks.

A 5% human spot-check on judge outputs catches most of these.

## Fixing low faithfulness

1. Tighter system prompt: "Answer only using information from the provided context. If the answer is not in the context, say so."
2. Citation requirement: every claim must reference a chunk ID. This forces the model to ground each statement.
3. Smaller context: too many irrelevant chunks invite the model to ignore them and fall back on training data.
4. Bigger context: too few relevant chunks force the model to extrapolate. Find the right window for your task.
5. Different model: smaller models hallucinate more in RAG. If budget allows, test a tier up.

Faithfulness fixes usually live in the prompt, not the retrieval pipeline. Diagnose first; do not blame retrieval for a prompt problem.
