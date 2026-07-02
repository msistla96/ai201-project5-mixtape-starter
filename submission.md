#### Mixtape Bugfixes

### Codebase map

### Data Models

There are 7 models: User, Playlist, Notification, Rating, ListeningEvent, Song, Tag. 
SQLAlchemy uses db.Model for defining a table for each entity, db.Relationship to establish one to many or many to many relationships and db.Table to setup complex many to many relationships i.e A Song has many Tags and a Tag is associated with many Songs. 

#### Routers and Services
There are 4 routes: feed, playlists, songs, users which each have its own services: feed_service, notification_service, playlist_service, search_service, streak_service. Each router handles data validation and empty values and delegates actual business logic to the respective service instead.


#### Data flow

When a user creates a playlist for the first time, they hit the `POST playlist/` endpoint in `playlists.py`.
This takes the name, created by and is collaborative fields, making the name and created by as required field.
This calls `create_playlist` from the `playlist_service.py`, searches the user in User table. 
Then creates a Playlist object and inserts into the Playlists table.
Since this uses Blueprint, this allows to define, reuse and provide a unique namespace for any playlist related routing. 

#### Bug 1: My listening streak keeps resetting

### Reproduction of bug

When running pytests/tests, `test_streak_increments_on_sunday` from  `test_streaks.py` threw an error. 

### How to get the root cause

I then traced to the `update_listening_streak` function and checked what could trigger the streak reset.

1. Line 58: If the `last_listened` field for the user is None i.e first song they listened to, it resets the streak. 
I ruled this out as the issue since this service always sets it. 
2. Line 73: If `days_since_last` > 1 i.e more than one day since they last listened or
`days_since_last` = 1 and `today.weekday() == 6` i.e its only been one day since they last listened but today is Sunday,
then it resets the streak. If a user had to see a streak resetting, it would happen when the day they listen is a Sunday even if it's only been a day since the last listen or its been more than one day since the last listen.

### Root cause

In `update_listening_streak`, Line 73 introduces a condition `today.weekday() != 6`. The only valid case for a streak to reset is when it's been more than a day. However this code also resets when the current day is not a Sunday, which has nothing to do with a streak.

### Fix and side effects

#### Fix

Removing `today.weekday() != 6` from the condition makes sure that `days_since_last` is the only relevant check that's done.

#### Side effects

`record_listening_event` is the function that calls `update_listening_streak`.Hence unit testing and user testing should and did confirm that no functionality broke for either functions. 

#### Bug 2: Friends Listening Now shows people from yesterday

### Reproduction of bug

Get a user id from the User table. Make a curl request to `GET feed/<user_id>/listening-now`

    curl -X GET http://localhost:9000/feed/178764e6-ca0f-4ab0-b3a1-693744f26b7a/listening-now 

    {"count":3,"feed":[{"friend":{"id":"f633d101-ae06-4bdb-b4de-13580daad5dc","last_listened_at":"2026-06-30T23:50:40.569017","listening_streak":3,"username":"darius"},"listened_at":"2026-07-01T23:40:40.569017","song":{"album":null,"artist":"The Wanderers","genre":"indie rock","id":"3ae88dfb-ac1b-4d7c-bfec-fef14ecccf55","share_note":null,"shared_at":"2026-06-26T23:50:40.569017","shared_by":"178764e6-ca0f-4ab0-b3a1-693744f26b7a","tags":[],"title":"Midnight Drive"}},{"friend":{"id":"1a0deefd-7047-4599-b67b-bf8909737576","last_listened_at":null,"listening_streak":0,"username":"simone"},"listened_at":"2026-07-01T23:35:40.569017","song":{"album":null,"artist":"Elara Moon","genre":"ambient","id":"a9db71fe-f6ab-4554-9b13-f6dcdc3b4c52","share_note":null,"shared_at":"2026-06-26T23:50:40.569017","shared_by":"178764e6-ca0f-4ab0-b3a1-693744f26b7a","tags":[],"title":"Still Waters"}},{"friend":{"id":"33a0f114-2df8-4e20-af16-4cedba960551","last_listened_at":"2026-07-01T20:50:40.569017","listening_streak":12,"username":"kenji"},"listened_at":"2026-07-01T23:30:40.569017","song":{"album":null,"artist":"Coastal Highway","genre":"indie","id":"3af684aa-cb33-4954-b0dc-a49a7b08d1eb","share_note":null,"shared_at":"2026-06-26T23:50:40.569017","shared_by":"178764e6-ca0f-4ab0-b3a1-693744f26b7a","tags":[],"title":"First Light"}}]}

