## AI Usage

    The use of AI tools during codebase navigation and debugging primarily involved asking questions about the purpose of a function and it's current logical flow. For example, I would ask the LLM something like what the return value of get_playlist_songs was and it explained to me that it was returning all songs in a playlist except for the last one. From there, I used this information to guide my debugging process and created an informed diagnostic of the issue. Due to the nature of this project, I did not use AI tools to write any code, but rather to understand the codebase and provide information that allowed me to make my own hypotheses. Any bug fixes that were conducted were done manually and not by an AI tool.

## Roles of Each Layer 
    
### Routes:

    feed.py - The feed route handles requests regarding information on their friends current and past activity, which are songs they are currently listening to or songs they've listened to in the past. Endpoints include /<user_id>/listening-now and /<user_id>/activity which are both GET requests. 

    playlists.py - The playlists route handles requests regarding creating a playlist and getting a playlist's metadata, and adding a song and getting songs in a playlist in sorted order. Endpoints include '/' and /<playlist_id>/songs, which are both POST requests, and /<playlist_id> and /<playlist_id>/songs, which are both GET requests.

    songs.py - The songs route handles requests regarding getting a song through searching, getting it's metabata via it's id, rating the song, or listening to it. Endpoints include /<song_id> and /search, which are both GET requests while /<song_id>/rate and /<song_id>/listen are both POST requests.

    users.py - The users route handles requests regarding getting a specific user, their streak, notifications, and reading a notification. This route is the primary route that alerts the user to new activity that may have occurred on other routes. Endpoints include /<user_id>, /<user_id>/streak, and /<user_id>/notifications which are GET requests, while /<user_id>/notifications/<notification_id>/read is a POST request.


### Services:

    feed_service.py - The feed service handles the business logic for the feed route. It retrieves information about a user's friends' current and past activity, including songs they are currently listening to or have listened to in the past. Functions include get_friends_listening_now() and get_activity_feed().

    notification_service.py - The notification service handles the business logic for notifying an action to the user regarding activity causes from a mutual's interaction. It creates and retrieves notifications for a user, including notifications for when a friend rates a song or adds a song to a playlist. Functions include create_notification(), add_to_playlist(), rate_song(), get_notifications(), and mark_as_read().

    playlist_service.py - The playlist service handles the business logic for the playlists route. It creates and retrieves playlists and adds songs to playlists. Functions include create_playlist(), get_playlist_songs(), get_playlist(), and get_user_playlists().

    search_service.py - The search service handles the business logic for search queries regarding songs. It retrieves songs based on search queries and song IDs. Functions include search_songs() and get_song().

    streak_service.py - The streak service handles the business logic for the streaks. The streak service can get the current streak for a user and update the streak when a user listens to a song. Functions include get_streak(), record_listening_event(), and update_listening_streak().

### Models:

    models.py - The models file defines the database models for the Mixtape Bug Hunt application. The databse models include a User, Tag, Song, Playlist, Rating, ListeningEvent, and Notification models. The User model represents a user of the application, the Tag model represents a tag that can be associated with a song, the Song model represents a song in the application, the Playlist model represents a playlist of songs, the Rating model represents a user's rating of a song, the ListeningEvent model represents an event where a user listens to a song, and the Notification model represents a notification sent to a user. There are also association tables for many-to-many relationships between users and songs, users and playlists, and songs and tags.
    
### Tests:

    test_playlists.py - The test_playlists.py file contains unit tests for the playlist service. It tests the functionality of retrieving playlist songs and confirms if all the songs are returned, they are in the correct order, and that an empty playlist returns an empty list. 

    test_search.py - The test_search.py file contains unit tests for the search service. It tests the functionality of searching for songs and confirms if the search returns matching songs, no duplicates for a song with single tag song, no duplicates songs for a song with multiple tag songs, and that an empty search returns no matches.

    test_streaks.py - The test_streaks.py file contains unit tests for the streak service. It tests the functionality of updating a user's listening streak and confirms if the streak starts at 1 for a new user, increments on consecutive days, does not increment on non-consecutive days (or days when the user is listening more than once), resets after a skipped day, and increments on a Sunday.

