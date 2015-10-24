# Udacity: Conference Central App

## Setup:

1. Create a new project in the Google Developers console if remote deployment is desired
1. Enter the name of the new project in `app.yaml` as `application:`
1. Generate a client ID from the Google Developers console
2. Paste the new ID in `settings.py` as `WEB\_CLIENT\_ID`
3. and in `app.js` as `CLIENT\_ID`
6. Clone this repo
7. Open it in Google AppEngineLauncher and either run locally or deploy to App Engine
8. Open the API explorer in a browser. The default local address is http://localhost:8080/_ah/api/explorer but your port may vary. If the app is deployed remotely, the host address will change, but the path is the same.

## Task 1:

Implement basic session functions. The required endpoints have been implemented:

		getConferenceSessions(websafeConferenceKey)
		getConferenceSessionsByType(websafeConferenceKey, typeOfSession)
		getSessionsBySpeaker(websafeSpeakerKey)
		createSession(SessionForm, websafeConferenceKey)

The Session class and SessionForm class are defined as required by the instructions, with the addition
that SessionForm also carries a `websafeKey` for the session for easy retrieval later

`createSession()` makes the new session a child of its parent conference.

`getSessionsBySpeaker()` requires a `websafeSpeakerKey` to search. Therefore I have also provided 
a `getSpeakers()` endpoint which returns a list of all speakers, including their `websafeKey`s.

### Design choices:
The specified data model for a Session is implemented as required, though I wouldn't have split `date` and `startTime` into separate fields when there's a perfectly good `DateTimeProperty()` available.

Since `startTime` was specified in the instructions, I chose to implement it as an `IntegerProperty()`. It's not really being used as a time, merely as an ordering index, much like `month` in the `Conference` model (also not necessary, if I were designing the model). Since the only thing we're doing with `startTime` is asking "is one greater than the other", making it an integer suffices, and saves conversions during the `copyToForm` step.

I chose to implement the speaker as an entity rather than a string. This has a number of advantages.
Most importantly, the entity can simply carry more information. The current implementation only has the speaker name, but it would also be useful to include fields for email, specialty, fee, etc. 

Finding featured speakers is also simplified by this implementation. During session creation, if the speaker entity doesn't exist yet, the app doesn't need to check to see if the speaker should be featured. If the speaker does exist, a task is added to the taskqueue to asynchronously check whether the speaker should be featured.

The Speaker kind also allows the speakers to be easily listed, which enables the additional queries in Task 3 below.


## Task 2:

Wishlist functionality is implemented via a repeated StringProperty on the Profile kind.
`addSessionToWishlist(websafeSessionKey)` requires a `websafeKey` which is displayed from any of the endpoints which return sessions.
`getSessionsInWishlist()` simply displays all sessions the user has added to the wishlist

I chose to allow any sessions in the wishlist, not just those ate conferences where the user is registered. The less restrictive implementation gives the user more latitude in planning conference and session attendance.


## Task 3:

All necessary indices were created automatically during local testing, and are listed in `index.yaml`

First additional query: `getSpeakers()` lists all speakers scheduled to give a session at any conference. This list could be useful to a conference creator while organizing the conference. It is also necessary during testing to retrieve `websafeSpeakerKey`s to plug into `getSessionsBySpeaker()`.

Second additional query: `getProlificSpeakers()` lists all speakers who are presenting sessions at more than one conference. Knowing what speakers are in demand can help a conference creator decide who to invite.

Query problem: Finding non-workshop sessions before 7pm presents a difficulty as DataStore only allows one inequality filter per query. I chose to bypass the problem by querying for sessions before 7pm, and then letting the app filter the results through a list comprehension that rejects workshops. It might be more suitable to query on non-workshops, and then reject those after 7pm. Which way is better depends on which restriction rejects the most sessions. Rejecting more on the first step would be more efficient.


## Task 4:

During session creation, the app checks to see if the speaker is already in the DataStore. (In production, this check would need to be much more robust, perhaps offering a choice if two speakers have the same name.) If the speaker doesn't exist yet, they get added. If they're already there, the app sends a task to the taskqueue which will asynchronously perform the queries to determine if the speaker has more than one session at the conference, and thus should be featured. If so, the app adds a promotion string to memcache under a key containing the `websafeConferenceKey`. This string lists the speaker name and their session names.

