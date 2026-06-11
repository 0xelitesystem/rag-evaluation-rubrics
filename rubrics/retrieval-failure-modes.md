# Retrieval failure modes

The 8 ways retrieval breaks in production. Each one looks different in your eval data; the fixes are different too.

## 1. Vocabulary mismatch

The user's words and the document's words do not overlap. Embeddings should bridge this but often do not for domain-specific terms.

**Spot it**: the relevant chunk has 0 of the query's content words. The retriever scored it below chunks that share surface vocabulary but are off-topic.

**Examples**: user says "billing", docs say "invoicing". User says "cancel subscription", docs use "terminate plan".

**Fix**: hybrid search (BM25 plus embeddings). Or domain-tuned embeddings. Or build a synonym table for the worst offenders.

## 2. Chunk boundary mismatch

The answer spans a chunk boundary. Each chunk individually does not contain the full answer.

**Spot it**: top 2 chunks both score highly and each contains half the information. Or no chunk scores well because each contains only a fragment.

**Examples**: a definition spans two paragraphs; chunking split it. A table where the header is in one chunk and the row is in another.

**Fix**: overlap between chunks (50 to 100 tokens). Hierarchical chunking with parent retrieval. Different chunking strategy for tabular content.

## 3. Specificity mismatch

The user asks a specific question, the relevant chunks are general; or the user asks generally, the chunks are too specific.

**Spot it**: retrieved chunks are about the topic but at the wrong level of detail.

**Examples**: user asks "what is the rate limit on the free tier", retrieved chunks are about rate limiting in general but never name the free tier number.

**Fix**: multi-granularity indexing (summary chunks and detail chunks). Query expansion to add specificity. Better document structuring.

## 4. Acronym and abbreviation collapse

The user types an acronym; the documents spell it out (or vice versa). Embeddings sometimes handle this, often do not.

**Spot it**: query contains an uncommon acronym, no chunks score well, but a manual check shows the topic is documented.

**Examples**: user asks "MFA setup", docs say "multi-factor authentication". User asks "SSO with Okta", docs use "single sign-on".

**Fix**: pre-process queries to expand known acronyms. Index documents in both forms.

## 5. Stale or wrong content surfacing

The retriever returns the right chunk, but the chunk is out of date and reflects an old version of the product or policy.

**Spot it**: the cited chunk contains an answer, but the answer is wrong. Cross-reference with current docs to find the discrepancy.

**Examples**: changelog from 2 years ago describing a feature that has since changed. Old API version documentation indexed alongside current.

**Fix**: filter retrievals by document version metadata. Set up content freshness checks. Archive old docs out of the index.

## 6. Distractor dominance

A high-scoring chunk is on the same topic but irrelevant to this specific question. It crowds out the actually relevant chunk in the top k.

**Spot it**: the top 3 chunks are all from the same document section but only one answers the question. The right answer is at rank 5 or below.

**Examples**: pricing page has 5 chunks; all score highly for "what does X cost"; only one mentions the specific plan asked about.

**Fix**: rerank with a cross-encoder. Diversity-aware retrieval (MMR). Filter to top-1-per-document then re-expand.

## 7. Negation and qualifier loss

The user asks "what plans do not include X" or "when is Y not allowed". Embeddings tend to retrieve chunks that mention X or Y positively because negation does not affect the vector much.

**Spot it**: queries with negations score poorly. The top chunks all describe the positive case.

**Examples**: "which plans do not have rate limiting" returns chunks about rate limiting. "When is the export feature unavailable" returns chunks describing when export works.

**Fix**: query rewriting for negations before embedding. Or use an LLM to expand the query into the positive form ("which plans have unlimited usage").

## 8. Out-of-corpus questions

The user asks something the documents do not cover. The retriever returns the top-k anyway, with low scores. The generation model uses them and hallucinates.

**Spot it**: top-k scores all below a threshold. The generated answer is plausible but unfounded.

**Examples**: user asks "how do I integrate with Stripe" against a product that does not integrate with Stripe.

**Fix**: confidence threshold below which the system refuses or asks for clarification. This is the single most impactful fix for ungrounded generation in narrow-corpus systems.

## How to diagnose which mode is hurting you

Build a small set of failed queries (questions where the generated answer was wrong). For each:

1. What were the top 5 retrieved chunks?
2. Is the correct chunk in the corpus at all?
3. If yes, what was its rank? What scored above it?
4. Tag the failure with one of the 8 modes above.

After 30 to 50 failures, you have a distribution. Fix the top 1 or 2 modes first; the others get amplified by the same fixes or are not worth chasing yet.

## A note on hybrid search

Hybrid search (BM25 plus embeddings) fixes modes 1, 4, and parts of 7 essentially for free. If you are not running hybrid search and you have any of those failure modes, that is the first change to make.
