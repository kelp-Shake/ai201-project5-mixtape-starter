# Mixtape Bug Hunt — Submission

> Branch: `bugfix/mixtape`
> Repo (branch URL): `https://github.com/<you>/ai201-project5-mixtape-starter/tree/bugfix/mixtape`

---

## AI Usage

I used AI mainly to navigate the codebase and to pressure-test my debugging, not to write the fixes for me.

- **Codebase orientation:** I had AI explain what each file was responsible for (app.py's factory pattern, the models and their relationships, how routes delegate to services) so my codebase map was accurate. I checked its descriptions against the actual files and rewrote them in my own words.
- **Data flow tracing:** I asked it to help trace the "friends listening now" feed from the route down to feed_service, then verified the call chain and the 24h cutoff myself in the code.
- **Issue #3 (search duplicates):** This is where AI was most useful. My test kept returning the song once instead of three times, and AI helped me figure out that SQLAlchemy's legacy `query()` API auto-dedupes full entities, which is why the join wasn't showing duplicates. I confirmed this myself by rewriting the query with `select()` and seeing the song come back 3 times (one per tag). I also verified the tags still came from `to_dict()` and not the join before removing it.
- **Where I verified/overrode:** I didn't take diagnoses on faith. For each bug I reproduced the behavior and read the code myself before committing. The AI was helpful for explaining *why* something behaved a certain way, but I confirmed every root cause and fix by running the tests and inspecting the data.

---

## Codebase Map

<!--
Written BEFORE any bug work (Milestone 1).
Cover at minimum:
- The main files and what each one does
- The data flow for at least one feature (trace the actual function call chain)
- Any patterns you notice in how the app is organized
Name responsibilities, don't just list file names.
-->

### Main files and their roles

| File | Responsibility |
| --- | --- |
| `app.py` | Flask app factory and database setup. `create_app()` creates the flask app, sets up the default configuration sql alchemy database then it is initialized as `db`.  models and services where the `/songs`, `/playlists`, `/user`, and `/feed` routers import the database. |
| `models.py` | SQLAlchemy models that define the tables and their relationships:<br>• **User** stores user fields (username, email, `listening_streak`, `last_listened_at`) and connects to other tables via relationships<br>• **Song** stores song metadata (title, artist, album, genre) plus who shared it<br>• **ListeningEvent** associates a song playback with a time, user, and id<br>• **Rating** holds a user-supplied rating for each song<br>• **Playlist** groups songs via the `playlist_entries` join table, which stores each song's `position` (explicit ordering, not insertion order)<br>• **Tag** is a label given to individual songs that are grouped by the tag<br>• **Notification** is a message that is delivered to the user |
| `routes/` | The endpoint files (`songs.py`, `playlists.py`, `users.py`, `feed.py`) that each define a blueprint. They read the request, pull out the inputs, then hand off to a service function and return the result as json. Not much logic lives here, they mostly parse and respond. |
| `services/` | Where the actual logic lives. Each file handles one area: `search_service` searches and fetches songs, `notification_service` rates songs and creates/gets notifications, `streak_service` records listening events and updates streaks, `playlist_service` handles playlist songs, and `feed_service` builds the listening now feed. The routes call into these. |
| `seed_data.py` | Fills the database with test data so the app has something to work with. It creates 5 users with friendships, 25 songs with different tag counts, 3 playlists, listening events over the past 2 weeks, some existing streaks, and playlist-add notifications. Run with `python seed_data.py`. |

### Data flow: "user views listening now feed"
1. Request GET /feed/user_id/listening-now
2. In `routes/feed.py` feed_service.get_friends_listening_now() is called
3. A look up of the user friends listening_events is then filtered to events within the cutoff period ordered by most recent
4. The result is a list of each friend and the song they most recently listened to
   
### Patterns I noticed
- routers and services focus on one area of the service which allows for better tracing
- the models tables ids are created with uuid and not incremental integers, making rows uniquely identifiable between all tables 
---

## Root Cause Analysis

<!--
One entry per bug you fix (at least 3). Each entry must have ALL 5 fields.
Duplicate the block below for each bug. Write it BEFORE moving to the next bug.
-->

### Bug #1 — Streak Reset
**1. Issue number and title**
Issue # 1: Listening Streak Reset

<!-- e.g. Issue #1 — My listening streak keeps resetting -->

**2. How you reproduced it**
Ran the test_streaks.py test and noticed the streak reset error on test_streak_increments_on_sunday()




<!-- Exact steps, inputs, sequence, or data condition that triggered the behavior before you touched any code. -->

**3. How you found the root cause**
I looked at the streak_service.py and focused on the update listening streak function
I traced through function and noticed an error on the if-else clause 
The streak only updates on monday through saturday. so it will enter the else clause the resets the counter on sundays 

<!-- Which files you looked at, your navigation path, and the moment you were confident you'd found the specific cause (not just a suspicious area). -->

**4. The root cause**

<!-- Plain English. Name the exact function, comparison, condition, or missing step. Explain what the code assumed vs. what was actually true. -->
condition checking if the current day is sunday. this condition is not apart of streak counter requirements and needs to be removed 

**5. Your fix and side-effect check**
+ changed the line ```elif days_since_last ==1 and today.weekday != 6``` 
to this ```elif days_since_last == 1:```
+ reran the streak tests and passed all of them. Streak updates for weekdays also work. The streak does not update more than once a day. Missing days or other gaps will now fall to the else clause resting the streak.



<!-- What you changed and why it fixes the root cause. What related functionality you checked to confirm you didn't break anything (test both sides of any boundary). -->

**Commit:** `<commit hash / message>`

---

### Bug # 3 — Repeated song in search

**1. Issue number and title**
Issue #3: Same song repeated in search

**2. How you reproduced it**
Tried running the search tests and they all passed
Looked into the query line and noticed .query auto dedupes duplicated results 
so the bug doesn't show duplicated results 
I did a select statement and bug is able to be reproduced 

**3. How you found the root cause**
The bug would be cause by tags being joined but isn't used for flittering so the tag join creates a new row for each tag.  

**4. The root cause**
outer join of song tags with search songs creating extra lines remove the join as the tag of a song is not needed for search  song tags can be found with search.tags relationship directly

**5. Your fix and side-effect check**
commented out the outerjoin the filter still works and tags and songs can be still be associated 

**Commit:** `<commit hash / message>`

---

### Bug #5 — last song in playlist missing

**1. Issue number and title**
Issue #5: Playlist missing last song
**2. How you reproduced it**
ran the playlist tests and 2/3 failed excluding empty playlist tests 
so I knew that playlist are correctly being initialized and appended to

**3. How you found the root cause**
I looked at how the playlist songs were queried and filtered and noticed no issues. I then moved down to the return and noticed the songs list slice.

**4. The root cause**
get_playlist_songs returns songs[:-1], which drops the final element. Because the query orders by position ascending, the final element is the most recently added song, so the newest song is always missing.

**5. Your fix and side-effect check**
Removed the slice and returned the whole songs list. The songs were always stored correctly, only the return was slicing one off. Verified a 7-song playlist now returns 7 in the right order, and the empty-playlist case still returns [].

**Commit:** `<commit hash / message>`

---

## Stretch (optional)

- [ ] 4th bug fixed (add another RCA entry above)
- [ ] All 5 bugs fixed
- [ ] Regression test written — file: `<path>`, references bug #<n>

---

## Git Log Screenshot

<!-- Milestone 4: paste/embed screenshot of `git log --oneline` on bugfix/mixtape showing one commit per fix. -->
