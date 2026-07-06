# Project 5 - Mixtape Bug Hunt

## Codebase Map

### Main Files

#### app.py
- Creates and configures the Flask application.
- Initializes the database.
- Registers all application routes.

#### models.py
- Defines the SQLAlchemy database models used throughout the application.
- Stores the relationships between users, songs, playlists, notifications, and other entities.

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
