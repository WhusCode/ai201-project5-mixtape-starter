# Mixtape — Submission

## Codebase Map

### Architecture

The app follows a strict layered pattern: Flask app factory (`app.py`) registers Blueprints (`routes/`), which delegate to service functions (`services/`), which operate on SQLAlchemy models (`models.py`). Every route is thin — parse the request, call exactly one service function, format the response as JSON. All business logic lives in `services/`. Services raise `ValueError` on bad input or missing records, and routes catch that to return the appropriate HTTP status code (400 for bad input, 404 for not found).

### Main files

- **`app.py`** — Flask application factory (`create_app`). Configures the SQLAlchemy database URI (defaults to local SQLite), registers the four blueprints (`songs`, `playlists`, `users`, `feed`) under their URL prefixes, and creates all tables on startup via `db.create_all()`.

- **`models.py`** — Defines 7 SQLAlchemy models: `User`, `Tag`, `Song`, `ListeningEvent`, `Rating`, `Playlist`, `Notification`. Also defines 3 association tables: `friendships` (symmetric many-to-many, self-referential on `User`), `song_tags` (many-to-many, `Song` ↔ `Tag`), and `playlist_entries` (many-to-many between `Playlist` and `Song`, but with extra columns — `position`, `added_by`, `added_at`). The `position` column is what makes playlists explicitly ordered rather than relying on insertion order or timestamp, and it's directly relevant to Issue #5.

- **`routes/songs.py`** — Search, get-by-id, rate, and listen endpoints. `POST /songs/<id>/rate` calls `notification_service.rate_song()`; `POST /songs/<id>/listen` calls `streak_service.record_listening_event()`.

- **`routes/playlists.py`** — Create playlist, get playlist metadata, get playlist songs, add song to playlist. The add-song endpoint calls `notification_service.add_to_playlist()`, which both records the addition and notifies the original sharer.

- **`routes/users.py`** — Get user, get streak, get/mark notifications. The basic user lookup queries `User` directly rather than going through a service; everything else delegates to `streak_service` or `notification_service`.

- **`routes/feed.py`** — Listening-now and activity-feed endpoints, both delegating to `feed_service`.

- **`services/streak_service.py`** — Owns listening-streak logic. `record_listening_event` logs a `ListeningEvent` and calls `update_listening_streak`, which compares `last_listened_at.date()` to the current date: same day = no-op, exactly one day later = increment, more than one day = reset to 1.

- **`services/feed_service.py`** — `get_friends_listening_now` filters a user's friends' listening events to a recency window and dedupes to the most recent song per friend. `get_activity_feed` returns the same shape without the recency filter, capped by `limit`.

- **`services/search_service.py`** — `search_songs` joins `Song` to `song_tags` and filters by title/artist substring match (case-insensitive).

- **`services/notification_service.py`** — `create_notification` is the generic writer used elsewhere in the codebase. `add_to_playlist` adds a song to a playlist and notifies the original sharer (unless they added it themselves). `rate_song` validates and upserts a `Rating` record. Despite its name, this file is really "notifications + ratings" — a grouping that isn't obvious from the filename alone.

- **`services/playlist_service.py`** — `create_playlist`, `get_playlist` (metadata only), `get_playlist_songs` (ordered by `position`), `get_user_playlists`.

### Data flow — user rates a song

`POST /songs/<song_id>/rate` in `routes/songs.py` parses `user_id` and `score` from the JSON body and calls `notification_service.rate_song(user_id, song_id, score)`. That function validates the score is between 1 and 5, confirms the song and user exist, then checks for an existing `Rating` row for that `(user_id, song_id)` pair (enforced by a unique constraint) — updating it if present, otherwise inserting a new one. The route returns the resulting rating as JSON with a 201.

Notably, this function lives in `notification_service.py` but never creates a `Notification`. Compare this to `add_to_playlist` in the same file, which *does* call `create_notification` after adding a song. This asymmetry is the root of Issue #4 — a friend rating your song produces no notification, even though a friend adding your song to a playlist does.

### Patterns noticed

1. **Routes are uniformly thin.** Every route follows parse-input → call-one-service → jsonify-output, with no business logic in the route layer itself.
2. **Service file names don't always reflect current responsibility.** `notification_service.py` contains rating logic (`rate_song`) that has nothing to do with notifications by itself — it's grouped there historically, not by what it currently does. This matters when tracing "who calls what," since the natural guess ("rating logic must live in a rating service") is wrong.
3. **Association tables carry metadata, not just foreign keys.** `playlist_entries` stores `position`, `added_by`, and `added_at` — ordering and provenance are first-class, not incidental. Any code that reads playlist contents needs to respect `position`, not just insertion order.
4. **Every model defines `to_dict()`** for consistent JSON serialization at the API boundary, so services return dicts (already serializable) rather than raw model instances.
5. **Tests double as bug specs.** Several existing test files contain comments directly on the failing assertions (e.g. `# Bug causes this to return 4`) — these were useful both for identifying which issues had ready-made reproduction tools and for confirming expected vs. actual behavior precisely.

---

## Bug Reproductions

### Issue #1 — Listening streak resets on Saturday → Sunday

**How I reproduced it:** Ran `pytest tests/test_streaks.py::test_streak_increments_on_sunday -v`. The test sets a user's `last_listened_at` to a Saturday, then calls `update_listening_streak` with a Sunday timestamp exactly one calendar day later. Expected: streak increments 1→2. Actual: `assert 1 == 2` fails — streak resets to 1. This is a wall-clock-date-driven condition rather than something triggerable via request input, so the unit test was the correct reproduction vehicle rather than a live HTTP request.

