# Mixtape — Submission

## Root Cause Analyses

### Bug 1 — My listening streak keeps resetting

**How I reproduced it.** The repo ships a pytest that already pins the bug down: `tests/test_streaks.py::test_streak_increments_on_sunday`. It sets up a user who listens on Saturday 2024-06-15 and again on Sunday 2024-06-16, then asserts the streak should be 2. Running `pytest tests/test_streaks.py -v` before touching any code, that test failed with `assert 1 == 2` — confirming the streak reset instead of incrementing on Sunday. The other four streak tests (new user, consecutive weekday, same-day double-count, skipped day) all passed, which narrowed the bug to the Sunday boundary specifically.

**How I found the root cause.** Function-trace top-down: `POST /songs/<id>/listen` → `routes/songs.py::listen` → `streak_service.record_listening_event` → `streak_service.update_listening_streak(user, now)`. Only the last function actually mutates the streak, so that's where I stopped. Reading it, the branch that increments the streak had an extra guard tacked onto it:

```python
elif days_since_last == 1 and today.weekday() != 6:
    user.listening_streak += 1
else:
    user.listening_streak = 1
```

`datetime.weekday()` returns `0=Monday … 6=Sunday`. The `!= 6` clause literally excludes Sunday from the "increment" branch. So Sat→Sun listens fell through to `else` and reset the counter. The moment I was sure this was the right place: the failing test uses exactly Saturday and Sunday dates with a comment `weekday() == 6`, so the test author was pointing at this line.

**The root cause.** The increment branch was gated on `today.weekday() != 6`, which excludes Sunday. On any Saturday-then-Sunday sequence, `days_since_last == 1` was True but the `and today.weekday() != 6` half was False, so the streak took the `else` branch and reset to 1. There is no rule in the docstring that the streak should be week-bounded — the guard was simply wrong.

**Fix and side-effect check.** Removed the `and today.weekday() != 6` half in `services/streak_service.py`. The increment branch now reads `elif days_since_last == 1:`, which matches the docstring rule "If the user listened yesterday: streak increments by 1." Ran the full `tests/test_streaks.py` suite — all 5 tests pass, including the 4 that were already passing (new user starts at 1, consecutive weekday increments, same-day no double count, skipped day resets). The `else` branch still handles gaps of >1 day, and the `days_since_last == 0` early return still handles same-day. No route or other service imports the weekday guard.

### Bug 2 — Friends Listening Now shows people from yesterday

**How I reproduced it.** No test file exists for the feed service, so I reproduced it by reasoning through the code with a concrete timestamp. The endpoint filters listening events where `listened_at >= now - RECENT_THRESHOLD`, and `RECENT_THRESHOLD` was `timedelta(hours=24)`. Concrete example: it's Monday 8:00 pm UTC, a friend listened Sunday 9:00 pm UTC (23 hours ago). Cutoff = Sunday 8:00 pm UTC. `Sunday 9pm >= Sunday 8pm` → True, so the friend appears in a feed labeled "listening now" on Monday evening. That friend hasn't opened the app since yesterday, but the UI shows them as currently listening — exactly the reported symptom.

**How I found the root cause.** Function-trace: `GET /feed/<user_id>/listening-now` → `routes/feed.py::listening_now` → `feed_service.get_friends_listening_now`. Inside that function, the only piece of logic that decides who counts as "now" is `cutoff = datetime.now(timezone.utc) - RECENT_THRESHOLD`. Followed `RECENT_THRESHOLD` back to line 13 and saw it was 24 hours — that's obviously not "now." The query, dedup, and ordering logic all read correctly; only the constant was wrong. The moment I was confident: grepping `RECENT_THRESHOLD` across the whole project returned only two hits — the definition and the one use inside this function — so nothing else was leaning on that value.

**The root cause.** `RECENT_THRESHOLD = timedelta(hours=24)` is not a "now" window. Anyone who listened at any point in the last 24 hours passed the filter, so friends whose most recent listen was yesterday (up to 23h59m ago) showed up in "listening now." A "now" feed needs a window measured in minutes, not hours — otherwise it can span calendar days.

