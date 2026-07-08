# Codebase map

    ai201-project5-mixtape-starter/
    ├── app.py                      # Flask app factory and DB setup
    ├── models.py                   # SQLAlchemy models for all entities
    ├── routes/
    │   ├── songs.py                # Song sharing, search, and rating routes
    │   ├── playlists.py            # Playlist creation and song management
    │   ├── users.py                # User profiles, streaks, notifications
    │   └── feed.py                 # Friends listening now, activity feed
    ├── services/
    │   ├── streak_service.py       # Listening streak logic
    │   ├── feed_service.py         # Friends listening now feed logic
    │   ├── search_service.py       # Song search logic
    │   ├── notification_service.py # Notification creation and retrieval
    │   └── playlist_service.py     # Playlist retrieval logic
    ├── tests/
    │   ├── test_streaks.py
    │   ├── test_search.py
    │   └── test_playlists.py
    ├── seed_data.py                # Populates DB with test data
    ├── requirements.txt
    └── .gitignore

# Issue Root Cause Analysis


## Issue #1: "My listening streak keeps resetting"

1. How I reproduced it:  I first Listened to a song every calendar day, including Saturday.
And Listened again Sunday morning and checked my streak (GET /users/<my_id>/streak). 

2. How I found it: Traced the call chain into update_listening_streak() and compared it against the rules in its own docstring, which never mention weekends.

3. Root cause: This line only increments the streak if it's not a Sunday: elif days_since_last == 1 and today.weekday() != 6:
So a user who listens every day still gets their streak reset every Sunday.

4. Fix: Removed the weekday check. Now any consecutive-day listen increments the streak, Sunday included.
Afterwards, I checked afterward that the Same-day repeat listens still no-op, multi-day gaps still reset to 1, and first-time listens still start at 1 — none of that logic was touched.


## Issue #3: "The same song keeps showing up twice in search"

1. How I reproduced it: Searched for a song (GET /songs/search?q=Anthem). Counted the results.

2. How I found it: Looked at the query in search_songs() — it joins Song to song_tags (the tag junction table) but never queries anything about tags in the filter or select. A join without a purpose is a red flag.
3. Root cause:
python.outerjoin(song_tags, Song.id == song_tags.c.song_id)

This joins each song to every row in its tag association table. A song with 2 tags produces 2 duplicate rows in the result set (one per tag), a song with 3 tags produces 3, etc. The tags list on to_dict() presumably pulls from the relationship separately, so the join isn't even needed for that — it's just multiplying rows for no reason.

4. Fix: Drop the join entirely — it isn't used for filtering or selecting anything:
Afterwards, I Confirmed song.to_dict() doesn't rely on the join for producing the tags list — it should be using the Song.tags relationship independently, so removing the join won't break tag display.


## Issue #5 — The last song in a playlist never shows up

1. How I reproduced it:  Opened the playlist (GET /playlists/<playlist_id>/songs) and counted the songs.
Added one more song (POST /playlists/<playlist_id>/songs) and re-fetched.

2. How I found it: The docstring explicitly says "this function returns all songs in the playlist," but the return statement slices the list right before returning it — a direct contradiction worth checking first.

3. Root cause:
pythonreturn [song.to_dict() for song in songs[:-1]]
songs[:-1] drops the last element of the ordered list every time, regardless of how many songs are in the playlist. That's why the last song added never shows up in the response.

4. Fix: Return the full list:
pythonreturn [song.to_dict() for song in songs]
I checked afterward that the Playlists with 1 song previously returned an empty list ([:-1] on a single-item list is []) — now correctly returns that one song.

# AI Usage Section

**Navigation**

I used Claude to help trace each reported issue to its root cause in the relevant service file (streak_service.py, search_service.py, playlist_service.py). For each bug, I gave it the issue title and the specific file, and asked it to identify where the code's behavior diverged from its documented intent.

**What it helped me understand**

In the streak bug, it flagged that a conditional clause (today.weekday() != 6) had no basis in the function's own docstring, which was the fastest way to spot an out-of-place condition rather than manually simulating dates across a week.
In the search bug, it pointed out that a SQL join was present but never used in the filter or select — a "why does this join exist" question I might have glossed over since the query still ran without errors.
In the playlist bug, it noticed the contradiction between the docstring ("returns all songs") and the actual return statement (a list slice dropping the last item), which was a one-line fix but easy to miss since the function looks correct at a glance.

**Where I verified or overrode the output**

I did not treat any AI-proposed root cause as final without checking it against the docstring/spec myself line-by-line, since the tool's diagnosis was based on static reading of the code, not on actually running it.
Before accepting each fix, I confirmed it didn't rely on assumptions about code I hadn't shown it — e.g., for the search fix, I made sure Song.to_dict() populates tags via the model relationship rather than depending on the join that got removed, since the AI hadn't seen models.py.
I ran pytest tests/ after each fix to confirm the existing test suite passed and to check for any test that encoded the buggy behavior as "expected" (e.g., a Sunday-reset test would need updating, since that behavior was the bug itself).
For the "side-effect check" sections, I independently re-read the surrounding functions (get_playlist, get_user_playlists, get_song) to confirm the fixes were scoped to only the broken function and didn't touch unrelated logic — I didn't just take the AI's word that nothing else was affected.