Based on the return values, these are the following `listened_at` values:

listened_at":"2026-07-01T23:40:40.569017
listened_at":"2026-07-01T23:35:40.569017
listened_at":"2026-07-01T23:30:40.569017

Based on Line 112-117 in  `seed_data.py`, the recently listened events should be within the last 30 minutes threshold. 
But based on the response from above, it is pulling every record beyond the threshold. 


### How to get root cause

From the same route I followed from reproduction, I traced the call from `listening-now` to `get_friends_listening_now` in `feed_service.py`. 


### Root cause

Line 32 was where the valid dates were calculated:
    cutoff = datetime.now(timezone.utc) - RECENT_THRESHOLD
Line 13 has the `RECENT_THRESHOLD = timedelta(hours=24)`
Line 45 filtered by each 
    ListeningEvent.listened_at >= cutoff

Putting this together, this meant that the `listened_at` timestamps could be behind upto 24 hours and before the present time and date. This did not match with the expectations set by `seed_data.py`. 

### Fix and side effects

## Fix

The fix was to Line 13:
    `RECENT_THRESHOLD = timedelta(minutes=30)`

After making the change, I retested our reproduction scenario:

Before 7pm

curl -X GET http://localhost:9000/feed/178764e6-ca0f-4ab0-b3a1-693744f26b7a/listening-now
{"count":3,"feed":[{"friend":{"id":"f633d101-ae06-4bdb-b4de-13580daad5dc","last_listened_at":"2026-06-30T23:50:40.569017","listening_streak":3,"username":"darius"},"listened_at":"2026-07-01T23:40:40.569017","song":{"album":null,"artist":"The Wanderers","genre":"indie rock","id":"3ae88dfb-ac1b-4d7c-bfec-fef14ecccf55","share_note":null,"shared_at":"2026-06-26T23:50:40.569017","shared_by":"178764e6-ca0f-4ab0-b3a1-693744f26b7a","tags":[],"title":"Midnight Drive"}},{"friend":{"id":"1a0deefd-7047-4599-b67b-bf8909737576","last_listened_at":null,"listening_streak":0,"username":"simone"},"listened_at":"2026-07-01T23:35:40.569017","song":{"album":null,"artist":"Elara Moon","genre":"ambient","id":"a9db71fe-f6ab-4554-9b13-f6dcdc3b4c52","share_note":null,"shared_at":"2026-06-26T23:50:40.569017","shared_by":"178764e6-ca0f-4ab0-b3a1-693744f26b7a","tags":[],"title":"Still Waters"}},{"friend":{"id":"33a0f114-2df8-4e20-af16-4cedba960551","last_listened_at":"2026-07-01T20:50:40.569017","listening_streak":12,"username":"kenji"},"listened_at":"2026-07-01T23:30:40.569017","song":{"album":null,"artist":"Coastal Highway","genre":"indie","id":"3af684aa-cb33-4954-b0dc-a49a7b08d1eb","share_note":null,"shared_at":"2026-06-26T23:50:40.569017","shared_by":"178764e6-ca0f-4ab0-b3a1-693744f26b7a","tags":[],"title":"First Light"}}]}


