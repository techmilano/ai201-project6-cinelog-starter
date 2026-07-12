# PR Response Doc — CineLog Watchlist Feature

## AI Usage

I used an AI assistant to inspect the repository, apply each review comment as an
isolated change, run the test suite after every step, resolve the rebase onto the
UUID-migrated `main`, and perform a read-only final audit. I reviewed and verified
every diff and every commit before it was made.

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
I am keeping `public=True` as the model default, while allowing API callers to
override it explicitly when adding an entry.

**Reasoning:**
CineLog is a community film-tracking application rather than a private personal
database. A public default supports the product's social use case: users can
share films they plan to watch, discover common interests, and receive
recommendations without configuring every entry individually.

The default also keeps the basic add operation simple. A caller that sends only
a `film_id` receives the existing community-oriented behavior, while a caller
with a privacy requirement can send `public: false`.

**Tradeoff acknowledged:**
The reviewer's privacy concern is valid. A public default means that a user who
does not notice the setting may share viewing intentions unintentionally. A
private default would provide stronger privacy by design, but it would add
friction to CineLog's community-discovery workflow. Supporting an explicit
`public` request value reduces this tradeoff without silently changing the
feature's original social behavior.

## Comment 5 — Sort Order

**My position:**
I changed the default watchlist order from alphabetical title order to
`date_added` descending, so the newest additions appear first.

**Reasoning:**
A watchlist is an active queue of films a user plans to watch. The most common
immediate return workflow is locating a film that was just saved. Newest-first
keeps recent intent visible and matches the ordering already used by
`get_collection()`.

It also avoids making title order the only available organizational assumption.
Alphabetical order is useful for scanning a large stable library, but a
watchlist is generally more temporal than archival.

**Engagement with the reviewer's point:**
The alphabetical implementation was deterministic and made locating a known
title straightforward. I agree that this is beneficial for large lists.
However, for the default API response, I prioritize recent user activity and
consistency with the collection service. A future API could expose a `sort`
query parameter for callers that prefer title ordering.

## Comment 6 — Rebase and UUID Migration

**What conflicted:**
`models.py` conflicted because the watchlist branch was created before the main
branch migrated film IDs from auto-incrementing integers to UUID strings. The
feature branch added `WatchlistEntry` against the old integer-based `Film`
model, while `main` changed `Film.id` and `CollectionEntry.film_id` to
`db.String(36)` and no longer contained `WatchlistEntry` at all.

**How I resolved it:**
I retained the UUID-based `Film` and `CollectionEntry` definitions from
`main`, then reapplied the `WatchlistEntry` model with
`film_id=db.String(36)` and the matching foreign key, keeping the
`Film.watchlist_entries` relationship that `get_watchlist()` depends on. I also
updated the watchlist service docstring, the route request docstring, and the
nonexistent-film test value to use UUID strings. These UUID-resolution changes
are captured in a single commit, `fix: migrate watchlist feature to UUID film IDs`,
kept separate from the pre-rebase feature commits.

**How I verified no conflict remains:**
I searched for Git conflict markers and remaining integer film-ID assumptions,
ran the complete test suite, verified SQLAlchemy mapper configuration succeeds
(so the `WatchlistEntry.film` relationship is valid at runtime), and inspected
`git log --merges origin/main..HEAD`. The suite passed, no markers or stale
integer assumptions remain, and the merge-commit query returned no results.

## Stretch Features

### remove_from_watchlist()
Implemented as an optional stretch feature. `remove_from_watchlist()` mirrors
`remove_from_collection()`: it deletes the matching entry and raises
`NotInWatchlistError` when the film is not on the user's watchlist. It is exposed
via `DELETE /watchlist/<user_id>/remove` (returns 404 on `NotInWatchlistError`)
and covered by happy-path and not-in-watchlist tests. This was beyond the
required review comments and could equally have shipped as a follow-up PR.

### Additional edge-case test
Implemented. Beyond the required nonexistent-film test, `tests/test_watchlist.py`
also covers duplicate rejection (`test_add_to_watchlist_duplicate_raises`) and
newest-first ordering (`test_get_watchlist_returns_newest_first`).

### Explicit visibility parameter
Implemented. `add_to_watchlist()` accepts `public=True` by default, and the
`POST /watchlist/<user_id>/add` endpoint honors an optional `public` field in the
request body, allowing callers to create a private entry with `{"public": false}`.

## Final Verification

```
$ pytest tests/ -v
9 passed

$ python -m compileall app.py models.py services routes tests
(no syntax errors)

$ git diff --check origin/main...HEAD
(no whitespace errors)

$ git log --merges origin/main..HEAD
(empty — no merge commits)

$ git merge-base --is-ancestor origin/main HEAD && echo "rebased onto main"
rebased onto main

$ git status
On branch feature/watchlist
nothing to commit, working tree clean
```

## Commit History

![git log](git_log_features.png)

## PR Description

**Feature overview**
Adds a watchlist so a user can save films to watch later, separate from their
watched collection. It introduces the `WatchlistEntry` model and two service
functions, `add_to_watchlist()` and `get_watchlist()`, exposed through
`POST /watchlist/<user_id>/add` and `GET /watchlist/<user_id>`. Adding a film
that is already saved raises `AlreadyInWatchlistError`; adding an unknown film
raises `FilmNotFoundError`.

**Visibility decision**
`public` defaults to `True` (community-visible), consistent with CineLog's
social/discovery model. Callers may override this per entry by sending
`{"public": false}`. The privacy tradeoff of a public default is discussed in the
Comment 4 response.

**Sort-order decision**
`get_watchlist()` returns entries ordered by `date_added` descending (newest
first), matching `get_collection()`, because a watchlist is an active
"save for later" queue where the most recently added film is the one a user is
most likely looking for.

**How to manually test end to end**
1. Start the app: `python app.py` (serves http://127.0.0.1:5000).
2. Seed a user and two films and capture their UUIDs:
   ```
   python -c "from app import create_app, db; from models import User, Film; \
   app=create_app(); ctx=app.app_context(); ctx.push(); \
   u=User(username='alice', email='alice@example.com'); \
   f1=Film(title='Dune', year=2021); f2=Film(title='Arrival', year=2016); \
   db.session.add_all([u, f1, f2]); db.session.commit(); \
   print('USER', u.id); print('FILM', f1.id); print('SECOND_FILM', f2.id)"
   ```
3. Add the first film to the watchlist (public default):
   `curl -X POST http://127.0.0.1:5000/watchlist/<USER_ID>/add -H 'Content-Type: application/json' -d '{"film_id":"<FILM_ID>"}'` → expect 201.
4. Add a second, distinct film privately (deduplication blocks re-adding the same
   film, so a different `film_id` is required):

   ```
   curl -X POST \
     http://127.0.0.1:5000/watchlist/<USER_ID>/add \
     -H "Content-Type: application/json" \
     -d '{"film_id":"<SECOND_FILM_ID>","public":false}'
   ```

   Expect HTTP 201 with `"public": false`.
5. View the watchlist (newest first): `curl http://127.0.0.1:5000/watchlist/<USER_ID>`.
