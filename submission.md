# Mixtape Bug Hunt — Submission

## AI Usage

- **Issue #5 investigation:** Claude helped me isolate the suspicious line (`songs[:-1]`) by first having me re-run the underlying query without the final slice, directly in `flask shell`, to confirm the query itself was correct and the bug was isolated to the return statement. Claude also pushed me to test a boundary case I hadn't considered — a playlist with exactly 1 song — which surfaced that the original bug was actually worse than reported (it would return 0 songs, not just drop 1 of several). While setting up that boundary test, I hit an unrelated `IntegrityError` in `add_to_playlist()` (missing `position`/`added_by` when appending via the ORM relationship); Claude identified this as a separate, real bug worth noting but not something to fix as part of Issue #5, so I inserted directly into the `playlist_entries` table instead to isolate the test. I also had briefly self-introduced a stray `from routes import songs` import while poking at the file; Claude caught that this violated the routes→services dependency direction from my own codebase map and had me remove it as part of the same fix.
- **Issue #1 investigation:** Rather than being told the root cause outright, Claude had me manually trace both operands of the `elif days_since_last == 1 and today.weekday() != 6:` condition using the actual reproduction values, and asked me to state what `and` evaluates to when one operand is `False` before confirming the diagnosis. I confirmed `today.weekday() == 6` for Sunday myself in the shell earlier, which made the trace concrete rather than abstract.
- **Issue #2 investigation:** I initially misread the reproduction numbers myself (conflating "4.6 hours ago" with the actual `>= cutoff` comparison the code performs), and Claude had me work through the specific timestamps rather than accepting my framing, which clarified that the two comparisons happened to agree in that instance but weren't the same check. After applying the fix, my first verification attempt showed the fix appearing to fail (an event that should have been excluded still showed up); rather than assume the fix was wrong, Claude had me inspect darius's full event history directly in the database, which revealed leftover events from an earlier manual test and from seed data that were independently contaminating the result. Clearing those out and re-testing in isolation confirmed the fix was correct — a useful reminder that a failed verification doesn't always mean the fix is wrong; sometimes the test itself is contaminated.

---

## Codebase Map

**Main files and their roles:**

- `app.py` — Flask application factory (`create_app`). Registers four blueprints (`songs`, `playlists`, `users`, `feed`), configures SQLAlchemy, and calls `db.create_all()` on startup.
- `models.py` — Defines 7 SQLAlchemy models: `User`, `Tag`, `Song`, `ListeningEvent`, `Rating`, `Playlist`, `Notification`, plus 3 association tables: `friendships` (symmetric many-to-many for friends), `song_tags` (plain many-to-many between songs and tags), and `playlist_entries` (many-to-many between playlists and songs, but with extra columns — `position`, `added_by`, `added_at` — meaning playlist membership is explicitly ordered, not just insertion order).
- `routes/` — Thin controllers, one file per resource area (`songs.py`, `playlists.py`, `users.py`, `feed.py`). Each route parses request args/JSON, calls exactly one service function, and formats the JSON response. No business logic lives here.
- `services/` — All business logic, one file per feature area:
  - `streak_service.py` — listening streak calculation and update rules
  - `feed_service.py` — "Friends Listening Now" and general activity feed logic
  - `search_service.py` — song search by title/artist
  - `notification_service.py` — notification creation and retrieval, plus rating/playlist-add logic
  - `playlist_service.py` — playlist creation and song retrieval