At 7.06 pm
curl -X GET http://localhost:9000/feed/178764e6-ca0f-4ab0-b3a1-693744f26b7a/listening-now
{"count":1,"feed":[{"friend":{"id":"f633d101-ae06-4bdb-b4de-13580daad5dc","last_listened_at":"2026-06-30T23:50:40.569017","listening_streak":3,"username":"darius"},"listened_at":"2026-07-01T23:40:40.569017","song":{"album":null,"artist":"The Wanderers","genre":"indie rock","id":"3ae88dfb-ac1b-4d7c-bfec-fef14ecccf55","share_note":null,"shared_at":"2026-06-26T23:50:40.569017","shared_by":"178764e6-ca0f-4ab0-b3a1-693744f26b7a","tags":[],"title":"Midnight Drive"}}]}

7:17 pm

curl -X GET http://localhost:9000/feed/178764e6-ca0f-4ab0-b3a1-693744f26b7a/listening-now
{"count":0,"feed":[]}


Explanation for why the results changed correctly:

The three events and their fixed timestamps:

Friend	listened_at	Falls out of the 30-min window once "now" passes
kenji	23:30:40	24:00:40
simone	23:35:40	24:05:40
darius	23:40:40	24:10:40
First curl (count: 3): all three request cutoffs. Server "now" was still ≤ 24:00:40, so now − 30min ≤ 23:30:40, keeping kenji, simone, and darius all inside the window.

Second curl (count: 1): by the time it re-ran, server "now" had passed 24:00:40 (and 24:05:40), pushing the cutoff past kenji's and simone's timestamps — now − 30min > 23:30:40 and > 23:35:40 — so they dropped out. Darius's 23:40:40 was still ≥ now − 30min, so he's the only one left.

Validity going forward: darius will also disappear once real time passes 24:10:40 (i.e., 23:40:40 + 30min). After that, this same seeded data will return count: 0 for that user.

## Regression Tests

Since live testing against time is not always feasible, I also added tests simulating these scenarios under `tests/test_streaks.py` . With these tests present before, the bug would have been caught early on.

## Side effects

Using regression and user testing, it is confirmed that no functionality broke. Moreover, `get_friends_listening_now` is only present for `feed.py` and hence does not break other services.

#### Bug 3: The same song keeps showing up twice in search

### Reproduction of bug

To reproduce the bug, I first took a look at search_songs() `search_service.py`. I saw that it used the song_tags table to get the list of tags and hypothesized that this could be the issue.
I then searched for songs that have more than one tag:

    SELECT song.id, song.title, GROUP_CONCAT(tag.name, ', ') AS tags
    FROM song
    LEFT JOIN song_tags ON song.id = song_tags.song_id
    LEFT JOIN tag ON tag.id = song_tags.tag_id
    GROUP BY song.id, song.title;

I chose Crown Heights as the song since it had 3 tags:

    ["rap","hip-hop","boom bap"],"title":"Crown Heights Anthem"]
I then hit the search endpoint using the song name:

    curl -G "http://localhost:9000/songs/search" --data-urlencode "q=Crown Heights"


Surprisingly, it gave me only one result:

    {"count":1,"results":[{"album":null,"artist":"Borough Kings","genre":"rap","id":"48508db6-8b2c-4a82-b064-687df84a075a","share_note":null,"shared_at":"2026-06-30T20:06:15.807978","shared_by":"f597475e-3afb-42e6-8740-30df30635a5d","tags":["rap","hip-hop","boom bap"],"title":"Crown Heights Anthem"}]}


### Root cause

Using AI and a bunch of searches, I came across the following explanation:
    session.query(Song)...all() (legacy Query) generates the same SQL as 2.0-style select(), but adds an extra post-processing step that auto-dedupes full-entity results by primary key.

This explained why even though the join happened with `song_tags`, it automatically deduplicated Song results by primary key. 

