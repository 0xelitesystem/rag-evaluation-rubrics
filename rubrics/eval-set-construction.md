# Building a RAG eval set

The eval set is where most teams go wrong. They use 10 cherry-picked questions, get 100% accuracy, ship, and then watch real users break it.

A useful eval set is 50 to 200 questions covering the distribution of real queries, including the ugly ones.

## Target size

| System maturity | Eval set size |
|---|---|
| Just built, no real users yet | 30 to 50 hand-written questions |
| Some real traffic (under 100 users) | 50 to 100, half from real logs |
| Production with steady traffic | 100 to 200, 70% from real logs |
| Mission-critical (medical, legal, finance) | 200 plus, regular human review |

Bigger is not always better. Past 200 questions, you spend more time maintaining the set than getting signal from it. Quality over quantity.

## Composition

A useful eval set mixes 5 categories. Skip any and you have a blind spot.

### 1. Easy correct (20 to 30%)

Questions a working system should always get right. Single-fact lookups. Direct paraphrases of source content. These are your regression smoke test. If these break, something is badly wrong.

Example: "What is the maximum file upload size?" with the answer literally in your docs.

### 2. Hard correct (30 to 40%)

Questions that need multi-document synthesis, careful reading, or domain knowledge. These are where the value lives.

Example: "If I upgrade from Pro to Team mid-cycle, what happens to my unused API credits?" requiring two pricing docs and the credit policy.

### 3. Out-of-scope (15 to 20%)

Questions the system should refuse or redirect. Critical for catching ungrounded generation.

Example: "What is the weather in Tokyo?" against a product docs system. The correct behavior is "I do not have information about that."

### 4. Adversarial (10 to 15%)

Real user-style questions that are easy to get wrong:
- Misspellings of product names
- Wrong assumptions ("How do I export to XML?" when only JSON is supported)
- Compound questions
- Questions with leading wrong premises

### 5. Real ugly queries (10 to 20%)

Pulled directly from your logs. Typos, fragments, multi-language, weird capitalization. If you do not have logs yet, write 5 to 10 that mimic how non-technical users actually type.

## Labeling

Each question needs:
1. The question text
2. The expected behavior category (answer, refuse, clarify)
3. If "answer": the key facts the answer must contain, written as a list of must-include points
4. If "answer": which source documents should be retrieved (for retrieval scoring)
5. Optional: known-wrong answers to watch for

Avoid writing the expected answer as full prose. Writing "the answer must include X, Y, Z" than checking exact strings, which is more tolerant of phrasing variation.

## How to find real ugly queries

Search your logs for:
- Queries under 4 words
- Queries with no retrieved chunks
- Queries with low confidence scores
- Queries where the user followed up with frustration signals (rephrasing 3+ times, "no that is wrong", thumbs down)

Pull 30 of each, dedupe, sample 20. That is your real-ugly seed set.

## Versioning

Eval set v1 should never be deleted. When you add questions, create v2 and keep v1 as the baseline comparison.

Track which version of your system was scored against which version of the eval set. Otherwise you cannot tell if a regression came from a system change or an eval set change.

Suggested structure:

```
evals/
  v1/
    questions.jsonl
    labeled.jsonl
  v2/
    questions.jsonl
    labeled.jsonl
  scores/
    2026-04-15-v2-system-baseline.json
    2026-04-22-v2-prompt-change.json
```

## What gets cut

- Duplicates and near-duplicates
- Questions where the labeler could not agree on the expected answer (those need clarification first)
- Questions where the source documentation has changed since labeling (re-label or remove)
- Questions where "the answer is in the docs" but only a senior engineer could find it (those are eval set bugs)

## Refresh cadence

Add 10 to 20 new questions every 2 to 4 weeks from real traffic. Retire questions that the system has gotten right 5 times in a row across 5 different prompt versions; they have become noise.

The eval set should evolve with the system. A frozen eval set goes stale within 3 months in fast-moving products.