- `seed_data.py` — Populates the DB with 5 users, 25 songs (with varying tag counts), 3 playlists, and a mix of recent/old listening events, specifically designed to expose the reported bugs (e.g., "Crown Heights Anthem" has 3 tags for Issue #3; the "Friday Energy" playlist has exactly 7 songs for Issue #5).
- `tests/` — existing pytest tests for streaks, search, and playlists.

**Pattern noticed:** every route delegates immediately to a single service function — routes only do input parsing and response formatting. This means every bug's real logic lives in `services/`, matching what the README states directly.

**Data flow — user adds a song to a playlist:**

`POST /playlists/<id>/songs` (`routes/playlists.py::add_song`) parses `song_id` and `added_by` from the request body, then calls `notification_service.add_to_playlist(playlist_id, song_id, added_by)`. That function:
1. Looks up the `Song`, the adding `User`, and the `Playlist` (raising `ValueError` → 400 if any are missing).
2. Appends the song to `playlist.songs` (writing into the `playlist_entries` join table) if it isn't already present, and commits.
3. If the person who added the song isn't the original sharer, calls `create_notification()` to notify `song.shared_by` with a message like `"{adder} added your song '{title}' to the playlist '{name}'."`

This is the "working" notification path referenced in Issue #4 — useful as a direct comparison against the rating path, which does not call `create_notification()` at all (see `notification_service.rate_song()`).

---

## Root Cause Analysis

> **Status:** Issues #1, #2, and #5 are fully investigated, fixed, verified, and committed on `bugfix/mixtape`.

### Issue #1 — My listening streak keeps resetting

**How I reproduced it:**
In `flask shell`, set `kenji.listening_streak = 12` and `kenji.last_listened_at` to a fixed Saturday (`2026-07-04 09:00 UTC`). Called `update_listening_streak(kenji, sunday)` with `sunday = 2026-07-05 09:00 UTC` (confirmed `sunday.weekday() == 6`, i.e. Sunday) — one calendar day after the Saturday listen. Expected the streak to increment to 13 (consecutive day). Actual result: `kenji.listening_streak` became `1`.

**How I found the root cause:**
Read `update_listening_streak()` in `services/streak_service.py` and traced the branch handling `days_since_last == 1`:
```python
elif days_since_last == 1 and today.weekday() != 6:
    user.listening_streak += 1
else:
    user.listening_streak = 1
```
Walked through both operands of the `and` using the reproduction values: `days_since_last == 1` was `True`, but `today.weekday() != 6` was `False`, since Python's `weekday()` returns `6` for Sunday (confirmed directly in the shell). Because one operand of the `and` was `False`, the whole condition was `False`, so execution fell through to the `else`, resetting the streak instead of incrementing it. Checked the function's docstring, which documents only "if the user listened yesterday: streak increments by 1" — no day-of-week exception — confirming this wasn't an intentional weekly-reset feature but a genuine bug.

**The root cause:**
The `elif` branch responsible for incrementing the streak on a consecutive-day listen included an extra, unjustified condition, `today.weekday() != 6`, which excludes Sundays specifically. Since `datetime.weekday()` returns `6` for Sunday, any listen that happened on a Sunday — even one immediately following the previous day's listen — failed this check and fell through to the `else` branch, which unconditionally resets the streak to 1. This matches the report exactly: the bug only occurred on Sundays, and the streak resumed counting normally afterward (Monday's listen bumped the reset value from 1 to 2), since Monday isn't excluded by the check.

**My fix and side-effect check:**
Removed the `and today.weekday() != 6` clause, leaving `elif days_since_last == 1:` to increment the streak for any consecutive-day listen regardless of weekday. Verified three cases in `flask shell`: a Sunday listen following a Saturday listen (now correctly increments 12 → 13), a Tuesday listen following a Monday listen (still correctly increments 12 → 13, confirming no regression on non-Sunday days), and a listen after skipping two days (still correctly resets to 1, confirming the reset branch itself is untouched and still works as intended).

**How I reproduced it:**
In `flask shell`, computed `now` (`2026-07-08 02:37:42 UTC`) and `start_of_today` (midnight UTC). Created a `ListeningEvent` for `darius` timestamped `2026-07-07 22:00 UTC` — 2 hours before midnight, i.e. "yesterday evening" by calendar date, but only ~4.6 hours before the current time. Called `get_friends_listening_now(nova.id)` (nova and darius are friends). Expected darius to be excluded, since his listen was from a previous calendar day. Actual result: darius appeared in nova's feed.

**How I found the root cause:**
Read `get_friends_listening_now()` in `services/feed_service.py` and traced the cutoff calculation: `cutoff = datetime.now(timezone.utc) - RECENT_THRESHOLD`, where `RECENT_THRESHOLD = timedelta(hours=24)`. Worked through the actual timestamps from the reproduction: `cutoff` came out to roughly 2:37am the previous calendar day, and darius's event was at 10pm that same calendar day — later in the day than the cutoff timestamp, so `listened_at >= cutoff` was `True` and the event passed the filter. Confirmed there's no separate check anywhere in the function for whether an event falls on today's calendar date; the only recency check is the rolling cutoff.

