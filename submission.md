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


### Bugs

| # | Title | Affected service |
|---|-------|-----------------|
| 1 | My listening streak keeps resetting | `streak_service.py` |
| 2 | Friends Listening Now shows people from yesterday | `feed_service.py` |
| 4 | I got notified when a friend added my song to a playlist but not when they rated it | `notification_service.py` |

#### Bug 1

### Reproduction of bug

### Root cause

### Fix


#### Bug 2

### Reproduction of bug

### Root cause

### Fix


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


#### Bug 4

### Reproduction of bug

### Root cause

### Fix


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

Fixing Line 66 to `for songs in songs` resolved the issue. The above tests also pass as a result.

`get_playlist_songs` is only used in `playlists.py` and once, so the best way to check the fix worked was to re-rerun my request to the router and verify that it returns the 7 items.

