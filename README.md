# Problem Statement

## Task

**Given a set of books a user has previously interacted with, recommend other books from the catalog that they are also likely to have interacted with.**

This is a **static graph completion** problem on the bipartite `(user, book)` interaction graph. We do **not** predict what the user will read next — the dataset has no timestamps, and row order is not reliable across books. Instead, we treat each user's interactions as an unordered set and ask: given part of that set, can we recover the rest?

## Why this framing

The brief asks for "next chapter" or "next book" prediction, but the provided data does not support that:

- No timestamp column in either CSV.
- Row order in `interactions.csv` has no guarantee of being chronological across different books.
- `chapter_sequence_no` only orders chapters *within* a single book — not across books, and not across users.

Without temporal signal, "next" has no definition. Rather than fabricate one, we scope the task down to what the data actually supports: predicting held-out edges in the user-book graph. This is a legitimate, well-defined recommendation problem — it is the same shape as the classic Netflix Prize setup — and the output is directly useful ("users who read these books also read…") even without a notion of order.

## Formal definition

Let **U** be the set of users and **B** be the set of books. Each user *u ∈ U* has an interaction set **H_u ⊆ B** — the books they've engaged with.

For evaluation, we randomly split each user's set into:

- **known history** — given to the model as input
- **held-out target** — one book kept hidden

The task: given the known history, produce a ranked list of top-K candidate books from **B**. The recommendation is correct if the held-out target appears in the top-K.

## Example

**User `user_2378720`** has interacted with 5 books: `{444295, 785684, 999595, 748410, 418083}`.

We randomly hold out book `999595`, and give the model the remaining 4 books as input:

```
Input history:    [444295, 785684, 748410, 418083]
Held-out target:  999595
```

The model returns its top-10 recommendations based on the input history:

```
Recommendations:  [999595, 302711, 881044, 412900, 550621, ...]
```

The target `999595` appears at rank 1, so:

- **Hit@10 = 1** (target is in top-10)
- **Reciprocal Rank = 1 / 1 = 1.0**

If instead the recommendations had been `[302711, 881044, 999595, 412900, ...]`, the target would be at rank 3:

- **Hit@10 = 1**
- **Reciprocal Rank = 1 / 3 ≈ 0.33**

If the target didn't appear at all in the top-10:

- **Hit@10 = 0**
- **Reciprocal Rank = 0**

## Evaluation metrics

Averaged over all evaluable users (those with at least 2 books):

- **Hit@10** — fraction of users whose held-out book appears in top-10. Primary metric.
- **MRR (Mean Reciprocal Rank)** — average of 1/rank (0 if missing). Smoother than Hit, captures ranking quality.

## Assumptions made explicit

- **No temporal ordering is assumed or used.** Each user's interactions are treated as an unordered set.
- **Chapter granularity is collapsed to books.** Most books in the data have only one chapter represented, so modeling at chapter level adds no signal while adding sparsity. Book-level is the working unit.
- **Repeated interactions are deduplicated.** Multiple rows of `(user, chapter)` for the same `(user, book)` are collapsed to a single edge.
- **Only users with ≥ 2 books are evaluable.** A user with 1 book has nothing to split into history + target. Users with 1 book still contribute to the training signal (other users' co-occurrences with their one book) but don't appear in the evaluation numerator.

## What this system is *not* doing

To avoid ambiguity:

- It is **not predicting future interactions.** It has no concept of future.
- It is **not predicting the next chapter within a book.** It doesn't model chapter-level progression.
- It is **not solving cold start for brand-new users.** Users with zero interactions are outside the evaluation; the popularity baseline is the implicit fallback for them.
- It is **not personalizing by time-of-day, recency, or session.** None of those signals exist in the data.

## The question the system answers in plain English

> *"Based on the books this user has read, which other books from the catalog would you guess they've also read?"*

That's it. A user looking at a book in the catalog can ask "what else would readers like me enjoy?" and the system returns a ranked list based on (a) what other users with overlapping histories have read and (b) which books match the genre profile of what they've read. That's a useful recommender even without any temporal dimension.

> ## Refer approach&eval.md for details about the pipeline along with the jupyter notebook
