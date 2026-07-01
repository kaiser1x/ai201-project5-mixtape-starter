# Mixtape – Submission

## 1. Main Files

### `app.py`
Flask application factory (`create_app()`). Initializes SQLAlchemy, registers four blueprints with URL prefixes: `songs_bp` (`/songs`), `playlists_bp` (`/playlists`), `users_bp` (`/users`), `feed_bp` (`/feed`). Database URL comes from `DATABASE_URL` env var with SQLite fallback.

### `models.py`
Defines seven SQLAlchemy models and three association tables:

- **User** — `id`, `username`, `email`, `listening_streak` (int, default 0), `last_listened_at` (datetime); has relationships to shared songs, ratings, listening events, notifications, playlists, and friends (many-to-many via `friendships` table)
- **Song** — `id`, `title`, `artist`, `album`, `genre`, `shared_by` (FK → User), `shared_at`, `share_note`; many-to-many with Tag via `song_tags`
- **ListeningEvent** — `id`, `user_id`, `song_id`, `listened_at` (DateTime UTC); created each time a user listens to a song
- **Rating** — `id`, `user_id`, `song_id`, `score` (1–5), `rated_at`; unique constraint on (user_id, song_id)
- **Playlist** — `id`, `name`, `created_by` (FK → User), `is_collaborative` (bool); songs linked via `playlist_entries` association table which stores `position`, `added_by`, `added_at`
- **Notification** — `id`, `user_id`, `notification_type`, `body`, `created_at`, `read` (bool)
- **Tag** — `id`, `name` (unique)

### `routes/`
HTTP layer only. Each route handler parses the request, delegates immediately to a service function, and returns a JSON response. No business logic lives here.

| File | Endpoints |
|------|-----------|
| `songs.py` | `GET /songs/search`, `GET /songs/<id>`, `POST /songs/<id>/rate`, `POST /songs/<id>/listen` |
| `playlists.py` | `POST /playlists/`, `GET /playlists/<id>`, `GET /playlists/<id>/songs`, `POST /playlists/<id>/songs` |
| `users.py` | `GET /users/<id>`, `GET /users/<id>/streak`, `GET /users/<id>/notifications`, `POST /users/notifications/<id>/read` |
| `feed.py` | `GET /feed/<id>/listening-now`, `GET /feed/<id>/activity` |

### `services/`
All business logic lives here.

- **`streak_service.py`** — Records `ListeningEvent`, calls `update_listening_streak()` to increment/reset `User.listening_streak` based on consecutive calendar days; exposes `get_streak()`.
- **`notification_service.py`** — Creates notifications, handles `add_to_playlist()` (creates notification to song sharer), `rate_song()` (updates Rating record), `get_notifications()`, `mark_as_read()`.
- **`playlist_service.py`** — Creates playlists, retrieves playlist songs ordered by `position` ASC.
- **`search_service.py`** — Queries Song with OUTER JOIN on `song_tags`, filters title/artist case-insensitively.
- **`feed_service.py`** — `get_friends_listening_now()` returns deduplicated friend events within last 24 hours; `get_activity_feed()` returns last 20 friend events.

### `seed_data.py`
Populates the database with 5 users, 25 songs, 3 playlists, listening events, tags, and one sample notification for testing.

---

## 2. Data Flow

### Feature: User Listens to a Song

```
POST /songs/<song_id>/listen  { "user_id": "..." }
  ↓
routes/songs.py → listen()
  ↓
streak_service.record_listening_event(user_id, song_id)
  │  1. Load User from DB
  │  2. Create ListeningEvent(user_id, song_id, listened_at=now)
  │  3. Call update_listening_streak(user, now)
  │       - today = now.date()
  │       - If no prior listen: streak = 1
  │       - If same day: no change
  │       - If previous day: streak += 1
  │       - If gap > 1 day: streak = 1
  │       - Always update user.last_listened_at
  │  4. db.session.commit()
  ↓
Return ListeningEvent.to_dict()  →  200 JSON
```

### Feature: Add Song to Playlist (with Notification)

```
POST /playlists/<playlist_id>/songs  { "song_id": "...", "added_by": "..." }
  ↓
routes/playlists.py → add_song()
  ↓
notification_service.add_to_playlist(playlist_id, song_id, added_by_user_id)
  │  1. Load Song, User (adder), Playlist from DB
  │  2. Append song to playlist.songs (if not already present)
  │  3. db.session.commit()
  │  4. If song.shared_by != added_by_user_id:
  │       create_notification(
  │         user_id=song.shared_by,
  │         notification_type="song_added_to_playlist",
  │         body=f"{adder.username} added your song '{song.title}' to '{playlist.name}'."
  │       )
  ↓
Return 201 { "message": "Song added to playlist" }
```

---

## 3. Organizational Patterns

**Routes delegate immediately.** Route handlers do one job: parse the request and call one service function. All branching, DB access, and notification logic lives in services.

**Services own business logic.** Streak logic, notification creation, playlist ordering — all in `services/`. Routes never touch `db.session` directly (except one `users.py` profile lookup).

**Notification creation is co-located with the triggering action.** `add_to_playlist()` in `notification_service.py` both modifies the playlist AND creates the notification in the same function. `rate_song()` should follow the same pattern but currently does not.

**Streaks are stored incrementally.** `User.listening_streak` is updated on every `record_listening_event()` call rather than recomputed from history. This means the stored value can drift if the update logic is wrong.

**Many-to-many with metadata.** `playlist_entries` stores `position`, `added_by`, and `added_at` alongside the join keys — a join table with payload, not just a pure pivot.

---

## 4. Issues

### Issue 1 — Listening streak resets on Sunday (`streak_service.py:73`)
`update_listening_streak()` has condition `today.weekday() != 6` before incrementing the streak. `weekday() == 6` is Sunday, so this condition *blocks* incrementing on Sunday and falls through to the reset branch. A Saturday → Sunday consecutive listen resets the streak to 1 instead of incrementing it. Fix: remove the weekday condition entirely.

### Issue 2 — Friends Listening Now shows stale activity (`feed_service.py:13`)
`RECENT_THRESHOLD = timedelta(hours=24)` is too wide. The feature is meant to show who is listening *right now*, but 24 hours includes events from yesterday. Reduce threshold to a smaller window (e.g., 30 minutes or a few hours).

### Issue 3 — Same song appears multiple times in search (`search_service.py:27`)
`search_songs()` uses `OUTER JOIN` on the `song_tags` association table. A song with N tags produces N joined rows, so it appears N times in results. Fix: add `.distinct()` to the query.

### Issue 4 — No notification when a friend rates a song (`notification_service.py:73`)
`rate_song()` creates/updates the Rating record and commits, but never calls `create_notification()`. The song's original sharer should receive a notification. Fix: add `create_notification()` call after commit, guarded by `song.shared_by != user_id`.

### Issue 5 — Last song in playlist never appears (`playlist_service.py:66`)
`get_playlist_songs()` returns `[song.to_dict() for song in songs[:-1]]`. The `[:-1]` slice removes the last element unconditionally. Fix: change to `songs` (no slice).