**The root cause:**
The function's docstring describes returning friends who listened "recently... or at least what they've played today," but the implementation only checks a rolling 24-hour window (`now - 24h`), never a calendar-day boundary. A rolling window and "today" are different things: "today" resets at midnight, but a 24-hour window slides continuously with the current time. An event from 10pm yesterday stays inside a rolling 24-hour window until 10pm *today*, which produces exactly the reported symptom — stale entries persisting until the same time the next day.

**My fix and side-effect check:**
Changed the cutoff calculation from `now - RECENT_THRESHOLD` to `now.replace(hour=0, minute=0, second=0, microsecond=0)` — the start of the current UTC day — so the filter checks calendar-day membership instead of a rolling window. Verified in isolation (after clearing out darius's other listening events, both leftover test data and seed data, to avoid cross-contamination) that an event from 2 hours before midnight is now correctly excluded, while an event from 5 minutes after midnight is correctly included — confirming the fix works on both sides of the midnight boundary. On my first attempt to verify, the fix appeared not to work (an excluded case still showed up); tracing it further showed this was test contamination — darius still had leftover events from an earlier test run and from `seed_data.py`'s own "recent events" batch that independently qualified as "today." Clearing all of darius's events before testing confirmed the fix was correct all along. Checked `get_activity_feed()` in the same file, which explicitly doesn't filter by recency at all and doesn't reference `RECENT_THRESHOLD` or `cutoff`, so it's unaffected by this change.

---

### Issue #5 — The last song in a playlist never shows up

**How I reproduced it:**
In `flask shell`, looked up the "Friday Energy" playlist (seeded with 7 songs) and called `get_playlist_songs(friday_energy.id)`. Expected 7 songs back. Actual result: 6 songs returned, missing "Frequencies" (the last song in seed insertion order / highest `position` value).

**How I found the root cause:**
Traced `routes/playlists.py::get_songs` → `playlist_service.get_playlist_songs()`. Ran the function's underlying query directly in `flask shell` without its final return line — join `playlist_entries`, filter by `playlist_id`, order ascending by `position` — and it correctly returned all 7 songs in the right order. That isolated the bug to the return statement itself: `return [song.to_dict() for song in songs[:-1]]`.

**The root cause:**
The function built the correct, fully-ordered list of songs, but the return statement applied a Python slice, `songs[:-1]`, which unconditionally drops the last element of any list, regardless of playlist size. Since the list is already sorted ascending by `position`, the dropped element is always the song with the highest position — i.e., whichever was added most recently. This explains the exact reported behavior: adding a new song didn't just fail to show that song, it also "freed" the previously-hidden one, because the new song became the new last element being sliced off.

**My fix and side-effect check:**
Changed `songs[:-1]` to `songs`, returning the full list. Verified against the seeded "Friday Energy" playlist (now correctly returns all 7 songs, including "Frequencies"). Also tested a boundary case not covered by the original report — a playlist with exactly 1 song — directly in `flask shell`: before the fix this would have returned 0 songs (an even more severe version of the bug), and after the fix it correctly returns 1. While setting up that boundary test I discovered `add_to_playlist()` in `notification_service.py` raises an `IntegrityError` when appending via `playlist.songs.append(song)`, because the `playlist_entries` table requires `position` and `added_by`, which the ORM relationship's automatic append doesn't populate — I've noted this as a separate bug worth flagging but did not fix it as part of Issue #5, since it's unrelated to the slicing bug and outside this fix's scope. I also removed an unused `from routes import songs` import in the same file, which shadowed the local `songs` variable and violated the routes→services dependency direction described in the codebase map.

## Screenshot: `git log --oneline` on `bugfix/mixtape`

Confirmed on `bugfix/mixtape`:

```
ef7685f (HEAD -> bugfix/mixtape) fix: use calendar-day boundary instead of rolling 24h window for listening-now feed
ee637c9 fix: remove incorrect Sunday exclusion from streak increment logic
6556ac6 fix: return all songs in playlist instead of dropping the last one
2dfdeaa (origin/main, origin/HEAD, main) Add .gitignore file and update README with setup instructions
7b64551 initial commit
```
Put screenshot in the repo along with this file. 