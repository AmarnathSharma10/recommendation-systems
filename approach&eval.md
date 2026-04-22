
## Approach

Four recommenders are built and compared:

1. **Popularity** — ignore user history; return globally most-read books. Floor baseline.
2. **Item-item collaborative filtering** — cosine similarity on the user-book matrix. Score each candidate book by summed similarity to the user's history.
3. **Content (genre)** — build a genre profile from the user's history and score candidates by cosine similarity to that profile.
4. **Hybrid** — normalized weighted blend of CF and content. Weight on CF grows with history length (sparse users lean on content, dense users lean on CF).

### Evaluation

Leave-one-out per user: randomly hold out one `(user, book)` edge from every user with ≥ 2 books, feed the rest to the model, check whether the held-out book lands in the top-K recommendations.

- **Hit@10** — fraction of held-out books recovered in the top-10 (primary).
- **MRR** — mean reciprocal rank, 0 if the target isn't in the top-10.

Results are also reported stratified by user history length (1, 2–3, 4–7, 8+ known books) to show where each model actually works.

### Key design choices

- **Book-level, not chapter-level.** With ~50k chapters across ~50k books, most books have one chapter in the data. Chapter granularity adds sparsity without adding signal.
- **No temporal assumptions.** See `problem_statement.md` — the data has no reliable ordering, so we don't pretend to predict "next."
- **No separate train/test user split.** The held-out edge serves as the test signal; the rest of the graph is training data. This is the Netflix Prize setup.
- **Deliberate simplicity.** The candidate set is the whole catalog (~50k books) and median user history is ~3 books. Heavy models (matrix factorization, neural, sequential) are not justified at this data density.

---

## Results

After running the script, two tables are printed:

### Overall

```
         model  Hit@10     MRR   n_users
    popularity   ...       ...    10000
  item_item_cf   ...       ...    10000
 content_genre   ...       ...    10000
        hybrid   ...       ...    10000
```

### Breakdown by history length

```
         model  history  Hit@10     n
    popularity       1     ...    ...
    popularity     2-3     ...    ...
    popularity     4-7     ...    ...
    popularity      8+     ...    ...
  item_item_cf       1     ...    ...
  ...
```

### What to look for

- **CF and content should beat popularity** on users with 4+ known books. On users with 1 known book, the gap is usually small — there's just not much signal to personalize with.
- **Hybrid should win on mid-length histories** (2–7 books), where CF alone is still noisy and content alone is too coarse.
- **Absolute numbers will look modest.** 50k-book catalog × median 3 books per user = inherently hard retrieval. A Hit@10 in the 5–15% range is a plausible, honest result. Anything dramatically higher probably indicates a bug or a leak.
- **The history-length breakdown is the honest picture.** A headline Hit@10 averaged over all users hides the fact that most users are sparse. The breakdown exposes where personalization actually helps.

---

## What I'd improve given more time

**Model side**

- **Implicit-feedback matrix factorization (ALS).** The user-book matrix is classic implicit-feedback territory and ALS often beats item-item CF on sparse graphs. Would add roughly 50 lines via the `implicit` library.
- **Author as a content feature.** Currently only genre is used in the content model. Author co-occurrence is often a stronger signal than genre — two books by the same author are more similar than two books in the same broad genre.
- **Popularity debiasing.** Item-item CF as implemented is biased toward popular items. Dividing raw co-occurrence by √(pop_i × pop_j) (or using BM25-style weighting) tends to improve long-tail recommendations measurably.

**Evaluation side**

- **Multiple held-out items per user**, not just one. Would reduce evaluation variance and let us report Recall@K alongside Hit@K.
- **Stratify by item popularity, not just user history length.** Does the hybrid win on long-tail books (where CF has little signal)? That's the breakdown that would really justify the hybrid over plain CF.
- **Coverage and diversity metrics.** A recommender that gets 12% Hit@10 by only ever recommending the top 50 popular books is worse than one that gets 10% across a diverse catalog. Worth reporting intra-list diversity and catalog coverage.

**Framing side**

- **Explicit cold-start handling.** Brand-new users (0 history) currently fall through to popularity by default. A production version would want a lightweight onboarding flow — show a genre-diverse popular slate, use the first click to bootstrap personalization.
- **If timestamps were added**, the "next-read" framing from the brief becomes answerable and the problem gets significantly more interesting — recency, reading pace, and session structure all become usable signals.

---

## Honest notes on scope

- No deployment, no API, no persistence — the brief explicitly says this isn't expected.
- AI was used for discussion and rubber-ducking the problem framing, per the brief's guidance to keep generated code minimal; the code itself is hand-written and intentionally small (~270 lines).
- Roughly a day of work. Kept the scope tight rather than shipping a half-finished fancier model.