**Fix and side-effect check.** Changed the constant to `timedelta(minutes=30)` in `services/feed_service.py:13`. Thirty minutes is long enough to catch a song that just finished playing and short enough that it cannot cross midnight. Side-effect check: grepped `RECENT_THRESHOLD` — only used inside `get_friends_listening_now`. The neighboring `get_activity_feed` function deliberately has no time filter (its docstring says: *"Unlike get_friends_listening_now, this is not filtered by recency"*), so it's unaffected. The query, dedup-per-friend, and ordering-by-most-recent logic are unchanged.

### Bug 3 — The same song keeps showing up twice in search

**How I reproduced it.** The `seed_data.py` comment on line 73 makes the intent explicit: *"Songs with 3+ tags — these are the ones that expose Issue #3."* The pre-written test `tests/test_search.py::test_search_no_duplicates_multi_tag_song` sets up a song with three tags and asserts the search returns it exactly once (the comment reads: `# Should be 1, bug causes it to be 3`). I ran the raw SQL manually to see what the query actually produces:

```
SELECT song.* FROM song LEFT OUTER JOIN song_tags ON song.id = song_tags.song_id WHERE ...
via raw execute: 3 rows
```

A song with three tags produces three rows from the join. The legacy `session.query(Song).all()` API happens to auto-deduplicate by primary key in this SQLAlchemy version, which is why the current tests pass — but the query is doing the wrong thing under the hood and would produce duplicates the moment anyone migrated to `session.execute(select(...))`, or relied on the raw result in any other way.

**How I found the root cause.** Function-trace: `GET /songs/search?q=…` → `routes/songs.py::search` → `search_service.search_songs`. Inside `search_songs`, the query is a single expression:

```python
db.session.query(Song)
  .outerjoin(song_tags, Song.id == song_tags.c.song_id)
  .filter(db.or_(Song.title.ilike(...), Song.artist.ilike(...)))
  .all()
```

I read the query line by line. The `outerjoin(song_tags, …)` is never referenced again — no filter uses it, no column from `song_tags` is selected, and the `Tag` and `song_tags` imports at the top of the file are otherwise unused. So the join has no functional role; its only observable effect is to fan the row count out to one row per tag per matching song. That's the mechanism that produces the duplicates the issue describes.

**The root cause.** The `.outerjoin(song_tags, Song.id == song_tags.c.song_id)` clause causes the SQL engine to emit one row per song–tag pair. For a song with three tags, the raw result has three identical `song.*` rows; for a song with no tags, one row. The join is dead code — it serves no filter and contributes no data to the SELECT — but it multiplies rows. The legacy ORM Query API masks this at `.all()` time via identity-map deduplication, but the underlying SQL is still returning duplicates.

**Fix and side-effect check.** Removed the `.outerjoin(...)` clause from `services/search_service.py` and dropped the now-unused `Tag, song_tags` imports. The query now selects songs matching title or artist without any join to `song_tags`. Ran `pytest tests/test_search.py` — all 5 tests still pass, including `test_search_no_duplicates_multi_tag_song`. Song tags are still populated in the response because `Song.tags` uses `lazy="subquery"`, which loads tags in a separate SELECT after the main query — that mechanism is untouched by removing the join. No other function calls the join or the imports I removed.

### Bug 4 — I got notified when a friend added my song to a playlist but not when they rated it

**How I reproduced it.** Traced the two endpoints against seed data mentally: `POST /playlists/<id>/songs` calls `add_to_playlist`, which does a `create_notification(...)` before returning. `POST /songs/<id>/rate` calls `rate_song`, which upserts the `Rating` and returns — with no notification call anywhere. The `seed_data.py` file confirms the pattern: line 168 creates a "song_added_to_playlist" notification as a working example, and the file has no equivalent for "song_rated" — because the code path never produces one.

