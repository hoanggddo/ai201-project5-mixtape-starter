# Mixtape Bug Hunt — Submission

## AI Usage

I used Claude to help orient myself to the codebase. I uploaded all the source
files and asked it to summarize each file's responsibility, trace two data flows,
and identify patterns in how the app is organized. Claude identified the likely
root causes of all five bugs from reading the code. I verified each diagnosis
by reading the relevant code myself and reproducing the bug before writing any
fix. For Issue #4, Claude walked me through the reproduction steps — identifying
which user IDs to use, constructing the curl command, and confirming the bug
by checking the notifications endpoint. I verified the fix worked by re-seeding
and re-running the same curl command after the change.

---

## Codebase Map

### App Structure

The app follows a strict route → service pattern. Every blueprint in `routes/`
handles only input parsing and JSON formatting. All business logic lives in
`services/`. There are no exceptions to this pattern.

### Main Files and Their Roles

**`app.py`** — Flask application factory. Creates the app, initializes
SQLAlchemy, and registers the four blueprints (`songs`, `playlists`, `users`,
`feed`). The DB is SQLite by default. Must be started with
`FLASK_APP=app:create_app flask run` — not `python app.py`.

**`models.py`** — Defines 7 SQLAlchemy models:
- `User` — has `listening_streak` (int) and `last_listened_at` (datetime).
  Friends are a self-referential many-to-many via the `friendships` table.
- `Song` — belongs to a User (`shared_by`). Tags attached via `song_tags`
  (many-to-many).
- `Tag` — just an id and name string.
- `ListeningEvent` — records each listen with a timestamp.
- `Rating` — a 1–5 score per (user, song) pair. Unique constraint enforced
  at the DB level.
- `Playlist` — songs attached via `playlist_entries`, which adds `position`,
  `added_by`, and `added_at` columns to the join table.
- `Notification` — a message for a user with a `notification_type`, `body`,
  and `read` boolean.

**`routes/songs.py`** — Handles song search (`GET /songs/search`), song detail
(`GET /songs/<id>`), rating (`POST /songs/<id>/rate`), and listening events
(`POST /songs/<id>/listen`).

**`routes/playlists.py`** — Handles playlist creation, detail, and song
management. Adding a song to a playlist goes through `notification_service`,
not `playlist_service`, because it also creates a notification.

**`routes/users.py`** — User profile, streak, notifications, and mark-as-read.

**`routes/feed.py`** — Two feed endpoints: friends listening now
(`GET /feed/<id>/listening-now`) and general activity feed
(`GET /feed/<id>/activity`).

**`services/streak_service.py`** — Streak increment/reset logic. The core
function is `update_listening_streak(user, now)`, which compares today's date
to `user.last_listened_at`.

**`services/feed_service.py`** — Queries `ListeningEvent` for friends within
a recency window. Deduplicates to show only each friend's most recent song.

**`services/search_service.py`** — Queries `Song` with an `outerjoin` on
`song_tags` to support tag-aware search. Returns `to_dict()` results.

**`services/notification_service.py`** — Creates notifications via
`create_notification()`. Has two action functions: `add_to_playlist()` and
`rate_song()`. Also handles retrieval and mark-as-read.

**`services/playlist_service.py`** — Creates playlists and retrieves songs
ordered by `position` from `playlist_entries`.

---

### Data Flow: User Rates a Song

1. Client sends `POST /songs/<song_id>/rate` with `user_id` and `score`
2. `routes/songs.py` parses the body and calls
   `notification_service.rate_song(user_id, song_id, score)`
3. `rate_song()` validates the score (1–5), looks up the Song and User,
   checks for an existing Rating, creates or updates it, and commits
4. After the fix: a notification is created for the song's original sharer
   if the rater is a different user

### Data Flow: User Listens to a Song

1. Client sends `POST /songs/<id>/listen` with `user_id`
2. `routes/songs.py` calls
   `streak_service.record_listening_event(user_id, song_id)`
3. A `ListeningEvent` is created with the current UTC timestamp
4. `update_listening_streak(user, now)` is called — compares today's date to
   `user.last_listened_at.date()` and increments, skips, or resets the streak
5. Both changes are committed together

### Pattern I Noticed

The `notification_service` is unusual — it's not just about notifications.
`add_to_playlist()` and `rate_song()` both live there because those actions
are the ones that trigger notifications. The service combines the write action
with the notification side effect. This is why the rating notification is easy
to miss — `rate_song()` feels complete once it saves the Rating, but the
notification call is supposed to follow.

---

## Root Cause Analyses

### Issue #4 — I got notified when a friend added my song to a playlist but not when they rated it

**How I reproduced it**

Searched `/songs/search?q=Crown` to get the song ID
(`275171e2-2011-4eed-9650-38400270f67c`) and its sharer simone
(`6971ccca-f13a-4bf4-b5cd-503776accfdf`). Used the flask shell to list all
users and picked nova (`cfa4a0e8-a55e-45d3-bdd0-6afc931abd72`) as the rater.
POSTed to `/songs/<id>/rate` with nova's user_id and score 5 — the rating saved
successfully. Immediately checked simone's notifications at
`/users/<id>/notifications` and received `{"count": 0, "notifications": []}`.
The rating was saved but no notification was created.

**How I found the root cause**

Traced the call chain from the route: `POST /songs/<id>/rate` →
`routes/songs.py` → `notification_service.rate_song()`. Read `rate_song()`
in full and saw it validates the score, looks up the song and user, creates or
updates the Rating, and commits. Then it returns the rating. Compared it
line-by-line to `add_to_playlist()` directly above it in the same file, which
follows the same pattern but includes a `create_notification()` call after the
commit. The notification call was simply absent from `rate_song()`.

**The root cause**

`rate_song()` in `notification_service.py` saves the rating and commits
successfully, but never calls `create_notification()`. The `add_to_playlist()`
function in the same file shows the intended pattern — write the data, commit,
then notify the original sharer if the acting user is different. That final
step was missing entirely from `rate_song()`.

**The fix and side-effect check**

Added the following block to `rate_song()` between the commit and the return:

```python
if song.shared_by != user_id:
    create_notification(
        user_id=song.shared_by,
        notification_type="song_rated",
        body=f"{rater.username} rated your song '{song.title}' {score}/5.",
    )
```

The `if song.shared_by != user_id` guard ensures a user rating their own song
does not generate a notification. After the fix, re-seeded the database and
repeated the exact reproduction steps — simone's notifications endpoint
returned the expected notification. Checked that the rating itself still saves
correctly and that rating your own song produces no notification.

---

### Issue #1 — My listening streak keeps resetting

*(To be filled in after fix)*

---

### Issue #5 — The last song in a playlist never shows up

*(To be filled in after fix)*

---