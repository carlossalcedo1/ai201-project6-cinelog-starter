# PR Response Doc — CineLog Watchlist Feature

## AI Usage

I used an AI assistant once for codebase orientation at the very start. Before writing any code I asked it to help me find where the watchlist feature lived and how it related to the existing collection feature, so I could follow the same patterns already used in the project. It pointed me to the watchlist service, route, and model, along with the parallel collection files I used as a reference. The design decisions and code changes are my own.

## Comment 1 — Rename
**What I did:** Renamed save_to_watchlist() to add_to_watchlist() in services/watchlist_service.py and updated the single call site in routes/watchlist/watchlist.py — both the import on line 8 and the invocation on line 32.

**How I verified:** Ran a project-wide grep for save_to_watchlist across the whole repo (excluding .git/venv dirs) before and after — it returned three matches before and zero after, confirming no call site was missed. Confirmed the three new add_to_watchlist references are the definition + import + call. Also parsed both edited files with Python's ast module to confirm they're still syntactically valid.

## Comment 2 — Deduplication
**What I did:** Followed the same pattern as add_to_collection() in collection_service.py. In watchlist_service.py I added an AlreadyInWatchlistError exception class and, inside add_to_watchlist(), a check that queries WatchlistEntry.query.filter_by(user_id, film_id).first() after the film-exists check and raises AlreadyInWatchlistError before creating a duplicate entry. I also mirrored the route handling: watchlist.py now catches AlreadyInWatchlistError and returns a 409, matching how the collection route translates AlreadyInCollectionError.

**How I verified:** Ran the existing test suite and all four test cases passed.

## Comment 3 — Missing test
**What I did:** Created tests/test_watchlist.py and wrote test_add_to_watchlist_nonexistent_film_raises, the watchlist equivalent of test_add_to_collection_nonexistent_film_raises. It reuses the same fixture structure (an app fixture with an in-memory SQLite DB and a sample_user fixture) and the same assertion structure — pytest.raises(FilmNotFoundError) when calling add_to_watchlist() with a film_id that isn't in the DB. The one difference from the collection test is the fake id: watchlist film_id is an integer (per the service docstring), so I used a nonexistent integer id rather than a fake UUID string.

**How I verified:** Ran pytest on the new file, all cases passed and the full suite and now the 5th case is passing too. Trimmed the imports down to only what the test uses so there are no unused imports.

## Comment 4 — Default visibility
**My position:** I'm keeping public=True as the default on WatchlistEntry, but treating it as a default rather than a locked-in policy. The model already exposes a per-entry public flag, so any user can make a given entry private without a schema change.

**Reasoning:** CineLog is a social film-logging app, and the watchlist is one of its most social surfaces, it's the "here's what I'm planning to watch" list that invites recommendations and comparison with friends, the same way Letterboxd watchlists are public by default. I'm optimizing for the discovery-and-sharing behavior: the majority of users who add a film to their watchlist want it to be visible so that (1) friends can suggest what to watch next or flag films they've already seen, and (2) the feature contributes to the network effects that make a social app worth using. A private-by-default watchlist would mean that most entries stay invisible simply because users rarely change defaults, which quietly guts the social value of the feature for the common case. Defaulting to public means the feature does the useful, expected thing out of the box, and the minority who want privacy can opt out per entry.

**Tradeoff acknowledged:** The real cost is that public-by-default violates the "privacy by default" / data-minimization principle: a user's data becomes visible before they've made a conscious choice, and people who don't read settings are exposed by inaction rather than by decision. A watchlist can reveal sensitive interest — a documentary about a medical condition, a film tied to a religion or sexuality someone hasn't disclosed. The safer-by-default alternative (public=False) and protects most end-users. I think public-by-default is the right call for a social product specifically because the per-entry flag already gives users an escape.

## Comment 5 — Sort order
**My position:** I'm adopting the maintainer's preference and changing get_watchlist to sort by date_added descending (newest additions first), replacing the alphabetical-by-title sort. This also makes get_watchlist consistent with get_collection which already sorts newest-first.

**Reasoning:** The sort key should reflect the decision the user is making on this screen, which is "what do I want to watch." A film's title carries no signal about that — alphabetical order sorts on an attribute that is irrelevant to the choice, so a film starting with "A" isn't more watch-worthy than one starting with "W." Date-added, by contrast, is a real signal of intent: it captures recency of interest and gives the list a stable, meaningful narrative ("here's what's been on my mind lately"). It also matches the mental model users already have from the collection view, so the two lists behave the same way instead of surprising the user with different orderings.

**Engagement with reviewer's point:** The maintainer's argument for date-added is that it reflects when the user added the film and stays consistent with the rest of the app, and I think that's correct on both counts — consistency with get_collection was the deciding factor for me. I do want to give the alphabetical choice its due: its one genuine merit is findability. In a long watchlist, alphabetical lets you scan to a title you already have in mind. But that's a lookup problem, not a browse-and-decide problem, and the right tool for lookup is search/filter, not the default sort order — optimizing the default for the rare "I know exactly what I'm looking for" case at the expense of the common "show me what I might watch" case is the wrong trade. I also considered a third option, date_added ascending (oldest-first), which has a real argument for a watchlist specifically: it surfaces the long-standing backlog you keep meaning to get to, instead of burying it under recent impulse-adds. I rejected it as the default because it fights the consistency win — get_collection is newest-first — and newest-first better matches "what am I currently interested in." Oldest-first is a good candidate for an optional sort toggle later, not the default.