Here's a query that will produce duplicate song results for each tag:

    stmt = (
    #     db.session.query(Song)
    #     .outerjoin(song_tags, Song.id == song_tags.c.song_id)
    #     .filter(
    #         db.or_(
    #             Song.title.ilike(f"%{query}%"),
    #             Song.artist.ilike(f"%{query}%"),
    #         )
    #     )
    # )
    
    # results = db.session.execute(stmt.statement).scalars().all()


    {"count":3,"results":[{"album":null,"artist":"Borough Kings","genre":"rap","id":"48508db6-8b2c-4a82-b064-687df84a075a","share_note":null,"shared_at":"2026-06-30T20:06:15.807978","shared_by":"f597475e-3afb-42e6-8740-30df30635a5d","tags":["rap","hip-hop","boom bap"],"title":"Crown Heights Anthem"},{"album":null,"artist":"Borough Kings","genre":"rap","id":"48508db6-8b2c-4a82-b064-687df84a075a","share_note":null,"shared_at":"2026-06-30T20:06:15.807978","shared_by":"f597475e-3afb-42e6-8740-30df30635a5d","tags":["rap","hip-hop","boom bap"],"title":"Crown Heights Anthem"},{"album":null,"artist":"Borough Kings","genre":"rap","id":"48508db6-8b2c-4a82-b064-687df84a075a","share_note":null,"shared_at":"2026-06-30T20:06:15.807978","shared_by":"f597475e-3afb-42e6-8740-30df30635a5d","tags":["rap","hip-hop","boom bap"],"title":"Crown Heights Anthem"}]}

Note that there are multiple ways to remove the deduplication processing as it only works when querying full Entities.

### Fix
No fix is needed with the current code and the test `test_search_no_duplicates_single_tag_song` under `test_search.py` passes as well. This is a classic case of external libraries and behavior affecting the same piece of code, where ghe code on its own reads as a potential bug but the underlying behavior produces a different result that no tests could have caught.


#### Bug 4: I got notified when a friend added my song to a playlist but not when they rated it

### Reproduction of bug

The seed data has an example at Line 171 where a notification was created for the above. Using this example, I obtained the following data needed:

1. Song:
2. Song sharer:
3. Song listener/rater id:

First request is to get the notifications for the song sharer:

    curl -X GET "http://localhost:9000/users/178764e6-ca0f-4ab0-b3a1-693744f26b7a/notifications?unread_only=false"

    {"count":1,"notifications":[{"body":"darius added your song 'Midnight Drive' to the playlist 'Late Night Vibes'.","created_at":"2026-07-01T23:50:40.594485","id":"1d3a183b-0aa1-4148-9173-046f304be6fe","read":false,"type":"song_added_to_playlist","user_id":"178764e6-ca0f-4ab0-b3a1-693744f26b7a"}]}


Next request is to rate the song as the song listener

    curl -X POST http://localhost:9000/songs/3ae88dfb-ac1b-4d7c-bfec-fef14ecccf55/rate \
    -H "Content-Type: application/json" \
    -d '{"user_id": "f633d101-ae06-4bdb-b4de-13580daad5dc", "score": 3}'

    {"id":"30583878-ec62-43b6-a755-24a54c334bfc","rated_at":"2026-07-02T00:35:13.345241","score":3,"song_id":"3ae88dfb-ac1b-4d7c-bfec-fef14ecccf55","user_id":"f633d101-ae06-4bdb-b4de-13580daad5dc"}

Final request is to get the notifications for the song sharer again:

    curl -X GET "http://localhost:9000/users/178764e6-ca0f-4ab0-b3a1-693744f26b7a/notifications?unread_only=false"
    
    {"count":1,"notifications":[{"body":"darius added your song 'Midnight Drive' to the playlist 'Late Night Vibes'.","created_at":"2026-07-01T23:50:40.594485","id":"1d3a183b-0aa1-4148-9173-046f304be6fe","read":false,"type":"song_added_to_playlist","user_id":"178764e6-ca0f-4ab0-b3a1-693744f26b7a"}]}

This lines up with the user issue that the rating notification was not received.

### How to get root cause

I went to the `notification_service ` where `add_to_playlist` is called from `add_to_playlist`, based on the information I got from the seeded data and my reproduction. Lines 65-70 shows how the notification is send to the song sharer. In the same file, there is also `rate_song`, which is where the code gets and stores the rating.

### Root cause

`rate_song` did not have any notification setup even though it is part of the `notification_service` and hence it stored the rating, but did not notify the song sharer.

### Fix and side effects

#### Fix

The following code was added after Line 108(db.session.commit()):
    ``` python
    if song.shared_by != user_id:
            create_notification(
                user_id=song.shared_by,
                notification_type="song_rated_by_user",
                body=f"{rater.username} rated your song '{song.title}' as '{rating.score}'.",
            )
    ```
The notification type, username and scores are added.

After adding this change, I re-ran the setup from the reproduction section:

    curl -X GET "http://localhost:9000/users/178764e6-ca0f-4ab0-b3a1-693744f26b7a/notifications?unread_only=false"
    
    {"count":2,"notifications":[{"body":"darius rated your song 'Midnight Drive' as '3'.","created_at":"2026-07-02T00:42:01.202869","id":"e70fa4c7-9e46-4eba-9e34-3c6a85af1241","read":false,"type":"song_rated_by_user","user_id":"178764e6-ca0f-4ab0-b3a1-693744f26b7a"},{"body":"darius added your song 'Midnight Drive' to the playlist 'Late Night Vibes'.","created_at":"2026-07-01T23:50:40.594485","id":"1d3a183b-0aa1-4148-9173-046f304be6fe","read":false,"type":"song_added_to_playlist","user_id":"178764e6-ca0f-4ab0-b3a1-693744f26b7a"}]}

Compared to the last time, the song sharing user now gets a notification after another user has rated their song.

#### Side effects

This code change only affects `songs.py`, so unit testing and user testing confirms that the functionality did not break.

#### Bug 5: The last song in a playlist never shows up

### Reproduction of Bug

Get a random playlist id from `mixtape.db` through a SQL query(You can use AI to assist you with the query):

    select * from Playlist;

167a25c9-a048-45b2-a951-f4e00aab91b3 - Late Night Vibes

Use this playlist id to do a `GET playlists/<playlist_id>/songs` request to check what the request returns;
In this case it returns 6 songs

    {"count":6,"songs":[{"album":null,"artist":"The Wanderers","genre":"indie rock","id":"3f3c69c1-80c2-4958-99e6-c1951cadccb5","share_note":null,"shared_at":"2026-06-26T16:28:30.530247","shared_by":"94b0aca3-8e41-4630-a1c0-e92a8f25684d","tags":[],"title":"Midnight Drive"},{"album":null,"artist":"Elara Moon","genre":"ambient","id":"6bf8b3db-e2cc-439c-8253-49c6792e0ac4","share_note":null,"shared_at":"2026-06-26T16:28:30.530247","shared_by":"94b0aca3-8e41-4630-a1c0-e92a8f25684d","tags":[],"title":"Still Waters"},{"album":null,"artist":"Coastal Highway","genre":"indie","id":"2aaf3380-8618-45dd-ab1e-58ad51f8670e","share_note":null,"shared_at":"2026-06-26T16:28:30.530247","shared_by":"94b0aca3-8e41-4630-a1c0-e92a8f25684d","tags":[],"title":"First Light"},{"album":null,"artist":"Street Collective","genre":"hip-hop","id":"4f45aaf5-bce8-43c0-b40b-08dfda663b24","share_note":null,"shared_at":"2026-06-28T16:28:30.530247","shared_by":"a62adbea-dc84-49fe-b5c1-08e3d164f73e","tags":["hip-hop"],"title":"Block Party"},{"album":null,"artist":"Nova Blix","genre":"lo-fi","id":"9c7f13b1-a9ed-46e0-973b-124618676227","share_note":null,"shared_at":"2026-06-28T16:28:30.530247","shared_by":"a62adbea-dc84-49fe-b5c1-08e3d164f73e","tags":["lo-fi"],"title":"Late Night Session"},{"album":null,"artist":"Solange K","genre":"r&b","id":"643c31bd-6fb7-4ff8-912b-1fed011b02b1","share_note":null,"shared_at":"2026-06-28T16:28:30.530247","shared_by":"a62adbea-dc84-49fe-b5c1-08e3d164f73e","tags":["r&b"],"title":"Golden Hour"}]} 

Since the bug says that the last song doesn't show, verify if the actual playlist in the database has the same number of songs or not. You can do this by performing a SQL query that joins on playlist_entries through the playlist id to get the list of songs.(I used AI to help me construct a query as I was not familiar with flask shell)

    sqlite> SELECT song.*
    ...> FROM song
    ...> JOIN playlist_entries ON song.id = playlist_entries.song_id 
    ...> WHERE playlist_entries.playlist_id = '167a25c9-a048-45b2-a951-f4e00aab91b3';
    2aaf3380-8618-45dd-ab1e-58ad51f8670e|First Light|Coastal Highway||indie|94b0aca3-8e41-4630-a1c0-e92a8f25684d|2026-06-26 16:28:30.530247|
    3f3c69c1-80c2-4958-99e6-c1951cadccb5|Midnight Drive|The Wanderers||indie rock|94b0aca3-8e41-4630-a1c0-e92a8f25684d|2026-06-26 16:28:30.530247|
    4f45aaf5-bce8-43c0-b40b-08dfda663b24|Block Party|Street Collective||hip-hop|a62adbea-dc84-49fe-b5c1-08e3d164f73e|2026-06-28 16:28:30.530247|
    643c31bd-6fb7-4ff8-912b-1fed011b02b1|Golden Hour|Solange K||r&b|a62adbea-dc84-49fe-b5c1-08e3d164f73e|2026-06-28 16:28:30.530247|
    6bf8b3db-e2cc-439c-8253-49c6792e0ac4|Still Waters|Elara Moon||ambient|94b0aca3-8e41-4630-a1c0-e92a8f25684d|2026-06-26 16:28:30.530247|
    7c34d46d-c356-4cea-b628-4e5b3d419d37|Free Throws|Hoop Dreams||rap|a62adbea-dc84-49fe-b5c1-08e3d164f73e|2026-06-28 16:28:30.530247|
    9c7f13b1-a9ed-46e0-973b-124618676227|Late Night Session|Nova Blix||lo-fi|a62adbea-dc84-49fe-b5c1-08e3d164f73e|2026-06-28 16:28:30.530247|

(You can simply check the size of the result, but for this request it's not really needed)

From this, the playlist Late Night Vibes actually has 7 songs, but the request sends only 6.

### How root cause was found

I traced my request from my reproduction to `get_playlist_songs` in `playlist_service.py`. Upon reading the code, the filtering logic looked fine to me until line 66.

### Root cause

In `playlist_service.py` the function `get_playlist_songs` filters the songs correctly but in line 66 the loop was incorrect: `for songs in songs[:-1]` returned songs until the last song. 

Another place where the bug was flagged: 
In `test_playlists.py`, `test_playlist_returns_all_songs` and `test_playlist_returns_songs_in_order` threw errors when run before the bug fix with the following errors:

    FAILED tests/test_playlists.py::test_playlist_returns_all_songs - AssertionError: assert 4 == 5
    FAILED tests/test_playlists.py::test_playlist_returns_songs_in_order - AssertionError: assert ['Track 1', '...3', 'Track 4'] == ['Track 1', '...4', 'Track 5']


### Fix and side effects

#### Fix

Fixing Line 66 to `for songs in songs` resolved the issue. The above tests also pass as a result.

#### Side effects

`get_playlist_songs` is only used in `playlists.py` and once, so the best way to check the fix worked was to re-rerun my request to the router and verify that it returns the 7 items, along with the unit tests.

