# PR Response Doc — CineLog Watchlist Feature

## AI Usage
I used AI for ideas — mainly to help me get oriented in the codebase and bounce around approaches. The code changes, the design positions in Comments 4 and 5, and the final reasoning are my own.

## Comment 1 — Rename
**What I did:** Renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py` so the watchlist service matches the project's established `add_to_collection()` naming convention (verb `add` + `to_` + noun). Updated the one call site in `routes/watchlist/watchlist.py`, which had two references: the `from services.watchlist_service import ...` line and the call inside the `/add` endpoint.

**How I verified:** Before editing I ran a project-wide search (`grep -rn "save_to_watchlist" --include="*.py"`) to locate every reference — the definition plus the import and the call in the route. After renaming all three, I re-ran the same search and it returned nothing. I then confirmed the app factory still imports (`from app import create_app; create_app()`) and the full suite still passes (`pytest tests/ -q` → 4 passed).

## Comment 2 — Deduplication
**What I did:** Added a duplicate check to `add_to_watchlist()` that mirrors the existing `add_to_collection()` pattern in `services/collection_service.py`. Before creating a new `WatchlistEntry`, the function queries `WatchlistEntry.query.filter_by(user_id=user_id, film_id=film_id).first()`; if a row already exists it raises a new `AlreadyInWatchlistError` instead of silently inserting a second row. I modeled the new exception on `AlreadyInCollectionError` (same purpose, same message shape) and defined it in `watchlist_service.py`, matching how the collection service keeps its own errors local. The check is placed after the `FilmNotFoundError` check, in the same order as the collection service.

**How I verified:** I studied `add_to_collection()` first: it does the `filter_by(...).first()` lookup, and on a hit raises `AlreadyInCollectionError` — meaning a duplicate returns nothing and creates no row. I then wrote the equivalent for the watchlist and ran a manual check in a throwaway in-memory app: adding the same (user, film) twice raised `AlreadyInWatchlistError` on the second call, and `WatchlistEntry.query...count()` returned exactly 1. The full suite (`pytest tests/ -q`) still passes.

## Comment 3 — Missing test
**What I did:** Created `tests/test_watchlist.py`, modeled directly on `test_add_to_collection_nonexistent_film_raises` in `tests/test_collection.py`. I reused the same three fixtures (`app` with an in-memory SQLite database, `sample_user`, `sample_film`) so the setup matches the collection tests exactly, then wrote `test_add_to_watchlist_nonexistent_film_raises`: it calls `add_to_watchlist()` with a `film_id` that isn't in the database and asserts that `FilmNotFoundError` is raised (rather than a raw SQLAlchemy integrity error). The fake ID is currently an integer (`999999`) because the branch still uses integer film IDs at this point; it is switched to a nonexistent UUID string in Comment 6 after the rebase.

**How I verified:** Ran `pytest tests/test_watchlist.py -v` — the single test passes. Then ran the full suite (`pytest tests/ -q`) — all 5 tests pass (4 collection + 1 watchlist), confirming the new file doesn't interfere with existing tests.

## Comment 4 — Default visibility
My position: Keep public=True by default, but make it easy to switch to private.

CineLog is built around sharing what you want to watch, so public watchlists help power recommendations and social discovery. A watchlist is also less personal than ratings or viewing history since it reflects future plans, not past behavior. That said, I understand the privacy concern. I think this default only makes sense if users can easily change it, such as with a visibility toggle when adding movies or a profile setting to keep the entire watchlist private.

## Comment 5 — Sort order
My position: I agree with the maintainer and changed the watchlist to sort by date_added (newest first).

This matches how collections are already sorted, making the app more consistent. It also fits how people use a watchlist—the newest additions are usually what they're most interested in watching next. Alphabetical order can still be useful, but I'd rather support it as an optional sort instead of making it the default.

## Comment 6 — Rebase
**What conflicted:** Two things surfaced when I rebased `feature/watchlist` onto the updated `main` (`git fetch origin` + `git rebase origin/main`):
1. **`.gitignore` (add/add conflict):** `main` had gained its own `.gitignore` while my PR was open, and my branch added one too. Git flagged an add/add conflict on the file.
2. **Film ID type (the real conflict):** `main` migrated all film IDs from integer to UUID (`db.String(36)`). My watchlist code was written before that refactor, so it still treated `film_id` as an integer — `WatchlistEntry.film_id` was `db.Integer` with an integer foreign key, the service/route docstrings described `film_id` as an int, and my new test used an integer sentinel (`999999`) for the "nonexistent film" case.

**How I resolved it:** For the `.gitignore` I took the union of both versions (kept `main`'s superset, which additionally ignores `.pytest_cache/`); since the result was identical to `main`'s file, my redundant `.gitignore` commit dropped out of the rebase — the branch still has a `.gitignore`, it just comes from `main`. For the UUID conflict I updated the watchlist code to match the refactor: `WatchlistEntry.film_id` is now `db.Column(db.String(36), db.ForeignKey("film.id"))`, exactly like `CollectionEntry.film_id`; the docstrings now describe `film_id` as a UUID string; and the test's nonexistent-film sentinel is now a UUID string (`"00000000-0000-0000-0000-000000000000"`), matching how `test_add_to_collection_nonexistent_film_raises` does it. I used an interactive rebase (`git rebase -i`) to make sure the `WatchlistEntry` model landed cleanly on top of the UUID `Film` model rather than being silently mismerged.

**How I verified no conflict remains:** `git log --oneline --merges origin/main..HEAD` returns nothing (no merge commits — the branch is rebased, not merged). `git log --oneline origin/main..HEAD` shows a clean linear history on top of `main`. The full test suite passes (`pytest tests/ -v` → 5 passed), the app factory imports without error, and I drove the endpoints end-to-end with Flask's test client: adding a film returns 201, a duplicate returns 409, a nonexistent (UUID) film returns 404, and `GET /watchlist/<user_id>` returns the films newest-added first.

## PR Description

### What the watchlist feature does
Adds a **watchlist** — films a user wants to watch *later* — as a sibling to the existing **collection** (films already watched). It introduces:
- A `WatchlistEntry` model (`user_id`, `film_id`, `date_added`, `public`), with `film_id` as a UUID foreign key consistent with the post-refactor `Film` model.
- A service layer (`services/watchlist_service.py`): `add_to_watchlist(user_id, film_id)` and `get_watchlist(user_id)`.
- REST endpoints (`routes/watchlist/watchlist.py`, registered under `/watchlist`):
  - `GET /watchlist/<user_id>` — returns the user's watchlist, newest-added first.
  - `POST /watchlist/<user_id>/add` with body `{ "film_id": "<uuid>" }` — adds a film.
- Deduplication: adding a film already on the watchlist raises `AlreadyInWatchlistError` (409) instead of creating a duplicate row.
- Errors are mapped to HTTP codes like the collection endpoint: unknown film → 404, duplicate → 409.

### Design decisions made
- **Default visibility (`public=True`)** — new watchlist entries are public by default to power CineLog's community/discovery features, paired with an easy opt-out. Full reasoning and the tradeoff in *Comment 4* above.
- **Sort order (date-added, newest first)** — `get_watchlist()` sorts by `date_added` descending, matching `get_collection()` for a consistent mental model across both list views. Full reasoning in *Comment 5* above.

### How to manually test the feature
CineLog has no user- or film-creation endpoint (films are seeded, and the DB starts empty), so seed one user and two films first, then exercise the endpoints.

```bash
# 1. Set up and run
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
python app.py            # serves at http://127.0.0.1:5000

