# PR Response Doc — CineLog Watchlist Feature

## AI Usage

## Comment 1 — Rename
**What I did:**
Renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py`. Then I updated the one place that called it — `routes/watchlist/watchlist.py` — both the import line and the actual function call inside `add_film()`.

**How I verified:**
I ran `grep -rn "save_to_watchlist" .` across the whole repo to make sure I didn't miss any call sites. The only hit was a stale `.pyc` file in `__pycache__`, which is just old compiled bytecode from before the rename — not an actual source reference, so that was expected. After the rename I ran `pytest tests/ -v` and all 4 existing tests still passed, so nothing broke.

## Comment 2 — Deduplication
**What I did:**
Added a new `AlreadyInWatchlistError` exception to `watchlist_service.py`, following the same naming pattern as `AlreadyInCollectionError` in the collection service. Inside `add_to_watchlist()`, I added a check right after the film-existence check: query `WatchlistEntry` by `user_id` and `film_id`, and if something already exists, raise `AlreadyInWatchlistError` instead of silently creating a duplicate. I copied this pattern directly from `add_to_collection()`, just swapped `CollectionEntry` for `WatchlistEntry`.

I also noticed the watchlist route (`routes/watchlist/watchlist.py`) had no error handling at all on `add_film()` — so if I only fixed the service layer, a duplicate watchlist entry would've caused an unhandled exception and a 500 error instead of a clean response. I updated the route to catch both `FilmNotFoundError` and `AlreadyInWatchlistError`, returning 404 and 409 respectively, matching exactly how the collection route handles its equivalent errors.

**How I verified:**
I ran the full test suite and all 4 existing tests still passed. That said, I want to be upfront that this only proves I didn't break anything that already existed — none of the current tests actually hit the new dedup logic, since there's no watchlist test yet. I don't have real proof the duplicate check works until I write a test for it, which is Comment 3.

One more thing worth flagging: `WatchlistEntry` doesn't have a database-level `UniqueConstraint` the way `CollectionEntry` does (`unique_user_film_collection`). So right now my dedup check is the *only* thing preventing duplicate watchlist rows — it's happening at the application level, not enforced by the database. I didn't touch the model for this comment since that felt out of scope, but it's a gap worth knowing about.

## Comment 3 — Missing test
What I did:
Created tests/test_watchlist.py and wrote two tests, modeled directly on the patterns in test_collection.py. First, test_add_to_watchlist_nonexistent_film_raises, which mirrors test_add_to_collection_nonexistent_film_raises exactly — same fake UUID, same pytest.raises structure, just calling add_to_watchlist() instead. Second, since I'd already built the dedup logic in Comment 2 but had no test covering it, I also wrote test_add_to_watchlist_duplicate_raises, modeled on test_add_to_collection_duplicate_raises — it adds a film once, confirms a second add raises AlreadyInWatchlistError, then queries WatchlistEntry directly to confirm only one row actually exists in the database. I reused the same app, sample_user, and sample_film fixture pattern from test_collection.py rather than writing new ones.
How I verified:
Ran pytest tests/test_watchlist.py -v and both tests passed. Then ran the full suite with pytest tests/ -v to confirm all 6 tests (4 original + 2 new) pass together with no regressions.

## Comment 4 — Default visibility
**My position:**
**Reasoning:**
**Tradeoff acknowledged:**

## Comment 5 — Sort order
**My position:**
**Reasoning:**
**Engagement with reviewer's point:**

## Comment 6 — Rebase
**What conflicted:**
**How I resolved it:**
**How I verified no conflict remains:**

## PR Description
<!-- Written at the end — feature overview, design decisions, manual testing steps -->