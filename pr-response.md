# PR Response Doc — CineLog Watchlist Feature

## AI Usage

To be completed after implementation.

## Comment 1 — Rename

**What I did:**  
I renamed `save_to_watchlist()` to `add_to_watchlist()` in
`services/watchlist_service.py`. I also updated the import and function call in
`routes/watchlist/watchlist.py`.

**Why:**  
The existing collection service uses the project's `verb_to_noun` convention,
including `add_to_collection()`, `remove_from_collection()`, and
`get_collection()`. Using `add_to_watchlist()` keeps the new service consistent
with the established public API.

**How I verified:**  
I ran `git grep -n "save_to_watchlist"` and confirmed that the old name no
longer appears. I then ran `pytest tests/ -v` and confirmed that the existing
suite passed.

## Comment 2 — Deduplication

**What I did:**  
I added `AlreadyInWatchlistError` and queried `WatchlistEntry` for an existing
row with the same `user_id` and `film_id` before creating a new entry.

**Why:**  
Without the service-layer check, repeated calls could create multiple rows for
the same user and film. I followed the pattern already used by
`add_to_collection()` so collection and watchlist operations behave
consistently.

**How I verified:**  
I reviewed the resulting query against the `add_to_collection()` implementation,
ran the full test suite, and verified that attempting to add the same film twice
raises `AlreadyInWatchlistError`.

## Comment 3 — Missing Test

**What I did:**  
I created `tests/test_watchlist.py` and added a test confirming that
`add_to_watchlist()` raises `FilmNotFoundError` when the supplied film ID does
not exist.

**Why:**  
The service promises to raise `FilmNotFoundError`, but that behavior was not
covered by a watchlist test. The test protects the service contract and mirrors
the existing collection test for the same condition.

**How I verified:**  
I modeled the fixture setup and `pytest.raises()` assertion after
`test_add_to_collection_nonexistent_film_raises`. I ran
`pytest tests/test_watchlist.py -v` followed by `pytest tests/ -v`.

## Comment 4 — Default Visibility

**My position:**

**Reasoning:**

**Tradeoff acknowledged:**

## Comment 5 — Sort Order

**My position:**

**Reasoning:**

**Engagement with the reviewer's point:**

## Comment 6 — Rebase and UUID Migration

**What conflicted:**

**How I resolved it:**

**How I verified no conflict remains:**

## Stretch Features

### remove_from_watchlist()

### Additional edge-case test

### Explicit visibility parameter

## Final Verification

## Commit History

<!-- Add screenshot here -->

## PR Description