# 2. In a second terminal, seed a user + two films and print their IDs
python - <<'PY'
from app import create_app, db
from models import User, Film
app = create_app()
with app.app_context():
    u = User(username="omar", email="omar@example.com")
    f1 = Film(title="Zodiac", year=2007)
    f2 = Film(title="Amelie", year=2001)
    db.session.add_all([u, f1, f2]); db.session.commit()
    print("USER_ID =", u.id)
    print("FILM_1  =", f1.id, "(Zodiac)")
    print("FILM_2  =", f2.id, "(Amelie)")
PY

# 3. Add films (substitute the IDs printed above)
curl -s -X POST http://127.0.0.1:5000/watchlist/<USER_ID>/add \
  -H "Content-Type: application/json" -d '{"film_id": "<FILM_1>"}'      # -> 201
curl -s -X POST http://127.0.0.1:5000/watchlist/<USER_ID>/add \
  -H "Content-Type: application/json" -d '{"film_id": "<FILM_2>"}'      # -> 201

# 4. Verify deduplication (adding FILM_1 again) -> 409 with an error message
curl -si -X POST http://127.0.0.1:5000/watchlist/<USER_ID>/add \
  -H "Content-Type: application/json" -d '{"film_id": "<FILM_1>"}' | head -1

# 5. Verify a nonexistent film -> 404
curl -si -X POST http://127.0.0.1:5000/watchlist/<USER_ID>/add \
  -H "Content-Type: application/json" -d '{"film_id": "does-not-exist"}' | head -1

# 6. View the watchlist -> Amelie (added last) appears FIRST; each entry has "public": true
curl -s http://127.0.0.1:5000/watchlist/<USER_ID>
```

Automated coverage: `pytest tests/ -v` (5 passing; `tests/test_watchlist.py` covers the nonexistent-film case).

## Commit history (git log --oneline)

My feature commits, rebased cleanly on top of `origin/main` (`git log --oneline origin/main..HEAD`). Every message is in conventional-commit format, each is one logical change, and there are **no merge commits** (`git log --merges origin/main..HEAD` is empty):

```
c82d5d2 docs: add pr-response.md with review responses and design decisions
4c35c79 fix: map watchlist add errors to 404/409 responses like collection endpoint
dd5faf4 fix: migrate watchlist film_id references to UUID after main branch refactor
d786b83 refactor: sort watchlist by date added for consistency with collection
18ff14d fix: add WatchlistEntry-Film relationship so get_watchlist resolves films
016f062 test: add test for nonexistent film_id in add_to_watchlist
a6bb502 fix: add deduplication check to prevent duplicate watchlist entries
9cc07c3 fix: rename save_to_watchlist to add_to_watchlist per naming convention
acf655e refactor: use db.session.get instead of Query.get for film lookups
112319c fix: correct watchlist blueprint import path in app factory
36e2689 feat: add watchlist model and add_to_watchlist endpoint
```

> The `docs:` commit above is the tip; after committing this file the top hash will differ by a character or two. Run `git log --oneline` for the exact final values and paste a screenshot here if your submission requires an image.
