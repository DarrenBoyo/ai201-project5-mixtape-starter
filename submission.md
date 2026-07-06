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


### Issue #5: The last song in a playlist never shows up

How I reproduced it:
- Seeded the database with `python seed_data.py`.
- Started the app with `flask run`.
- Opened `/playlists/a3028f41-e377-4aae-bcfd-429989dfb0f6/songs`.
- The seeded playlist should contain 7 songs, but the response returned only 6.

Expected behavior:
- The endpoint should return all 7 songs in the playlist.

Actual behavior:
- The endpoint returned only 6 songs, meaning the last song was missing.

### Issue #2: Friends Listening Now shows people from yesterday

How I reproduced it:
- Seeded the database with `python seed_data.py`.
- Started the app with `flask run`.
- Opened `/feed/9e7dddd6-66c5-473e-b8f4-8a814c2f08ca/listening-now`.
- The response included a friend activity from the previous day.

Expected behavior:
- The feed should only show recent listening events, such as activity from the past 30 minutes.

Actual behavior:
- The feed included older activity from the previous day.
### Issue #4: Rating a song does not create a notification

How I reproduced it:
- Ran `python seed_data.py`.
- Started the Flask app.
- Sent a `POST` request to `/songs/<song_id>/rate` using a valid user ID and score.
- Checked the song owner’s notifications at `/users/<owner_user_id>/notifications`.

Expected behavior:
- The song owner should receive a notification when another user rates their song.

Actual behavior:
- The rating was saved, but no notification was created.

---
