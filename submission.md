## AI Usage

I used AI tools during codebase navigation and debugging to clarify what individual functions and paths were doing before I formed my own hypotheses. For example, I asked about the behavior of `get_playlist_songs`, and the explanation helped me trace how playlist data was being returned and why one bug was happening. I also asked about the flow around streak updates and feed generation so I could compare the code’s actual control flow against the bug reports.

I did not use AI to write or patch the fixes. I verified the AI’s explanations by checking the relevant files and following the calls myself, and in a few cases the AI’s first explanation was incomplete, so I had to go back to the code and narrow down the exact condition causing the bug. That back-and-forth was useful because it forced me to confirm the logic directly instead of trusting a surface-level explanation.

## Codebase Map
    
### Routes

- `feed.py` handles requests for a user’s friend activity. It exposes endpoints for “listening now” and broader activity history, and its job is to take request data, call the feed service, and return the response.
- `playlists.py` handles playlist creation, retrieving playlist metadata, and adding or retrieving songs in a playlist. It is the route layer for playlist-related requests and passes the actual work to the playlist service.
- `songs.py` handles song lookup, search, ratings, and listening events. It is the main entry point for song-related actions and delegates business logic to the search and song-related service functions.
- `users.py` handles user lookup, streak retrieval, notifications, and marking notifications as read. It is the route layer that connects user-facing requests to notification and streak logic.


### Services

- `feed_service.py` builds the feed data shown to users by collecting friends’ recent listening activity and past activity.
- `notification_service.py` creates, retrieves, and marks notifications as read. It is also used when other actions, such as rating a song or adding one to a playlist, should trigger a notification.
- `playlist_service.py` contains the logic for creating playlists, retrieving playlist metadata, and returning the songs in a playlist.
- `search_service.py` handles song search and song lookup by ID.
- `streak_service.py` handles listening streak logic, including recording listening events and updating the current streak.

### Models

- `models.py` defines the database schema for users, tags, songs, playlists, ratings, listening events, and notifications. It also defines the association tables that connect users to songs, users to playlists, and songs to tags.
    
### Tests

- `test_playlists.py` checks playlist-song retrieval, including ordering and the empty-playlist case.
- `test_search.py` checks song search behavior, including matches, duplicate prevention, and empty queries.
- `test_streaks.py` checks streak behavior across consecutive days, skipped days, multiple listens on the same day, and Sunday behavior.

### App

- `app.py` is the application entry point. It initializes Flask, configures the database connection, and registers the routes.

### Data Flow for streak updates and notifications:
 
1. The user listens to a song through the song/listening endpoint.
2. The request is handled in `songs.py`, which passes the event into the listening and streak logic.
3. `streak_service.py` records the event and updates the user’s streak.
4. return the listening event back to the songs/listening endpoint. 

### Patterns seen:

    - The application follows a layered architecture pattern, separating concerns into distinct layers: routes, services, and models. This promotes maintainability and scalability by allowing each layer to focus on its specific responsibilities. Additionally, because the application is structured in a modular way, it is easier to test and debug individual components without affecting the entire system. 

    - In almost every function, there is a form of error handling, which is important for ensuring that the application can gracefully handle situations where an action fails to complete successfully. 


### Root Cause Analysis of Bugs

### Bug #1: My listening streak keeps resetting | `streak_service.py`

- Reproduction steps: I created a user, had them listen on Saturday, then had them listen again on Sunday. The streak reset to 1 instead of continuing to 2.
- Navigation strategy: I started in `test_streaks.py` to see exactly which edge case was failing, then followed the test into `streak_service.py` and inspected the streak update path around the date comparison logic. Once I found the condition that treated Sunday differently, the source of the reset was clear.
- Root cause: The update logic used a condition that required the previous listen to be yesterday **and** the current day to not be Sunday. That extra Sunday check broke the streak even when the listen was correctly consecutive. The specific comparison was the problem, not the surrounding streak bookkeeping.
- Fix description: I removed the condition that prevented streak continuation on Sunday and kept the logic tied to the actual day gap instead.
- Side-effect check: After the fix, I re-ran the streak tests for consecutive days and skipped days to make sure the change only affected the Sunday case and did not break normal streak increments.

### Bug 2: Friends Listening Now shows people from yesterday | `feed_service.py` 

- Reproduction steps: I had one user listen late in the evening, then checked the feed after midnight when a friend had listened earlier the previous day. The friend still appeared in “Friends Listening Now” even though the activity was no longer current.
- Navigation strategy: I traced the feed output from `feed.py` into `feed_service.py`, then followed the recent-activity filter until I found the time cutoff used to decide which listens counted as “now.” The bug became obvious once I compared the threshold against the actual time window the feature was supposed to represent.
- Root cause: The filter used a fixed 24-hour threshold, which meant a listen from “yesterday” could still count as current if it was within the last 24 hours. The mistake was not the retrieval of activity, but the definition of recency.
- Fix description: I tightened the logic so the feed only includes activity that matches the intended “currently listening” window rather than a broad 24-hour span.
- Side-effect check: I checked the broader activity feed and the notification-related paths to make sure narrowing the recent-listening filter did not remove valid past activity or change unrelated feed behavior.

Bug #3: The last song in a playlist never shows up | `playlist_service.py`

- Reproduction steps: I created a playlist with 3 songs and then requested its songs. The response consistently omitted the final song in the list.
- Navigation strategy: I started from the playlist test in `test_playlists.py`, then followed the call into `playlist_service.py` and inspected the return path of `get_playlist_songs()`. The root cause was confirmed when I saw the slicing operation that intentionally dropped the last element.
- Root cause: `get_playlist_songs()` returned `songs[:-1]` instead of the full list. That slice always excluded the final song, so the bug appeared on every playlist with at least one song.
- Fix description: I changed the return value to include the full song list instead of excluding the last element.
- Side-effect check: I rechecked the empty-playlist case and the song-ordering test to confirm that returning the full list did not change sorting or break playlists with zero songs.

## Screenshot of git log showing the commits for the bug fixes
![Screenshot](image.png)