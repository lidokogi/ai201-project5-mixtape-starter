# Mixtape Bug Hunt — Submission

## AI Usage

I used an AI tool to help with codebase orientation after I had already read through the service files myself. Once I had a mental model of how the app was structured — routes delegating to services, the five service files each owning a distinct responsibility — I pasted the service files and asked the AI to explain what specific functions were doing, particularly the streak logic and the playlist query. This helped me confirm what I was already seeing rather than replace my own reading.

For Bug 1, I noticed the `weekday()` condition in the elif looked wrong while reading the streak logic, then asked the AI to explain the difference between `weekday()` and `isoweekday()` to confirm my understanding before making the fix.

For Bug 5, I spotted `songs[:-1]` immediately while reading `get_playlist_songs()` and recognized it as an off-by-one — I didn't need AI assistance for that one.

For Bug 4, I found the missing notification by comparing `add_to_playlist()` and `rate_song()` side by side myself, noticed `rate_song()` had no `create_notification()` call, and used the existing `add_to_playlist()` implementation as a reference pattern to write the fix.

---

## Codebase Map

**Main files:**
- `app.py` — Flask app factory, initializes the database and registers route blueprints
- `models.py` — SQLAlchemy models: User, Song, Playlist, ListeningEvent, Notification, Rating, and two join tables (song_tags, playlist_entries)
- `routes/songs.py` — endpoints for sharing, searching, and rating songs
- `routes/playlists.py` — endpoints for creating playlists and adding songs
- `routes/users.py` — endpoints for user profiles, streaks, and notifications
- `routes/feed.py` — endpoints for friends listening now and activity feed
- `services/streak_service.py` — listening streak calculation logic
- `services/feed_service.py` — friends feed and activity feed queries
- `services/search_service.py` — song search by title and artist
- `services/notification_service.py` — notification creation, rating, and playlist-add logic
- `services/playlist_service.py` — playlist creation and song retrieval

**Pattern:** Every route delegates immediately to a service function. Routes handle input parsing and response formatting; all business logic lives in services/.

**Data flow — user rates a song:**
`POST /songs/<song_id>/rate` in `routes/songs.py` calls `notification_service.rate_song(user_id, song_id, score)`. That function validates the score, looks up the song and user, creates or updates a Rating record, commits to the database, and (after the fix) creates a Notification for the song's original sharer.

---

## Bug 1 — Listening Streak Keeps Resetting

**Issue:** #1 — My listening streak keeps resetting

**How I reproduced it:** Checked the streak logic in `streak_service.py` and identified that listening on a Sunday would hit the reset branch instead of the increment branch, because of an extra condition on the elif.

**How I found the root cause:** Read `update_listening_streak()` line by line. The elif that increments the streak read `days_since_last == 1 and today.weekday() != 6`. Python's `weekday()` returns 6 for Sunday, so any listening event on a Sunday where the user had also listened the day before (Saturday) would fail the condition and fall through to the else branch, resetting the streak to 1.

**Root cause:** The condition `today.weekday() != 6` was added incorrectly. A Sunday listen after a Saturday listen is a valid consecutive day — the streak should increment. There is no reason to treat Sunday as a week boundary in this logic.

**Fix and side-effect check:** Removed the `and today.weekday() != 6` clause, leaving just `elif days_since_last == 1:`. Checked the other branches — the `days_since_last == 0` (already listened today) and else (gap of more than one day) cases are unaffected.

---

## Bug 5 — Last Song in Playlist Never Shows Up

**Issue:** #5 — The last song in a playlist never shows up

**How I reproduced it:** Read `get_playlist_songs()` in `playlist_service.py` and saw `songs[:-1]` in the return statement, which drops the last element of any list.

**How I found the root cause:** The return line was `return [song.to_dict() for song in songs[:-1]]`. Python slice `[:-1]` returns all elements except the last. This means every playlist query silently drops its final song regardless of playlist length.

**Root cause:** `songs[:-1]` was used instead of `songs`. This appears to be a typo or copy-paste error — there is no reason to exclude the last song. A playlist with one song would return an empty list; a playlist with five songs would return four.

**Fix and side-effect check:** Changed `songs[:-1]` to `songs`. Checked `get_playlist()` and `get_user_playlists()` — neither calls `get_playlist_songs()`, so no other function is affected.

---

## Bug 4 — No Notification When a Song Is Rated

**Issue:** #4 — I got notified when a friend added my song to a playlist but not when they rated it

**How I reproduced it:** Compared `add_to_playlist()` and `rate_song()` in `notification_service.py`. `add_to_playlist()` calls `create_notification()` after saving the data. `rate_song()` commits the rating and returns without ever calling `create_notification()`.

**How I found the root cause:** The issue description said notifications work for playlist adds but not ratings. I looked at both functions side by side. `add_to_playlist()` has an explicit `create_notification()` call after the commit. `rate_song()` has no such call — it just saves the rating and returns. The notification was never implemented for the ratings case.

**Root cause:** `rate_song()` was missing a `create_notification()` call after saving the rating. The infrastructure (the `create_notification` function, the Notification model) all existed — it just was never called from `rate_song()`.

**Fix and side-effect check:** Added a `create_notification()` call after `db.session.commit()` in `rate_song()`, guarded by `if song.shared_by != user_id` to avoid notifying someone that they rated their own song. Checked `create_notification()` — it only creates a DB record and commits, no side effects. Checked `get_notifications()` and `mark_as_read()` — unaffected.

---

## Git Log

*(paste screenshot of `git log --oneline` here before submitting)*