### App:

    app.py - The app.py file is the entry point for the Mixtape Bug Hunt application. It initializes the Flask application, sets up the database connection, and registers the routes for the application.

### Data Flow for creating a playlist:

    1. The user sends a POST request to the /playlists endpoint with the necessary data to create a new playlist.
    2. The playlists route receives the request and calls the create_playlist() function in the playlist service.
    3. The playlist service processes the request, creates a new playlist in the database, and returns the newly created playlist's metadata.
    4. The playlists route sends a response back to the user with the newly created playlist's metadata, if successful, or an error message if the request was invalid, as in if the user did not add in the necessary attributes needed for the playlist or an error occurred in making the playlist.

### Patterns seen:

    - The application follows a layered architecture pattern, separating concerns into distinct layers: routes, services, and models. This promotes maintainability and scalability by allowing each layer to focus on its specific responsibilities. Additionally, because the application is structured in a modular way, it is easier to test and debug individual components without affecting the entire system. 

    - In almost every function, there is a form of error handling, which is important for ensuring that the application can gracefully handle situations where an action fails to complete successfully. 


### Root Cause Analysis of Bugs

Bug #1: My listening streak keeps resetting | `streak_service.py`
    - On line 73 of streak_service.py, the code checks if the user listened to a song yesterday and if today is not Sunday. This would cause the streak to reset on Sunday even if the user listened to a song on Saturday. To replicate this bug, I created a user and had them listen to a song on Saturday. Then, I had them listen to a song on Sunday. The streak reset to 1 instead of incrementing to 2.

Bug 2: Friends Listening Now shows people from yesterday | `feed_service.py` 
    
    - Inside the feed_service.py file, the issue arises from the RECENT_THRESHOLD constant being set to 24 hours. This means that if a friend listened to a song within the last 24 hours, they will be shown in the "Friends Listening Now" section, even if they listened to it yesterday. To reproduce this bug, I created a user and had them listen to a song at 4:50 PM. Then, I had their friend listen to a song at 2:36 AM the next day. When I checked the "Friends Listening Now" section, the friend was shown as listening to a song even though they listened to it yesterday.
    
     To fix this bug, there are multiple approaches that can be taken. One approach is to change the RECENT_THRESHOLD constant to a smaller value, such as 1 hour, so that only friends who have listened to a song within the last hour will be shown in the "Friends Listening Now" section. The main issue with this approach is that even if the friend listened to a song within an hour ago, you could be listening to a song at around 12:01 AM and your friend could have listened to a song at 11:59 PM, which would still show them in the "Friends Listening Now" section even though they listened to it yesterday. So, it depends on how the user wants to define "listening now". Though, the possibility of exploring real-time updates or using a more sophisticated time-based filtering mechanism could be considered to ensure that the "Friends Listening Now" section accurately reflects current activity. But, for simplicity, changing the RECENT_THRESHOLD constant to a smaller value, such as 1 hour, is a straightforward solution that can be implemented quickly.

Bug #3: The last song in a playlist never shows up | `playlist_service.py`

    - Inside the playlist_service.py file, the issue arises from the return statement in the get_playlist_songs function. The code currently returns all songs in the playlist except for the last one (songs[:-1]). This means that if a playlist has 5 songs, only the first 4 songs will be returned, and the last song will never be shown. To reproduce this bug, I created a playlist with 3 songs and then retrieved the playlist's songs. The last song was not included in the response. 

    To fix this bug, I modified the return statement to include all songs in the playlist by removing the slicing operation (songs[:-1]). The updated return statement now returns all songs in the playlist, ensuring that the last song is included in the response.


## Screenshot of git log showing the commits for the bug fixes
![Screenshot](image.png)