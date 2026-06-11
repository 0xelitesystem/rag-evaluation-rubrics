# Retrieval precision and recall

Most retrieval scoring guides assume you have a labeled corpus. You probably do not. Here is how to build one with 4 hours of work and get useful numbers out of it.

## The metrics in plain language

| Metric | What it measures | When you care |
|---|---|---|
| Precision at k | Of the top k chunks retrieved, what fraction are actually relevant | When you have a tight context budget |
| Recall at k | Of all relevant chunks in your corpus, what fraction made it into the top k | When missing information matters more than noise |
| MRR (mean reciprocal rank) | How high in the list the first relevant chunk appears | When the model mostly trusts the top result |
| NDCG at k | Ranking quality including partial relevance | When you have graded relevance (highly/somewhat/not relevant) |

For most production RAG, **precision at 5** and **recall at 10** is enough. Skip NDCG until you have graded labels and a real reason.

## Building labeled retrieval data without a research budget

You need: a list of questions, and for each question, which document chunks are actually relevant.

1. **Start with 30 to 50 questions from your eval set.** Skip the out-of-scope category for this exercise.

2. **For each question, retrieve the top 20 chunks** using your current system. Yes, this biases the label set toward what your retriever can already find. That is acceptable as a first pass. Round 2 fixes it.

3. **A human reads each of the 20 chunks and marks: relevant, partially relevant, not relevant.** "Relevant" means the chunk contains information needed to answer the question. "Partially" means it contains some but not all. This is the slow part. Budget 5 minutes per question, 4 hours for 50 questions.

4. **For round 2 (later), do keyword search and BM25 retrieval** in parallel with your embedding retrieval. Label new chunks from those that did not appear in round 1. This finds the chunks your current retriever is missing.

You now have a labeled set you can score against.

## Computing the numbers

For each question, given your top-k retrieved chunks and your labels:

- Precision at k = (relevant chunks in top k) / k
- Recall at k = (relevant chunks in top k) / (all relevant chunks for this question in the labeled set)
- Reciprocal rank = 1 / (rank of first relevant chunk in top k), or 0 if none

Average across questions to get precision-at-k, recall-at-k, and MRR for your system.

## Realistic targets

The right numbers depend on your domain and how much your generation model can recover from retrieval misses.

| Domain | Precision at 5 target | Recall at 10 target |
|---|---|---|
| Product docs, low ambiguity | 0.7 plus | 0.85 plus |
| Customer support, varied queries | 0.5 to 0.65 | 0.7 to 0.85 |
| Open-domain Q&A | 0.4 to 0.55 | 0.6 to 0.75 |
| Legal or medical, high precision needed | 0.75 plus | 0.9 plus |

If you are below the low end, your retrieval is the bottleneck. Above the high end, retrieval is not where to spend more effort; the generation prompt or eval methodology is more likely the issue.

## What this rubric does not catch

- **Chunk boundary issues.** A chunk might be labeled "relevant" because it contains the answer, but be cut off such that the model cannot use it. Check generation-faithfulness too.
- **Retrieved-but-misused.** The retriever returned the right chunk and the model ignored it. That is a generation problem, not retrieval. Different rubric.
- **Distribution shift.** Your labeled set was built from a snapshot of queries. New query patterns will not be reflected. Refresh quarterly.

## Common mistake

Reporting precision and recall together as one number ("F1 of 0.7") without saying what k was used. P@1 and P@10 are very different problems. Always include k.

## Sanity check

If your precision at 5 is 0.95 and recall at 10 is 0.95, you either have a tiny corpus, a leaked eval set (questions written from documents you also labeled), or a labeling error. Real production RAG rarely scores that high.