**How I found the root cause.** The hint in the project brief said the root cause is architectural, not a typo, and instructed comparing the working notification pattern line-by-line against the missing one. Both functions live in the same file (`services/notification_service.py`), so I opened `add_to_playlist` and `rate_song` side by side. In `add_to_playlist`, right after committing the playlist mutation:

```python
if song.shared_by != added_by_user_id:
    create_notification(
        user_id=song.shared_by,
        notification_type="song_added_to_playlist",
        body=f"{adder.username} added your song '{song.title}' to the playlist '{playlist.name}'.",
    )
```

In `rate_song`, the corresponding position — right after `db.session.commit()` and before `return rating` — is empty. The commit happens, the rating is persisted, but nothing addresses the song's original sharer. That's the whole architectural miss: the entire notification block from the sibling function was never written.

**The root cause.** `rate_song` in `services/notification_service.py` never calls `create_notification`. The service is named after notifications and owns the paired write for playlist adds, but the rate flow only writes the `Rating` row. There is no typo, no wrong argument, no bad condition — the call site simply doesn't exist. So `song.shared_by` is never told anyone rated their song.

**Fix and side-effect check.** After `db.session.commit()` and before `return rating`, added the same guard-and-notify pattern used by `add_to_playlist`:

```python
if song.shared_by != user_id:
    create_notification(
        user_id=song.shared_by,
        notification_type="song_rated",
        body=f"{rater.username} rated your song '{song.title}' {score}/5.",
    )
```

The `song.shared_by != user_id` guard prevents self-notification, matching the sibling pattern. The `notification_type="song_rated"` string is new — no existing code compares against it, so it can be freely introduced (verified with `grep -rn "song_rated" .`). Side-effect check: ran the full test suite. `test_streaks.py` (5) and `test_search.py` (5) still pass. `test_playlists.py` shows two pre-existing failures from Bug 5, unrelated to this change. The `create_notification` helper is idempotent w.r.t. errors — it commits its own row inside a separate transaction step, so a notification failure would not roll back the rating write.

## Codebase Map

Mixtape is a Flask + SQLAlchemy backend for a social music app. Users share songs, rate them, build collaborative playlists, and see what their friends are listening to. Every route is JSON in / JSON out — there is no HTML/template layer.

### Top-level files

- **`app.py`** — Flask application factory (`create_app`). Constructs the `SQLAlchemy` instance (`db`), reads `DATABASE_URL` / `SECRET_KEY` from the environment (falling back to a local SQLite file `mixtape.db` and a dev secret), registers the four blueprints under URL prefixes (`/songs`, `/playlists`, `/users`, `/feed`), and calls `db.create_all()` inside an app context. The `db` object is imported by both `models.py` and every service — so the app *must* be started via `FLASK_APP=app:create_app flask run`, not by running `app.py` directly, or the module gets imported twice and SQLAlchemy complains about duplicate table definitions.
- **`models.py`** — All SQLAlchemy models plus three association tables (`friendships`, `song_tags`, `playlist_entries`). Every primary key is a UUID string generated by `generate_uuid()`; every timestamp is a UTC-aware `datetime`. Each model has a `to_dict()` that produces the JSON shape the routes return.
- **`seed_data.py`** — Populates the SQLite DB with fixture users, songs, playlists, and listening events. Meant to be run once after setup.
- **`requirements.txt`** — Flask ≥3.0, flask-sqlalchemy ≥3.1, sqlalchemy ≥2.0, python-dotenv, pytest.

### The data model at a glance

Seven entities:

