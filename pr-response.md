# PR Response Doc — CineLog Watchlist Feature

## AI Usage
I used an AI coding assistant (Claude Code) throughout, in these specific ways:

1. **Codebase orientation.** Before touching anything, I had the AI summarize
   `models.py` and `services/collection_service.py` — what each is responsible
   for, what depends on what, and how the collection service handles
   missing/duplicate IDs. This is how I found the `collection_service` pattern
   that Comments 2 and 3 told me to mirror, and how I first noticed the
   integer-vs-UUID mismatch that Comment 6 later confirmed.

2. **Stress-testing my design arguments (Comments 4 & 5).** After drafting my
   responses, I asked the AI to act as a skeptical reviewer: *"What is the
   strongest counterargument to this position, and what tradeoff am I not
   acknowledging?"* It surfaced two things I had genuinely missed:
   - *Comment 4:* my draft claimed the per-entry `public` opt-out "exists." The
     AI checked the code and showed it is **unenforced** — `get_watchlist` never
     filters on `public` and there's no auth/viewer concept. I rewrote my
     position around that: keep `public=True` as intended, but name the missing
     enforcement as required follow-up rather than pretending the opt-out works.
     That is a materially different (and more honest) argument than my first
     draft.
   - *Comment 5:* the AI pointed out I'd set up a false binary (alphabetical vs
     newest-first) and never engaged *oldest-first / FIFO queue*, the most
     on-point alternative, and that my sort had no tie-breaker. I added explicit
     engagement with the FIFO option and added an `id`-ascending tie-breaker plus
     a tie-case test. The final decision (newest-first) is still my own reasoning;
     the AI changed how thoroughly I defended it, not the conclusion.

