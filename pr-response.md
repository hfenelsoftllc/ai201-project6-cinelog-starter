# PR Response — CineLog Watchlist Feature

## AI Usage

Used Claude Code to cross-reference `services/collection_service.py` while writing the
watchlist service and tests, to confirm naming, error-handling, and fixture patterns
matched the established convention rather than inventing a parallel style. This document
was also drafted with AI assistance, at a maintainer's suggestion that written rationale
be treated as a first-class part of the PR, not an afterthought.

## Comment 1 — Rename `save_to_watchlist` to `add_to_watchlist`

**What I did:** Renamed the function in `services/watchlist_service.py` and its import/call
site in `routes/watchlist/watchlist.py` to match the `verb_to_noun` convention used by
`add_to_collection`, `remove_from_collection`, and `get_collection`.

**How I verified:** Ran `pytest tests/` to confirm no caller still referenced the old name.

## Comment 2 — Prevent duplicate watchlist entries

**What I did:** `add_to_watchlist()` now checks for an existing `(user_id, film_id)` row
before inserting, raising `AlreadyInWatchlistError` (mapped to `409` in the route) instead
of letting a duplicate insert fail on the database constraint. Backed by a
`unique_user_film_watchlist` constraint on `WatchlistEntry`, mirroring
`unique_user_film_collection` on `CollectionEntry`.

**How I verified:** `test_add_to_watchlist_duplicate_raises` asserts the second call raises
and that only one row persists.

## Comment 3 — Missing test coverage

**What I did:** Added `tests/test_watchlist.py`, following the fixture and structure of
`tests/test_collection.py`: an `app` fixture with an in-memory DB, `sample_user`/
`sample_film` fixtures, and one test per behavior (happy path, duplicate, nonexistent film,
removal, removal of a nonexistent entry, and sort order).

**How I verified:** `pytest tests/test_watchlist.py -v` — all 6 tests pass.

## Comment 4 — Default visibility (`public=False`)

**My position:** `public` should default to `False`.

**Reasoning, grounded in this codebase specifically:** There is no auth layer on any route
in this API — `GET /watchlist/<user_id>` will return whatever is in the database for any
`user_id` a caller supplies, no session or ownership check involved. If watchlists defaulted
to public, that endpoint would double as a way to enumerate every user's viewing intent by
walking user IDs, with no opt-in from the user. Defaulting to private means a user has to
take an affirmative action (`public: true` in the request body) before their data is exposed
by design, rather than by omission.

**Tradeoff acknowledged:** This adds friction for the social use case CineLog is presumably
building toward (sharing a watchlist with friends) — users have to know the flag exists and
flip it. I judged that acceptable because the cost of the opposite mistake (a user's list
silently public because they didn't pass a flag) is worse than the cost of a small amount of
friction on an explicitly opt-in feature.

**Note for the next PR:** the `public` flag is stored but not yet enforced anywhere —
`get_watchlist()` returns every entry regardless of its value, and there's no route yet for
viewing *another* user's watchlist. The default matters now mainly so that whichever PR adds
that route later inherits a safe default instead of having to retrofit one.

## Comment 5 — Sort order (`date_added` descending)

**My position:** Sort by `date_added` descending (newest first).

**Reasoning:** This matches `get_collection()`'s existing sort order, so the two features
behave consistently. Chronological order also answers the more common question — "what did I
just add?" — without depending on film titles being clean, single-language strings.

**Engagement with reviewer's point:** Agreed with the pushback on the original alphabetical
sort — it breaks down as soon as titles use non-Latin scripts, articles ("The"), or numeric
prefixes, and it doesn't reflect user intent the way recency does.

## Comment 6 — Rebase onto `main`

**What conflicted:** `main`'s UUID refactor (`07ca580`) rewrote `models.py` wholesale,
including a version of the file that predated the watchlist branch's `WatchlistEntry` model.
Because this branch's own commits hadn't touched `models.py` yet at that point, `git rebase
origin/main` applied without conflict markers — but the replayed history left `models.py`
without `WatchlistEntry` at all. A "successfully rebased" rebase with no visible conflicts
still silently dropped a model.

**How I resolved it:** Diffed the pre- and post-rebase trees by hand to catch the missing
model, then restored `WatchlistEntry` in its own commit (`d154cc1`) with `film_id` typed as
`db.String(36)` to match the refactored `Film.id`/`CollectionEntry.film_id` columns.

**Additional bug found while restoring it:** `Film` had a `collection_entries` relationship
back to `CollectionEntry` (`backref="film"`) but no equivalent for `WatchlistEntry`. That gap
existed even before the rebase — the original watchlist commit (`1805e87`) never added it,
and nothing caught it because no test exercised `get_watchlist()` yet. It surfaced only once
`tests/test_watchlist.py` (`1c64c73`) actually called `get_watchlist()`, which calls
`entry.film.to_dict()` — that line would have raised `AttributeError` on every real request.
The fix (`watchlist_entries = db.relationship("WatchlistEntry", backref="film", lazy=True)`
on `Film`) landed in that same test commit, once the missing coverage made the bug visible.

**Why UUIDs matter here specifically:** with no auth layer (see Comment 4), sequential
integer IDs would let a caller iterate `film_id=1, 2, 3, ...` and read every record in the
table. UUIDv4 IDs aren't guessable in sequence, which is one less thing an unauthenticated
API surface has working against it.

**How I verified no conflict remains:** `pytest tests/ -v` — 10/10 passing — and confirmed
`db.create_all()` succeeds against an in-memory DB with the restored schema. Branch history
is linear from this branch's own commits (the one merge commit present, `bbe206c`, was
inherited from `main` and predates this branch).

## PR Description

**What this PR does:** Adds a watchlist feature — users can save films they intend to watch
later, list their watchlist (newest first), and remove entries — following the same
service/route split as the existing collection feature.

**Design decisions:**
1. **Visibility:** `public` defaults to `False`. See Comment 4 above for the full reasoning;
   short version is that this API has no auth layer, so a public-by-default flag would be an
   enumeration risk with no user opt-in.
2. **Sort order:** `date_added` descending, matching `get_collection()`'s existing behavior.
   See Comment 5.

**Manual test steps:**
1. `python app.py` on this branch.
2. `POST /watchlist/<user_id>/add` with a valid `film_id` (no `public` field) — confirm the
   response has `"public": false`.
3. Repeat the same request — confirm a `409` with an `AlreadyInWatchlistError` message.
4. `GET /watchlist/<user_id>` — confirm films are ordered newest-added first.
5. `DELETE /watchlist/<user_id>/remove` with the same `film_id` — confirm `200`, then repeat
   the delete and confirm `404`.