| Model | Purpose |
|---|---|
| `User` | Account + `listening_streak` and `last_listened_at` columns (streaks are stored on the user, not derived on read). Symmetric self-join `friends` relationship via the `friendships` table. |
| `Song` | Title, artist, album, genre, `shared_by` (the user who introduced the song), `shared_at`, optional `share_note`. Tags are many-to-many via `song_tags`. |
| `Tag` | Simple named label, joined to songs through `song_tags`. |
| `ListeningEvent` | Immutable log of `(user_id, song_id, listened_at)`. Both the streak logic and the friends-listening-now feed read from this table. |
| `Rating` | `(user_id, song_id, score 1–5)` with a `UniqueConstraint` on `(user_id, song_id)` — the DB itself enforces "one rating per user per song," and `rate_song` in the notification service upserts against it. |
| `Playlist` | Name, creator, `is_collaborative` flag (defaults `True`). Songs are attached via `playlist_entries`, not a direct FK. |
| `Notification` | `(user_id, notification_type, body, read)`. The user_id is the *recipient*, and `notification_type` is a free-form string like `"song_added_to_playlist"` — there's no enum. |

The `playlist_entries` association table is the one to notice: it carries `position` (int) and `added_by` (user FK) columns *on the join row itself*. So song order in a playlist is explicit — it's not insertion order and not alphabetical, it's whatever `position` says. This will matter for playlist bugs.

### Routes layer (`routes/`)

Four blueprints, one file each. Every handler follows the same shape: pull fields off the JSON body or path, delegate to a service function, wrap the return value in `jsonify`, translate `ValueError` from the service into a 400 or 404. No business logic in the routes at all.

- **`routes/songs.py`** — `GET /songs/search?q=…`, `GET /songs/<id>`, `POST /songs/<id>/rate`, `POST /songs/<id>/listen`. Delegates to `search_service`, `notification_service.rate_song`, and `streak_service.record_listening_event`.
- **`routes/playlists.py`** — `POST /playlists/`, `GET /playlists/<id>`, `GET /playlists/<id>/songs`, `POST /playlists/<id>/songs`. Interestingly, adding a song to a playlist goes through `notification_service.add_to_playlist` (not `playlist_service`) — because the *side effect* is a notification, not a playlist mutation, from that service's point of view.
- **`routes/users.py`** — `GET /users/<id>`, `GET /users/<id>/streak`, `GET /users/<id>/notifications?unread_only=…`, `POST /users/notifications/<notification_id>/read`.
- **`routes/feed.py`** — `GET /feed/<user_id>/listening-now`, `GET /feed/<user_id>/activity`.

There is no route mapped to `/` — hitting the base URL returns 404. That's expected; it just means the server is up.

### Services layer (`services/`)

The README states plainly that "the bugs live in the `services/` layer" — this is where all logic sits.

- **`services/streak_service.py`** — `record_listening_event` inserts a `ListeningEvent` and then calls `update_listening_streak`, which mutates the user's `listening_streak` and `last_listened_at` based on the day gap since the last listen. `get_streak` just reads the stored value.
- **`services/feed_service.py`** — `get_friends_listening_now` returns each friend's most recent song within the last 24 hours (`RECENT_THRESHOLD = timedelta(hours=24)`), deduplicated to one row per friend. `get_activity_feed` returns the latest N events across all friends with no time filter.
- **`services/search_service.py`** — `search_songs` does a case-insensitive `ILIKE` match on title or artist. It `outerjoin`s `song_tags` in the query (though only songs are selected). `get_song` is a plain lookup by ID.
- **`services/notification_service.py`** — Owns both the notification records *and* the write actions that produce them: `add_to_playlist` mutates the playlist and then creates a `"song_added_to_playlist"` notification for the song's original sharer; `rate_song` upserts a `Rating`. `create_notification`, `get_notifications`, `mark_as_read` round it out.
- **`services/playlist_service.py`** — `create_playlist`, `get_playlist_songs` (ordered by `playlist_entries.position`), `get_playlist`, `get_user_playlists`.

### Tests

`tests/test_streaks.py`, `tests/test_search.py`, `tests/test_playlists.py`. One file per bug-carrying service that has verifiable output. Run with `pytest tests/`.

---

## Data flow — "friend adds one of my songs to a playlist"

This is the flow the notification system is built around, so it's worth walking end-to-end.

