# Mixtape Bug Hunt Submission

## AI Usage Section
I used an AI assistant to help me understand the codebase structure by asking it to read the main files and describe their purposes. The AI also helped me trace the data flow of the streak endpoint by looking at `models.py`, the routing, and the `streak_service`. During debugging, I relied on the AI to help me verify Python's `datetime.weekday()` behavior. The most critical collaboration was during the investigation of Bug 3: the AI hypothesized that an `.outerjoin()` caused the search duplication. However, when we ran the tests, the AI's explanation proved incomplete because SQLAlchemy 2.0 implicitly deduplicates objects. This realization forced me to course-correct and pivot to investigating Bug 5 instead.

## Codebase Map

### Main Files
- **`app.py`**: Initializes the Flask application, configures the SQLAlchemy database URI, and registers the application blueprints (routes).
- **`models.py`**: Contains the SQLAlchemy database models (`User`, `Tag`, `Song`, `ListeningEvent`, `Rating`, `Playlist`, `Notification`) and association tables (`friendships`, `song_tags`, `playlist_entries`). This defines the structure of the database and the relationships between entities.
- **`routes/`**: This directory contains the endpoints. For example, `routes/users.py` handles user-related HTTP requests and immediately delegates the core business logic to the `services/` directory.
- **`services/`**: This directory contains the business logic. Instead of putting logic in the routes, the app separates concerns by placing database queries and data manipulation in these service files (e.g., `streak_service.py`, `notification_service.py`).

### Data Flow Example (Fetching User Streak)
1. A client makes a GET request to `/users/<user_id>/streak`.
2. The route handler in `routes/users.py` calls the `get_streak(user_id)` function from `services/streak_service.py`.
3. Inside `get_streak`, the application queries the database using the `User` and `ListeningEvent` models to calculate or retrieve the current streak.
4. The service returns the streak integer back to the route.
5. The route packages the streak into a JSON response `{"user_id": ..., "streak": ...}` and returns it to the client.

## Root Cause Analysis

### Bug 1: My listening streak keeps resetting
- **How I reproduced it**: I wrote a test script (`reproduce_bug1.py`) that simulated a user who had an active streak of 5 and last listened on a Saturday. Then, I triggered the `update_listening_streak` function with `now` set to the following Sunday. The assertion checked if the streak incremented to 6, but it failed because the streak was reset to 1.
- **How I found the root cause**: I examined `services/streak_service.py` to trace the streak calculation logic. I followed the data flow through `update_listening_streak` and looked closely at how `days_since_last` was evaluated. When I spotted `today.weekday() != 6` on the `elif` branch, I knew with absolute confidence I had found the root cause, because there is no logical reason a streak should skip incrementing exclusively on Sundays.
- **The root cause**: The streak update logic had a conditional branch `elif days_since_last == 1 and today.weekday() != 6:`. Since `today.weekday() == 6` evaluates to True on Sundays, the logic bypassed the increment block and fell through to the `else:` block, resetting the streak to 1.
- **My fix and side-effect check**: I removed the `and today.weekday() != 6` condition so the `elif` branch correctly matches any consecutive calendar day. As a side-effect check, I verified that this fix didn't inadvertently break streak calculations for other days of the week. I did this by running the existing `pytest tests/test_streaks.py` suite, which covers Monday-Saturday behavior, and confirmed it still fully passed.

### Bug 2: Friends Listening Now shows people from yesterday
- **How I reproduced it**: I wrote a test script (`reproduce_bug2.py`) to create a listening event for a friend exactly 23 hours ago. I then retrieved the feed using `get_friends_listening_now` and verified that the friend incorrectly showed up in the 'Listening Now' results.
- **How I found the root cause**: I looked at `services/feed_service.py` specifically examining `get_friends_listening_now`. I saw that the logic filters events using a `cutoff` date variable. I traced `cutoff` back to its definition and saw it relied on a constant called `RECENT_THRESHOLD`. Finding that constant at the top of the file gave me the "aha" moment, as adjusting the constant directly controls the time window.
- **The root cause**: The `RECENT_THRESHOLD` constant was set to `timedelta(hours=24)`. This meant any friend who listened to a song within the entire previous day would be included in the "Listening Now" feed, making it a "Listening Recently" feed rather than "Now".
- **My fix and side-effect check**: I changed `RECENT_THRESHOLD` from `timedelta(hours=24)` to `timedelta(minutes=15)` which is a much more appropriate duration for a "now" status. For the side-effect check, I explicitly examined the `get_activity_feed` function to ensure it wouldn't be impacted. I verified that `get_activity_feed` does not rely on `RECENT_THRESHOLD` (it uses a simple limit query instead), confirming my fix was safely isolated.

### Bug 5: The last song in a playlist never shows up
- **How I reproduced it**: I ran the existing `pytest tests/test_playlists.py` test suite. The test `test_playlist_returns_all_songs` failed with an assertion error `assert 4 == 5` and `test_playlist_returns_songs_in_order` failed because "Track 5" was missing from the returned list of tracks.
- **How I found the root cause**: I examined `services/playlist_service.py`, looking at the `get_playlist_songs` function which is responsible for returning the playlist's songs. I traced the query from the database lookup down to the return statement. When I saw the list comprehension at the very end of the function containing `[:-1]`, I was instantly confident I had found the issue, as list slicing is a direct manipulation of the return size.
- **The root cause**: The function explicitly sliced the result list using `[:-1]` before returning it (`return [song.to_dict() for song in songs[:-1]]`). In Python, the slice `[:-1]` returns all elements in the list except for the very last one. This hardcoded slice systematically discarded the final song of every playlist.
- **My fix and side-effect check**: I removed the `[:-1]` slice, changing the line to `return [song.to_dict() for song in songs]`. For the side-effect check, I had to ensure that removing the slice didn't introduce unexpected null values or metadata into the list. I ran the full test suite `pytest tests/test_playlists.py` again, which validated both the length of the list and the specific order of the songs, confirming no regressions occurred.
