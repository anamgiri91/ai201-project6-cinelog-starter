# PR Response Doc — CineLog Watchlist Feature

## AI Usage
**Comment 4:** After drafting an initial position (keep `public=True`, justified by low friction for social discovery), I asked Claude to act as a devil's advocate and raise counterarguments a careful reviewer might make. It surfaced three: (1) the benefit of a public default is back-loaded while the privacy cost is front-loaded, so the tradeoff barely applies to brand-new users; (2) my proposed mitigation (surfacing the visibility setting clearly) isn't actually implemented in this PR, so I shouldn't present it as already reducing risk; (3) claiming the privacy cost is "relatively low" is a judgment call that varies per user and isn't really the developer's to make. I agreed with all three and revised my response — I kept the same final position (`public=True`), but changed the justification from "the cost is low" to "the feature is fundamentally social, and that's the design priority, not a claim about how much any given user should care about privacy." I also reframed the visibility-setting idea as a follow-up recommendation rather than a mitigation already in place.

**Comment 5:** After drafting an initial response agreeing with the maintainer's date-added preference, I asked Claude to raise counterarguments. It surfaced three: (1) my consistency argument (matching `get_collection()`'s sort order) conflated a collection's diary-like purpose with a watchlist's planning purpose, which are actually different; (2) my case for alphabetical ordering (helping locate a specific title) confused sorting with search — search is the better tool for that problem, not a different default sort; (3) I was implicitly treating my own reasoning as more evidence-based than the maintainer's, when really both are unproven design judgments. I agreed with all three and revised my response — I kept the same final position (newest-first) but demoted the consistency argument to a secondary point, replaced the alphabetical justification with the more honest "stable positioning" argument, and added an explicit acknowledgment that my conclusion is a judgment call, not something backed by usage data.

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
I would default watchlists to date-added order (newest first), which matches the maintainer's preference. That said, I want to explain my own reasoning rather than simply agreeing with it, and I also want to be honest about where my initial argument was weaker than I first thought.

**Reasoning:**
A watchlist functions more as a planning tool than an archive. It is a working list of things someone intends to watch rather than a record they browse from beginning to end. When I imagine opening a watchlist, the questions that come to mind are things like "What did I add recently?" or "What should I watch next?" Newest-first surfaces the entries that are most likely to reflect a user's current interests, which seems more aligned with that behavior than alphabetical ordering.

I initially also pointed to the fact that `get_collection()` already sorts films by `date_added` descending and argued that matching it would create consistency across the app. After thinking about it more, I do not think that is the strongest argument. A collection is a record of completed activity, where newest-first naturally supports a diary or viewing-history style experience. A watchlist serves a different purpose. The fact that one list uses a particular ordering does not automatically mean the other should. Consistency still has some value, but I see it as a secondary consideration rather than the main justification.

I also want to acknowledge that my reasoning is still a design judgment rather than an evidence-based conclusion. The maintainer's comment says that most users want to see what they added recently, and while I arrived at the same preference, I do not have usage data that proves it. My position is based on how I think people are likely to use a watchlist, not on measured user behavior. Because of that, I would not claim my argument is more rigorously supported than the maintainer's. I simply reached the same conclusion through a different line of reasoning tied to the purpose of the feature.

**Engagement with reviewer's point:**
The reviewer's point also made me reconsider my argument for alphabetical ordering. My first thought was that alphabetical order could help users find a specific title in a large watchlist, but I do not think that is actually a strong justification. If someone is trying to locate a particular film, search or filtering is a much better solution than relying on any default sort order. A stronger argument for alphabetical ordering is that it provides stable, predictable positioning that does not change every time a new film is added. That is a real advantage, but I do not think it outweighs the benefits of newest-first for a feature centered on future viewing plans and current interests.

For those reasons, I would implement the maintainer's preferred default of newest-first. At the same time, I would note that a future sort or filter option is probably the more complete answer to the "I can't find a specific title" problem than trying to solve it through the default ordering alone.

**Bug found while testing:**
While writing the sort-order test, I discovered `get_watchlist()` was already broken — it called `entry.film.to_dict()`, but `WatchlistEntry` had no relationship configured to `Film`, only `CollectionEntry` did (via the `backref="film"` on `Film.collection_entries`). This meant `get_watchlist()` would have thrown an `AttributeError` on any real call, but no existing test exercised it deeply enough to catch this before I added mine. I fixed it by adding a matching `watchlist_entries` relationship with `backref="film"` on the `Film` model.

## Comment 6 — Rebase
**What conflicted:**
**How I resolved it:**
**How I verified no conflict remains:**

## PR Description
<!-- Written at the end — feature overview, design decisions, manual testing steps -->