## Comment 6 — Rebase
**What conflicted:** Nothing conflicted in the way git normally reports — and that was the trap. main's "migrate film IDs from integer to UUID" refactor had also deleted the WatchlistEntry model from models.py, and my branch never modified models.py (WatchlistEntry was inherited from the initial commit). So the three-way merge saw main delete a block that my side left untouched, applied the deletion cleanly, and reported "Successfully rebased" with no conflict prompt. The real conflict was semantic: my WatchlistEntry silently disappeared, and the parts of my watchlist code that still assumed an integer film_id (the model column, a service docstring, the route docstring, and the test's fake id) were now inconsistent with the UUID Film.id on main. There was also a working-tree blocker before the rebase could even start: I had an untracked .gitignore, and main adds a tracked one, so the initial checkout would have aborted.

**How I resolved it:** First I removed the untracked .gitignore (main's version is a superset — it also ignores .pytest_cache/) so the checkout could proceed, then ran git fetch origin and git rebase origin/main. After the rebase I restored the WatchlistEntry model to models.py, this time with film_id as db.String(36) (a UUID foreign key) instead of db.Integer, so it matches the refactored Film.id and CollectionEntry.film_id. I then updated the remaining integer assumptions to UUIDs: the film_id docstring in watchlist_service.py, the "film_id": <int> example in the watchlist route, and the fake id in test_watchlist.py (999999 → a UUID string, matching the collection test). All of this went into a single follow-up commit.

**How I verified no conflict remains:** Ran the full test suite (5 passed) and an end-to-end check that adds a film with a real UUID film_id through the route — first add returned 201, a duplicate returned 409 — confirming the UUID path works after the refactor. I grepped the watchlist code for leftover integer references (Integer, <int>, (int), 999999) and found none. For history hygiene I confirmed git log --merges origin/main..HEAD is empty (no merge commits), that git log --graph shows a straight line of 7 commits, and that HEAD~7 equals the origin/main tip — so the branch is rebased directly on top of main, not merged.

## PR Description

### What this feature does

This PR adds a watchlist to CineLog, a per-user list of films a user wants to watch later. It is separate from the collection, which tracks films a user has already watched.

It adds a WatchlistEntry model that links a user to a film and stores the date the film was added and a public visibility flag. It also adds two endpoints:

- POST /watchlist/<user_id>/add adds a film to the user's watchlist. The request body is {"film_id": "<uuid>"}.
- GET /watchlist/<user_id> returns the user's watchlist, newest addition first.

The endpoints behave as follows:

- Adding a film returns 201 with the new entry.
- Adding a film that is already on the watchlist returns 409 and does not create a duplicate.
- Adding a film_id that does not exist returns 404.
- A request with no film_id returns 400.

### Design decisions

Default visibility is public. New watchlist entries are public by default. CineLog is a social app and a watchlist is a social surface used for recommendations and comparing with friends, so the default favors sharing and discovery. The public flag is per entry, so a user can still make individual entries private. The tradeoff is that this favors sharing over privacy by default. The full reasoning is in the Comment 4 section of this doc.

Sort order is by date added, newest first. GET /watchlist returns the most recently added entries first, replacing the original alphabetical by title sort. Date added reflects what the user wants to watch now and matches the ordering that GET /collection already uses, while alphabetical order sorts on something that does not help the user decide what to watch. The full reasoning is in the Comment 5 section of this doc.

### How to manually test

1. Install dependencies:
   ```bash
   pip install -r requirements.txt
   ```

2. Seed a user and two films (there is no API to create these) and note the
   printed UUIDs. This writes to the same cinelog.db the server uses:
   ```bash
   python -c "
   from app import create_app, db
   from models import User, Film
   app = create_app()
   with app.app_context():
       u = User(username='alice', email='alice@example.com')
       f1 = Film(title='Dune', year=2021, genre='Sci-Fi')
       f2 = Film(title='Arrival', year=2016, genre='Sci-Fi')
       db.session.add_all([u, f1, f2]); db.session.commit()
       print('USER_ID =', u.id)
       print('FILM_1  =', f1.id)
       print('FILM_2  =', f2.id)
   "
   ```

3. Start the server:
   ```bash
   python app.py    # http://localhost:5000
   ```

4. Add the first film (expect 201):
   ```bash
   curl -i -X POST localhost:5000/watchlist/<USER_ID>/add \
        -H "Content-Type: application/json" -d '{"film_id":"<FILM_1>"}'
   ```

5. Add the second film (expect 201):
   ```bash
   curl -i -X POST localhost:5000/watchlist/<USER_ID>/add \
        -H "Content-Type: application/json" -d '{"film_id":"<FILM_2>"}'
   ```

6. View the watchlist (expect 200, with the most recently added film Arrival first):
   ```bash
   curl localhost:5000/watchlist/<USER_ID>
   ```

7. Try adding a duplicate (expect 409):
   ```bash
   curl -i -X POST localhost:5000/watchlist/<USER_ID>/add \
        -H "Content-Type: application/json" -d '{"film_id":"<FILM_1>"}'
   ```

8. Try adding a nonexistent film (expect 404):
   ```bash
   curl -i -X POST localhost:5000/watchlist/<USER_ID>/add \
        -H "Content-Type: application/json" -d '{"film_id":"does-not-exist"}'
   ```

9. Run the automated tests (expect all passing):
   ```bash
   pytest tests/
   ```
