# Design a Dating App Like Tinder - Practice Notes

### Design a Dating App Like Tinder Guided Practice - April 18, 2026

<!-- EXCALIDRAW_PRACTICE practiceId="cmgtvbtus02bh08adx2o9vajx" -->

#### Key Takeaways

**Requirements**

* For matching systems like Tinder, strong consistency is critical for the core 'like' and 'match' features. When two users like each other, both must see the match immediately and simultaneously - eventual consistency would create a poor user experience with missed or delayed matches that could cause users to lose trust in the platform.

* The CAP theorem trade-off isn't always 'availability over consistency' - the correct choice depends on the specific feature. For Tinder, while general browsing can tolerate eventual consistency, the mutual matching logic requires strong consistency to prevent scenarios where User A sees a match but User B doesn't.

* Non-functional requirements should be quantified with specific metrics to make them measurable and testable. Instead of 'high scalability,' specify concrete numbers like 'handles 10 million swipes per day with 3x spike during evening peak hours (7-10 PM)' to guide architectural decisions.

**Core Entities**

**API**

* Dating app APIs need a 'feed' or 'discover' endpoint (e.g., GET /v1/matches with location query params) that returns potential profiles based on user preferences and location - this is distinct from a 'matched users' endpoint which only shows mutual matches.

* For swipe endpoints, only send the minimal data needed (swipe direction: yes/no) rather than entire user objects in the request body. This reduces payload size and prevents unnecessary data exposure.

* User IDs should be extracted from authentication headers (JWT tokens) rather than passed as URL path parameters for security - this prevents users from manipulating URLs to access or modify other users' data.

* Use PUT instead of POST for updating user preferences because it's an idempotent operation (setting preferences to the same values multiple times produces the same result), which aligns with REST conventions and enables safer retries.

**High Level Design**

* Mobile push notifications require platform-specific services: Apple Push Notification service (APNs) for iOS and Firebase Cloud Messaging (FCM) for Android. Your backend notification service cannot directly reach user devices without integrating with these platform services.

* High-volume write operations (like swipe data with ~2 billion writes/day at 20M DAU × 100 swipes) should be stored in separate tables from core user data. Mixing them degrades read performance on your Users table and makes it harder to scale writes independently.

* Device tokens must be stored in your data model to enable push notifications. When users install your app, the mobile OS provides a unique device token that your backend uses to target notifications through APNs/FCM to specific devices.

* Geospatial indexing (like PostGIS for PostgreSQL) enables efficient location-based queries for proximity matching. Without proper spatial indexes, filtering users by distance would require calculating distances for every user in the database, which doesn't scale.

**Deep Dives**

* Pre-computation of feeds through periodic batch jobs significantly reduces latency for feed-based systems. Instead of querying potential matches in real-time when users open the app, run background jobs (e.g., every few hours) to pre-calculate and store a stack of profiles for each user based on their preferences. This provides instant access to matches while still allowing real-time queries as a fallback.

* Write-through cache policy ensures consistency by writing to both cache and database simultaneously before acknowledging the write. For a dating app's swipe feature, this means checking Redis for reciprocal likes immediately while also persisting to the database, enabling instant match detection without risking data loss.

* Race conditions in distributed systems require explicit handling mechanisms. When two users swipe right on each other simultaneously, you need atomic operations (like Redis transactions with WATCH/MULTI/EXEC) or distributed locks to ensure both swipes are processed correctly and exactly one match notification is created.

​

#### My Answers

**Requirements**

**Q:** What are the non-functional requirements?

> Consistency > Availability - ensure matches are accurate Scalability- handles high traffic (high swipes, sign-ups, matches) during nights and weekends Latency < 200 ms between client and server. Security- sensitive personal and dating information. Compliance- collecting information we are legally allowed to store.

**Core Entities**

**Q:** What are the core entities of the system?

> User Matches Swipe Location

**API**

**Q:** What are the external REST APIs that your system will expose in order to satisfy the functional requirements?

> POST /v1/setPreferences/{userID} -> successCode: number body : { age: number interests: string [] maxDistance: number }\
> GET /v1/matches?latitude=value&longitude=value -> User[]\
> POST /v1/swipe/{targetUserID} -> successCode: number body : { swiped: boolean }

**High Level Design**

**Q:** How will users be able to create a profile and set their preferences?

> The user can set their preferences via the v1/setPreferences API. This will update the values within the postgres table of max distance and preferences. The other values (user ID, personal info, age) are static. Age will update over time.\
> The user can create their profile via a new API called v1/createProfile. This will take parameters like username, personal info (name, email, phone), max distance, and interests\
> Both these APIs call the profile service, which stores the users details in a postgres relational database with a users table with user ID, personal info, age, max distance, and a list of interests

**Q:** How will users be able to get a stack of recommended matches based on their preferences?

> The user calls /v1/matches to call the Match service.\
> The Match service collects others user from the user table in the database within a max distance of its own coordinates. After that, it would sort based on how similar the users preferences are.\
> We would need to add the fields of matches and location to the User table.\
> Introduce geospatial indexing and full text search using ElasticSearch on the DB. We can then sort by location and preferences.

**Q:** How will the system register and process user swipes (right/left) to express interest in other users, showing a match if you swipe right (like) on someone who already liked you?

> The user would call /v1/swipe which will call the Swipe service.\
> If left, the service will remove the target user from MatchFeed in Users table in database update the pair in Swipes table.\
> If right, the Swipe service will remove the target user from MatchFeed and update pair in Swipes table. Now, if both in pair in Swipes table have a boolean of True, this is a Match. Add to Matches attribute in User table and return.\
> Create algorithm in Swipe service to sort Matches by interests and distance, return in that order.

**Q:** The other user needs to know that they have a new match as well, how will your system notify them?

> Add a notification service. When the Swipes service finds a match, it pings the Notification Service with the current user and other user that had a match.\
> The notification service publishes a notification to the other user through a stream. Each user is subscribed to the stream via a publish-subscribe model.

**Deep Dives**

**Q:** How would you design the system to ensure that swipe actions are processed both consistently and rapidly, so that when a user likes someone who has already liked them—even if only moments before—they are immediately notified of the match?

> Implemented a AWS Load Balancer to handle large loads of requests.\
> Implement a Redis cache with a write-through policy on Swipes. We can check the cache for if the other user already has "like" attribute True, we notify the user. Use a Least-Recently Used eviction policy to prioritize swipes that are more popular. Add a time-to-live on swipes so that stale swipes are removed from cache.

**Q:** How can we ensure low latency for feed/stack generation?

> Load balancer will help to process lots of traffic.\
> Use AWS S3 blob to store images where user requests image from server, server returns presigned url, user requests image from S3 using presigned url.\
> We already use a DB with geospatial indexing, but use a CDN cache to store Users geographically.

![Alt text](/Whiteboards/Tinder.png)

​