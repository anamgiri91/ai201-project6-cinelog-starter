# PR Response Doc — CineLog Watchlist Feature

## AI Usage
**Comment 4:** After drafting an initial position (keep `public=True`, justified by low friction for social discovery), I asked Claude to act as a devil's advocate and raise counterarguments a careful reviewer might make. It surfaced three: (1) the benefit of a public default is back-loaded while the privacy cost is front-loaded, so the tradeoff barely applies to brand-new users; (2) my proposed mitigation (surfacing the visibility setting clearly) isn't actually implemented in this PR, so I shouldn't present it as already reducing risk; (3) claiming the privacy cost is "relatively low" is a judgment call that varies per user and isn't really the developer's to make. I agreed with all three and revised my response — I kept the same final position (`public=True`), but changed the justification from "the cost is low" to "the feature is fundamentally social, and that's the design priority, not a claim about how much any given user should care about privacy." I also reframed the visibility-setting idea as a follow-up recommendation rather than a mitigation already in place.

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
**What I did:**
**How I verified:**

## Comment 4 — Default visibility
**My position:**
I would keep `public=True` as the default.

**Reasoning:**
CineLog's watchlist feature exists to support social discovery. The goal is for friends and followers to find films through each other. If watchlists were private by default, that feature would be effectively invisible unless users manually opted in, which works against the reason the feature exists in the first place.

That said, I do not think the justification is completely straightforward, and there are two weaknesses worth acknowledging.

First, the benefit and the cost do not occur at the same time. The benefit of a public default comes later, once a user has built a watchlist that other people can actually discover. The privacy cost starts immediately with the first film they add. For a brand-new user, there may be very little social value yet, while the privacy risk is already present.

Second, I initially wanted to argue that the privacy cost is relatively low, but I do not think that is really my decision to make for every user. A watchlist entry might be nothing more than entertainment for one person, while for another it could reveal something they consider private, such as interests related to health, religion, politics, or sexual identity. That depends on the individual user, not the feature itself, and a developer cannot reliably predict how sensitive a particular watchlist may be. Because of that, my justification for `public=True` is not that the privacy risk is objectively small. Instead, it is that the watchlist is fundamentally a social feature and maximizing discovery is the design priority. Whether that priority outweighs the privacy cost is ultimately a product decision rather than a factual claim about how much any individual user should care about disclosure.

**Tradeoff acknowledged:**
The tradeoff I am accepting is that some users, especially new users and users whose viewing interests touch on topics they consider private, will have that information exposed by default before they have had a chance to notice the setting or decide whether they want their watchlist to be public. If this default remains, I would want the visibility setting to be clearly visible when a user creates or first uses a watchlist rather than buried in account settings. That is not something implemented in this PR, so I am mentioning it as a possible follow-up rather than treating it as a mitigation that already exists.

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