**How I found the root cause:** The failing test pointed directly at `update_listening_streak` in `services/streak_service.py`. Reading the function top to bottom, the day-gap logic has three branches: same-day (no-op), one-day-gap (increment), and anything else (reset). The one-day-gap branch reads `elif days_since_last == 1 and today.weekday() != 6:` — seeing the `weekday() != 6` clause was the moment I was confident this was the exact cause, since `datetime.weekday()` returns `6` specifically for Sunday, and there's no streak rule that says Sundays should behave differently from any other day.

**The root cause:** The increment condition required both a one-day gap *and* that the current day not be a Sunday. Since `weekday()` returns 6 for Sunday, any listen occurring exactly one day after the previous one, where that day happens to be a Sunday, fails the `!= 6` check and falls through to the `else` branch, resetting the streak to 1 instead of incrementing it. Every other day of the week correctly incremented; only the Saturday→Sunday transition was affected.

**My fix and side-effect check:** Removed the `and today.weekday() != 6` clause, leaving `elif days_since_last == 1:` as the sole condition for incrementing — the streak logic should only depend on the calendar-day gap, not the day of the week. After the fix, all 5 tests in `test_streaks.py` pass, including the previously-failing Sunday case. The other four tests (new user, consecutive day, same-day no-op, skipped-day reset) still pass unchanged, confirming the fix only affected the one path it was meant to fix. No other file in the codebase references `weekday()` or writes to `listening_streak`, so there was no other code path to check.

---

### Issue #4 — No notification when a friend rates your song

**How I reproduced it:** End-to-end via the running app. Checked `GET /users/<darius_id>/notifications` for the sharer of "Block Party" — baseline `count: 0`. Had a different user (`nova`) rate the song via `POST /songs/<id>/rate`. Re-checked notifications — still `count: 0`, confirming no notification was created despite a friend rating the sharer's song.

**How I found the root cause:** Since the reported behavior involved notifications, I went straight to `services/notification_service.py` and read both functions that write `Rating`/`Notification` data: `add_to_playlist` and `rate_song`. `add_to_playlist` ends with a call to `create_notification`, guarded by a check that the adder isn't the original sharer. `rate_song` performs the same shape of operation — validate, upsert, commit — but has no equivalent call at the end. Seeing the two functions side by side, doing structurally parallel work but only one of them notifying, was what confirmed this wasn't a subtle logic error but a missing step entirely.

**The root cause:** `rate_song` saves the `Rating` record and returns, but never calls `create_notification`. There's no logic error in the function — the notify step for the song's original sharer is simply absent, unlike the equivalent step in `add_to_playlist`.

**My fix and side-effect check:** Added a `create_notification` call at the end of `rate_song`, guarded by `if song.shared_by != user_id` to skip self-notification, mirroring the exact pattern already used in `add_to_playlist`. I verified two cases: (1) a different user rating the song correctly produces a notification for the sharer with the rater's username, song title, and score in the body; (2) the sharer rating their own song produces no new notification, and the existing notification from case 1 remains unaffected. No existing tests cover this file's notification behavior, so both cases were verified live via HTTP requests rather than pytest.

---

### Issue #5 — Last playlist song never shows up

**How I reproduced it:** Ran `pytest tests/test_playlists.py::test_playlist_returns_all_songs -v`. The test seeds a playlist with 5 songs at positions 1–5 and asserts `len(songs) == 5`; it fails, returning 4 instead.

**How I found the root cause:** The failing test pointed directly at `get_playlist_songs` in `services/playlist_service.py`. The function queries songs joined against `playlist_entries`, ordered ascending by `position` — that part looked correct. The return line was `[song.to_dict() for song in songs[:-1]]`. Seeing the `[:-1]` slice on an already-correctly-ordered list was the moment I was confident this was the cause — there's no reason to drop the last element of a correctly ordered, already-filtered result set, and the test's own comment ("Bug causes this to return 4") confirmed it lined up exactly with a 5-song playlist losing one.

**The root cause:** The function's return statement sliced the ordered song list with `[:-1]`, which unconditionally discards the last item in the list regardless of playlist size. Since the list is ordered ascending by `position`, this always drops the song in the highest position — the last song added — from every playlist's results, no matter how many songs it contains.

**My fix and side-effect check:** Changed the return statement from `[song.to_dict() for song in songs[:-1]]` to `[song.to_dict() for song in songs]`, removing the slice entirely. After the fix, all 3 tests in `test_playlists.py` pass, including the previously-failing all-songs test and the ordering test. I also checked the empty-playlist boundary (`test_empty_playlist_returns_empty_list`), which passed both before and after the fix since slicing an empty list with `[:-1]` doesn't raise or change anything — this test wouldn't have caught the bug on its own, which is worth noting for future test coverage. While verifying the single-song boundary live, I discovered a separate, pre-existing bug unrelated to Issue #5: adding a song to a playlist via `POST /playlists/<id>/songs` throws a 500 error (`NOT NULL constraint failed: playlist_entries.position`), because `add_to_playlist` in `notification_service.py` appends to the `playlist.songs` relationship directly, which does not populate the extra `position`, `added_by`, and `added_at` columns on the `playlist_entries` association table. This bug is out of scope for Issue #5 (it affects adding songs, not retrieving them) and isn't one of the five tracked issues, so I did not fix it, but I'm noting it here since it fully blocks the "add song to playlist" feature for any playlist not created via `seed_data.py`.