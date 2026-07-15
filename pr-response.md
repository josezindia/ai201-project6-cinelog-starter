# PR Response Doc — CineLog Watchlist Feature

## AI Usage
<!-- Fill in at the end — how you used AI tools during this project -->

### Comment 1 — Rename
**What I did:** Renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py` to match the project's verb_to_noun convention already used by `add_to_collection()`. Updated the one call site in `routes/watchlist.py` — both the import statement and the function call inside `add_film()`.
**How I verified:** Ran `grep -rn "save_to_watchlist" .` across the repo after the rename to confirm zero remaining references. Also ran `pytest tests/ -v` to confirm the existing 4 tests still pass with no import errors.

## Comment 2 — Deduplication
**What I did:** Added an `AlreadyInWatchlistError` exception class and a duplicate check in `add_to_watchlist()`, following the exact pattern used by `add_to_collection()` in `collection_service.py` — a `filter_by(user_id=..., film_id=...).first()` query runs after the film-existence check and before the entry is created. If an existing entry is found, `AlreadyInWatchlistError` is raised instead of creating a duplicate.
**How I verified:** Confirmed the file imports cleanly (`python -c "import services.watchlist_service"`) and the existing test suite still passes with no regressions. The dedup logic itself will be directly exercised by the new test written for Comment 3.


## Comment 3 — Missing test
**What I did:** Created `tests/test_watchlist.py` with two tests: `test_add_to_watchlist_nonexistent_film_raises`, mirroring `test_add_to_collection_nonexistent_film_raises`'s fixture and assertion structure exactly, and `test_add_to_watchlist_duplicate_raises`, which verifies the Comment 2 dedup fix by confirming `AlreadyInWatchlistError` is raised on a duplicate add and that only one entry persists.
**How I verified:** Ran `pytest tests/test_watchlist.py -v` (both pass) and the full suite `pytest tests/ -v` (all 6 pass, no regressions to the existing collection tests).

## Comment 4 — Default visibility
**My position:** Changed the default from `public=True` to `public=False`. New watchlist entries are private unless a user explicitly makes them public.

**Reasoning:** A watchlist is a "want to watch" list, which can include picks a person wouldn't want a stranger to see by default — an unusual genre, something out of character, or just a personal taste they're not ready to share. Defaulting to private means the exposure only happens if the user actively chooses it, not as a side effect of just using the feature normally. This also matches the convention used by comparable apps (Letterboxd, Goodreads) where "want to X" lists default private and public sharing is opt-in.

**Tradeoff acknowledged:** Defaulting to private adds friction for the social/discovery use case — users who *do* want friends to see their watchlist now have to take an extra step to make it public, and a private-by-default list may see less organic sharing than a public-by-default one would. I also confirmed that `get_watchlist()` doesn't currently enforce the `public` flag at all (it's stored and returned but not used to filter access) — regardless of the default, this is a real gap, since right now anyone who knows a `user_id` can see the full watchlist through `GET /watchlist/<user_id>`. Changing the default is the right first step, but enforcing it in the route/service layer would be needed for `public` to actually function as a privacy control.

## Comment 5 — Sort order
**My position:** Agreed with @dev-lead — changed `get_watchlist()` to sort by `date_added` descending (most recent first) instead of alphabetical by title.

**Reasoning:** When I think about actually looking at a want-to-watch list, I want to see what I just added, not scroll alphabetically to find it. Recency is what's relevant to a watchlist — it's a queue of what to watch next, not a reference list you'd look something up in by name.

**Engagement with reviewer's point:** This also brings the watchlist in line with `get_collection()`, which already sorts by `date_added.desc()`. The two features were inconsistent with each other for no clear reason — alphabetical on one, recency on the other — and @dev-lead's reasoning ("most users want to see what they added recently") applies just as much to the watchlist as it already does to the collection view. I don't see a strong case for alphabetical as the *default*; it's more useful as an optional sort a user could choose later, not the first thing they see.

## Comment 6 — Rebase
**What conflicted:** Two files conflicted during `git rebase origin/main`:
1. `.gitignore` — both branches had independently added one, with slightly different entries (main's version was missing `.pytest_cache/`, which mine included).
2. `models.py` — `main` had refactored `Film.id` from `Integer` to `String(36)` (UUID) and updated `CollectionEntry.film_id` to match. My branch's `WatchlistEntry` class didn't exist yet on `main` at that point in history, so git flagged it as a straight add/modify conflict rather than a line-level diff.

**How I resolved it:** For `.gitignore`, merged both lists into one file with no duplicate entries. For `models.py`, kept my `WatchlistEntry` class as-is but updated its `film_id` column from `db.Integer` to `db.String(36)` to match the same UUID pattern already applied to `CollectionEntry.film_id` and `Film.id`.

**How I verified no conflict remains:** Ran `grep -n "<<<<<<<\|=======\|>>>>>>>" models.py` to confirm no leftover conflict markers, then ran the full test suite (`pytest tests/ -v`) after the rebase completed — all 6 tests pass, confirming the UUID-typed `film_id` works correctly with the rest of the watchlist and collection logic. Also confirmed via `git log --oneline` that the branch history is fully linear with no merge commits.

## PR Description
<!-- Written at the end — feature overview, design decisions, manual testing steps -->