1. Client sends `POST /playlists/<playlist_id>/songs` with body `{"song_id": ..., "added_by": ...}`.
2. `routes/playlists.py::add_song` pulls `song_id` and `added_by` from the JSON, validates they're both present (400 if not), and calls `notification_service.add_to_playlist(playlist_id, song_id, added_by)`.
3. `notification_service.add_to_playlist`:
   - Loads the `Song`, the adder `User`, and the `Playlist`, each raising `ValueError` if missing (which the route turns into a 400).
   - If the song isn't already in `playlist.songs`, appends it and commits. This write goes through the `playlist_entries` association table.
   - Compares `song.shared_by` to `added_by_user_id`. If they differ (a friend added *your* song, not their own), it calls `create_notification` with `notification_type="song_added_to_playlist"`, addressed to `song.shared_by`.
4. `create_notification` inserts a `Notification` row and commits.
5. The route returns `{"message": "Song added to playlist"}` with a 201.
6. Later, the original sharer hits `GET /users/<their_id>/notifications` → `routes/users.py::notifications` → `notification_service.get_notifications` → returns notifications ordered `desc(created_at)`. The message body reads: `"<adder> added your song '<title>' to the playlist '<name>'."`

Two things to note from this trace. First, the notification service is doing double duty — it owns both the playlist write and the notification write, because the notification is the reason the write matters. Second, the check "did the user add their own song?" happens by comparing the `Song.shared_by` FK to the `added_by` argument, so you never get notified for your own action. That mirrors the same guard you'd expect on the rating flow.

---

## Data flow — "user listens to a song"

Shorter, but touches the streak system:

1. `POST /songs/<song_id>/listen` with `{"user_id": ...}` → `routes/songs.py::listen`.
2. `streak_service.record_listening_event(user_id, song_id)`:
   - Creates a `ListeningEvent(user_id, song_id, listened_at=now)`.
   - Calls `update_listening_streak(user, now)`, which reads `user.last_listened_at`, computes `days_since_last`, and mutates `user.listening_streak` in place — set to 1 on first listen or gap, incremented on a consecutive day, unchanged if the user already listened today.
   - Commits.
3. The route returns `event.to_dict()` (201).

The streak is stored *on the User row* and updated on every listen, not recomputed from the events table on read. So `get_streak` is a cheap column read, but any bug in the update path silently corrupts the stored value until a "correct" listen overwrites it.

---

## Patterns

- **Routes are dumb, services do everything.** Every route function is essentially: parse → call one service function → jsonify → translate `ValueError` to HTTP. No queries, no computation, no cross-service coordination in the routes.
- **Services own writes for the *effect* they produce, not the *entity* they touch.** `notification_service.add_to_playlist` writes to the playlist because the interesting outcome is the notification. `streak_service.record_listening_event` writes the listening event because the interesting outcome is the streak update. This is why "adding a song to a playlist" lives outside `playlist_service`.
- **Errors flow as `ValueError` from services and turn into 4xx at the route.** No custom exception hierarchy — services just raise `ValueError(...)` with a human-readable message, and the route decides 400 vs 404 based on context.
- **`to_dict()` on each model is the JSON contract.** Nothing else formats responses; routes call `jsonify(model.to_dict())` or `jsonify([m.to_dict() for m in ms])`.
- **Timestamps are UTC-aware.** Every default uses `datetime.now(timezone.utc)`, and code that reads `last_listened_at` re-attaches `timezone.utc` when the DB round-trip returns a naive datetime — a hint that mixing naive and aware datetimes was a concern here.
- **State lives in the DB, not in memory.** The streak counter is a column on `User`, not a derived value; the "did the user rate this song already?" check is enforced by a `UniqueConstraint`, not application logic. Reads are cheap; writes are the interesting surface.
- **App factory + blueprint pattern.** `create_app` builds the app, initializes the DB, registers blueprints, and calls `db.create_all()` — no top-level app instance, which is why the `flask run` command specifies `app:create_app`.