3. **Verifying commit format.** Before finalizing history, I gave the AI my
   `git log --oneline` and asked whether every message met Conventional Commits
   and whether any bundled multiple logical changes. It confirmed the format and
   flagged that `refactor: sort ...` was semantically wrong (a sort-order change
   alters observable behavior, so it isn't a pure refactor). I verified that
   against the spec myself and reworded it to `fix:`.

The AI did not write my design positions for me — it attacked drafts I had
already written, and I incorporated only the gaps that held up.

## Comment 1 — Rename
**What I did:** Renamed `save_to_watchlist()` to `add_to_watchlist()` in
`services/watchlist_service.py` to follow the project's `verb_to_noun` naming
convention (matching `add_to_collection()`). Updated both call sites in
`routes/watchlist/watchlist.py` — the import and the invocation in `add_film`.
Also updated the docstring summary ("Save a film" → "Add a film").

**How I verified:** `grep save_to_watchlist` returns no matches; confirmed the
app imports cleanly (`create_app()` succeeds, so the route's import resolves);
ran `pytest tests/` — all 4 existing tests pass. Committed as
`refactor:` since this is a rename with no behavior change.

## Comment 2 — Deduplication
**What I did:** Mirrored the collection service's proven pattern. In
`services/watchlist_service.py` I added an `AlreadyInWatchlistError` exception
and, in `add_to_watchlist()`, a check that queries for an existing
`(user_id, film_id)` `WatchlistEntry` before inserting — raising the error
instead of creating a duplicate. I also wired up `routes/watchlist/watchlist.py`,
which previously imported `FilmNotFoundError` but caught nothing (so both errors
would have surfaced as 500s): the `add` handler now maps `FilmNotFoundError` →
404 and `AlreadyInWatchlistError` → 409, matching the collection route exactly.

**How I verified:** Ran a manual script against an in-memory DB — adding the
same film twice raises `AlreadyInWatchlistError` and the entry count stays at 1.
`pytest tests/` still passes (4/4). Committed as `fix:` because the duplicate
insert was a defect. (Note: unlike `CollectionEntry`, `WatchlistEntry` has no
DB-level `UniqueConstraint`; the app-level check is the primary guard, matching
how the collection service raises before hitting its constraint. Adding a
constraint would need a `models.py` change + migration and remains a reasonable
future hardening, but the app-level check fully satisfies the reviewer's request.)

## Comment 3 — Missing test
**What I did:** Created `tests/test_watchlist.py` following the structure of
`tests/test_collection.py` (same `app` / `sample_user` / `sample_film`
fixtures). The reviewer specifically asked for the nonexistent-`film_id` case,
and CONTRIBUTING.md requires happy-path + duplicate + nonexistent for any new
service function, so I wrote all three:
- `test_add_to_watchlist_creates_entry` — happy path, entry persists.
- `test_add_to_watchlist_duplicate_raises` — duplicate raises
  `AlreadyInWatchlistError` and only one row exists (covers Comment 2).
- `test_add_to_watchlist_nonexistent_film_raises` — a fake UUID raises
  `FilmNotFoundError` (the reviewer's requested case).

**How I verified:** `pytest tests/test_watchlist.py -v` — all 3 pass; full
suite now 7/7. The nonexistent-film test uses the same fake UUID string as the
collection test, so it stays correct after the Comment 6 UUID rebase.

## Comment 4 — Default visibility
**My position:** Keep `public=True` as the *intended* default, and document it
as forward-looking — because the flag is not yet enforced. No functional code
change in this PR (reason below).

**What behavior I'm optimizing for:** CineLog is described (README) as a
*community film tracking app* — its value comes from users discovering each
other's taste. Public-by-default keeps that discovery working out of the box; a
private-by-default watchlist would be invisible to the community until each user
manually flipped a flag, quietly killing the social feature for the majority who
never touch defaults. For a community product, the default should serve the
community use case with privacy available as an opt-out.

**Tradeoff acknowledged:** A watchlist reveals *intent* ("films I want to watch"),
which is arguably more sensitive than a collection of films already watched.
Public-by-default means some users expose more than they realize. That is a real
cost; I accept it for a community app only if the default is clearly communicated
and a working per-entry opt-out exists.

**Honest gap (surfaced by stress-testing this argument — see AI Usage):** that
opt-out does **not** currently function. `get_watchlist()` filters only by
`user_id` and never reads `public`, and `view_watchlist` returns every entry to
any caller — there is no authenticated "viewer" (the `user_id` is just a URL path
param). So today a `public=False` entry is served exactly like a public one; the
flag is stored but unenforced. Two consequences:
1. The debate over the *default value* is currently moot — with no enforcement,
   `True` vs `False` produce the same observable result (everything is visible).
2. The correct conclusion is therefore **not** "the default is fine, move on." It
   is: keep `public=True` as the intended default, but the real work is
   *enforcing* it — `get_watchlist` should return all entries to the owner and
   only `public=True` entries to others. That requires an identity/auth layer
   CineLog does not yet have, so it is out of scope for this PR and called out
   here as required follow-up rather than left as a silent dead field. Once
   enforcement exists, revisiting the default to `False` for privacy would be a
   small, reasonable follow-on decision.

## Comment 5 — Sort order
**My position:** Agree with the reviewer — sort by `date_added` descending
(newest first). Implemented in `get_watchlist()`.

**Reasoning:** Two reinforcing arguments. (1) It matches what most users want
from a "saved for later" list — see what you just added at the top. (2) It
makes the two list endpoints consistent: `get_collection()` already sorts
newest-first, so aligning `get_watchlist()` removes a surprising inconsistency
between two nearly identical endpoints.

**Engagement with reviewer's point:** The reviewer invited discussion, so I
weighed the two live alternatives:
- *Alphabetical by title* (the original code): easier to locate a known title in
  a long list. I rejected it — a watchlist's job is "what should I watch next,"
  not "find this specific film," so recency is the more useful primary axis.
- *Oldest-first (FIFO queue)*: a genuinely strong alternative I want to name
  explicitly, because "what to watch next" arguably favors surfacing the films
  you've been meaning to watch longest, not your latest impulse-adds. I still
  chose newest-first for two reasons: (1) it matches the mental model of the add
  action ("I just added this, show me it worked"), and (2) it keeps the watchlist
  consistent with `get_collection()`, which already sorts newest-first —
  divergent sort orders across two near-identical endpoints is a worse surprise
  than the FIFO-vs-recency nuance. This is a defensible default rather than a
  provably correct one; if usage data showed users treat the watchlist as a
  strict queue, oldest-first would be worth revisiting.

**Robustness detail:** `date_added.desc()` alone is non-deterministic when two
entries share a timestamp (e.g. a bulk add), so I added `id` ascending as a
deterministic tie-breaker and a test that exercises the tie case (not just the
5-day-gap happy path). I also dropped the now-unnecessary `join(Film)` that only
existed to enable the alphabetical sort.

## Comment 6 — Rebase
**What conflicted:** `git fetch origin` then `git rebase origin/main` replayed my
7 branch commits onto main (which carries the integer→UUID film-ID migration).
The single conflict was in `models.py`: main had migrated `Film.id` and
`CollectionEntry.film_id` to `db.String(36)` UUIDs, while my branch's
`WatchlistEntry` still declared `film_id = db.Column(db.Integer, ...)`. Git
surfaced the entire `WatchlistEntry` class as the incoming change (the class
addition folded into the "add Film relationship" commit during replay), with the
integer `film_id` as the point of divergence.

(Pre-step: I first removed my untracked local `.gitignore` — backed up to the
scratchpad — because main already ships a `.gitignore` (a superset that also
ignores `.pytest_cache/`), and an untracked file would have blocked the rebase
checkout. Post-rebase the branch inherits main's committed `.gitignore`.)

**How I resolved it:** Kept the `WatchlistEntry` class but changed
`film_id = db.Column(db.Integer, db.ForeignKey("film.id"), ...)` →
`db.Column(db.String(36), db.ForeignKey("film.id"), ...)` so its foreign key
matches main's UUID `Film.id`, preserving the `film` relationship and `public`
column. Then `git add models.py` + `git rebase --continue`. I also swept the
watchlist code for stale integer references the reviewer warned about and fixed
two docstring/comment mentions (`film_id (int)` → `film_id (str): UUID`,
`Body: { "film_id": <int> }` → `"<uuid>"`) in a follow-up `docs:` commit. The
service body already used `db.session.get(Film, film_id)`, which works with
UUID strings unchanged.

**How I verified no conflict remains:**
- `grep` for conflict markers (`<<<<<<<`, `=======`, `>>>>>>>`) in `models.py` —
  none.
- `git log --merges origin/main..HEAD` is empty → linear history, no merge
  commits (satisfies CONTRIBUTING's "no merge commits" rule).
- `git log --oneline origin/main..HEAD` shows my 7 commits cleanly on top of
  main.
- `pytest tests/` → 9/9 pass.
- End-to-end runtime check on an in-memory DB: `Film.id` is a UUID string,
  `add_to_watchlist` stores the matching UUID `film_id`, `get_watchlist` returns
  it, and the duplicate / nonexistent-film paths still raise the right errors.
- Confirmed `.gitignore` is now tracked (inherited from main) so generated files
  (`.venv/`, `*.db`, `__pycache__/`) stay out of the working tree status.

## Final commit history

Output of `git log --oneline` for the branch (newest first). **Action item:**
paste an actual screenshot of this terminal output here before submitting.

```
0cd275d docs: update watchlist film_id references from int to UUID
dbebfab fix: order watchlist by date added, newest first
5f934fc test: add watchlist service tests
3612f0d fix: reject duplicate films on the watchlist
f38b2bc refactor: rename save_to_watchlist to add_to_watchlist
382e56c fix: update film retrieval method to use db.session.get in collection and watchlist services
9b7d283 feat: add watchlist model, service, and endpoints
```

7 feature commits, all Conventional Commits, one logical change each, linear
history, no merge commits. (This response doc is added on top as a separate
`docs:` commit, so `git log` shows 8 commits total.)

---

## PR Description

### What this adds
A **watchlist** feature for CineLog: users can save films they want to watch
later (distinct from their collection of films already watched). It adds a
`WatchlistEntry` model, a watchlist service (`add_to_watchlist` / `get_watchlist`),
and two endpoints:

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET`  | `/watchlist/<user_id>` | List a user's watchlist, newest-added first |
| `POST` | `/watchlist/<user_id>/add` | Add a film (`{"film_id": "<uuid>"}`); 404 if the film doesn't exist, 409 if it's already on the list |

### Design decisions
1. **Default visibility — `public=True` (public by default).** CineLog is a
   community app, so watchlists are discoverable out of the box; privacy is an
   opt-out. Caveat documented in this PR: the `public` flag is currently *stored
   but not enforced* (no viewer/auth concept yet), so enforcing owner-vs-others
   visibility is required follow-up. See Comment 4 above for full reasoning.
2. **Sort order — `date_added` descending (newest first), with `id` as a
   deterministic tie-breaker.** Matches `get_collection()`'s convention for
   consistency and surfaces recent adds first. I considered and rejected
   alphabetical and oldest-first/FIFO ordering — see Comment 5 above.

### How to manually test end to end
```bash
# 1. Install and run
pip install -r requirements.txt
python app.py            # serves http://127.0.0.1:5000 (no frontend; use curl)

# 2. In a second shell, create a user and a film to get their UUIDs
python -c "
from app import create_app, db
from models import User, Film
app = create_app()
with app.app_context():
    u = User(username='ada', email='ada@example.com'); db.session.add(u)
    f = Film(title='Arrival', year=2016, genre='Sci-Fi'); db.session.add(f)
    db.session.commit()
    print('USER', u.id); print('FILM', f.id)
"
# copy the printed USER and FILM UUIDs into the commands below

# 3. Add the film to the watchlist -> 201 with the new entry
curl -s -X POST http://127.0.0.1:5000/watchlist/<USER>/add \
     -H "Content-Type: application/json" -d '{"film_id": "<FILM>"}'

# 4. View the watchlist -> 200 with a list containing the film
curl -s http://127.0.0.1:5000/watchlist/<USER>

# 5. Add the same film again -> 409 (duplicate rejected)
curl -s -X POST http://127.0.0.1:5000/watchlist/<USER>/add \
     -H "Content-Type: application/json" -d '{"film_id": "<FILM>"}'

# 6. Add a nonexistent film -> 404
curl -s -X POST http://127.0.0.1:5000/watchlist/<USER>/add \
     -H "Content-Type: application/json" \
     -d '{"film_id": "00000000-0000-0000-0000-000000000000"}'
```
Automated coverage: `pytest tests/` (9 passing, including watchlist happy-path,
duplicate, nonexistent-film, sort-order, and tie-break cases).
