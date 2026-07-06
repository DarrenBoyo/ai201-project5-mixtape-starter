# Project 5 - Mixtape Bug Hunt

## Codebase Map

### Main Files

#### app.py
- Creates and configures the Flask application.
- Initializes the database.
- Registers all application routes.

### models.py

`models.py` defines the database structure for the Mixtape app using SQLAlchemy.

Main models:

- `User`: represents app users. Stores username, email, listening streak, and last listened time.
- `Song`: represents songs shared by users. Stores title, artist, album, genre, share note, and the user who shared it.
- `Rating`: represents a user's rating for a song. Each user can only rate the same song once because of the unique constraint on `user_id` and `song_id`.
- `Playlist`: represents a user-created playlist. Playlists can contain many songs.
- `Notification`: stores notifications sent to users, such as when someone interacts with their song.
- `ListeningEvent`: records when a user listens to a song.
- `Tag`: stores labels/categories that can be attached to songs.

Association tables:

- `friendships`: connects users to other users as friends.
- `song_tags`: connects songs to tags.
- `playlist_entries`: connects playlists to songs and also stores the song position, who added it, and when it was added.

Important pattern:

The models store data and relationships, but they do not contain much business logic. Most feature logic is handled in the `services/` files.

#### routes/

##### routes/songs.py
- Handles song-related endpoints.
- Supports searching, sharing, and rating songs.
- Delegates business logic to the appropriate service.

##### routes/playlists.py
- Handles playlist creation and playlist management.
- Calls `playlist_service.py` for playlist operations.

##### routes/users.py
- Handles user profile information, listening streaks, and notifications.
- Uses the appropriate service modules to perform the business logic.

##### routes/feed.py
- Handles the "Friends Listening Now" feed.
- Delegates feed generation to `feed_service.py`.

### services/

#### streak_service.py
Contains the business logic for calculating and updating user listening streaks.

#### feed_service.py
Generates the Friends Listening Now feed.

#### search_service.py
Implements the song search functionality.

#### notification_service.py
Creates and retrieves notifications for user activity.

#### playlist_service.py
Retrieves playlists and their songs.

---

## Data Flow Example

### User rates a song

1. Client sends:

```
POST /songs/<song_id>/rate
```

2. The request is handled by:

```
routes/songs.py
```

3. The route validates the request and calls:

```
notification_service.py
```

4. The service creates a notification for the song owner.

5. The notification is saved in the database through the models defined in `models.py`.

6. The route returns a response to the user.

---

## Organization Patterns

After reading the codebase, I noticed several design patterns:

- Routes are responsible for handling HTTP requests and responses.
- Business logic is separated into the `services/` directory.
- Database models are centralized in `models.py`.
- The application follows a layered architecture:
  ```
  Client
      ↓
  Route
      ↓
  Service
      ↓
  Models / Database
      ↓
  Response
  ```
- Each service focuses on a single feature, making it easier to isolate bugs.
- The README indicates that all five bugs are located in the service layer, so debugging should begin by tracing requests from the routes into the corresponding service functions.


## Root Cause Analysis

### Issue #1: My listening streak keeps resetting

**How I reproduced it:**  
I ran `pytest tests/test_streaks.py`. The test `test_streak_increments_on_sunday` failed. It listened on Saturday, then Sunday, and expected the streak to increase from 1 to 2, but the streak reset to 1.

**How I found the root cause:**  
I traced the failing test into `services/streak_service.py`, specifically `update_listening_streak()`. The failure happened only when the second listen was on Sunday, so I focused on the condition that checks `days_since_last == 1` and the weekday.

**The root cause:**  
Python’s `datetime.weekday()` returns `6` for Sunday. The code only incremented the streak when `days_since_last == 1` and the current day was not Sunday. Because Sunday was excluded, a valid Saturday-to-Sunday listen was treated as a reset instead of a consecutive day.

**Your fix and side-effect check:**  
I removed the Sunday exclusion so any listen exactly one day after the previous listen increments the streak. I reran `pytest tests/test_streaks.py` to confirm the Sunday test passed and the other streak tests still passed.
---

### Issue #2: Friends Listening Now shows people from yesterday

**How I reproduced it:**  
Seeded the database, started Flask, then opened `/feed/<nova_user_id>/listening-now`. The feed included activity older than the “listening now” window.

**How I found the root cause:**  
I traced `routes/feed.py` to `services/feed_service.py`, specifically `get_friends_listening_now()`.

**The root cause:**  
`RECENT_THRESHOLD` was set to `timedelta(hours=24)`, so the feed treated anything from the last 24 hours as “listening now.” That allowed yesterday’s activity to appear.

**My fix and side-effect check:**  
Changed the threshold to a shorter “now” window, such as `timedelta(minutes=30)`. I checked that recent friend activity still appears, while older activity no longer appears.

---

### Issue #3: The same song keeps showing up twice in search

**How I reproduced it:**  
Ran `pytest tests/test_search.py`. The multi-tag song test uses “Crown Heights Anthem,” which has three tags. The expected result is one copy of the song.

**How I found the root cause:**  
I looked at `routes/songs.py`, then traced the search route into `services/search_service.py`.

**The root cause:**  
`search_songs()` joined `Song` to `song_tags`, but it did not explicitly deduplicate the results. A song with multiple tag rows could appear multiple times after the join.

**My fix and side-effect check:**  
Added deduplication, such as `.distinct()`, to the query. I checked searches for songs with no tags, one tag, and multiple tags to make sure all still return once.

---

### Issue #4: I got notified when a friend added my song to a playlist but not when they rated it

**How I reproduced it:**  
Sent a POST request to `/songs/<song_id>/rate` with another user’s `user_id` and a score. Then I checked the song owner’s notifications at `/users/<owner_user_id>/notifications`. The rating was saved, but no rating notification appeared.

**How I found the root cause:**  
I traced `routes/songs.py` → `rate_song()` in `services/notification_service.py`. I compared it to `add_to_playlist()`, which already creates a notification.

**The root cause:**  
`rate_song()` created or updated a `Rating`, committed it, and returned it, but never called `create_notification()`. Playlist additions had notification logic, but ratings did not.

**My fix and side-effect check:**  
Added a notification after a successful rating when the rater is not the song owner. I checked that the rating still saves and that self-ratings do not notify the user about their own action.

---

### Issue #5: The last song in a playlist never shows up

**How I reproduced it:**  
Seeded the database and opened `/playlists/<playlist_id>/songs`. The seeded playlist should have 7 songs, but the response returned only 6.

**How I found the root cause:**  
I traced `routes/playlists.py` → `get_playlist_songs()` in `services/playlist_service.py`.

**The root cause:**  
The service queried all songs correctly, but returned `songs[:-1]`. That slice removes the final item from the list every time.

**My fix and side-effect check:**  
Changed the return statement to return every song: `[song.to_dict() for song in songs]`. I checked that playlists still return songs in position order and that empty playlists still return an empty list.
