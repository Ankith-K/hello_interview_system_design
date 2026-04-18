# Design Facebook's News Feed - Practice Notes

### Design Facebook's News Feed Guided Practice - April 18, 2026

<!-- EXCALIDRAW_PRACTICE practiceId="cmigbkj2203na08adeou4misi" -->

#### Key Takeaways

**Requirements**

* When defining latency requirements in system design, always quantify them with specific numbers (e.g., '< 500ms for feed loads' or 'p99 latency < 1s'). This enables concrete architectural decisions like choosing between databases, caching strategies, and CDN placement.

* Clarify scale metrics by specifying the type of users: Daily Active Users (DAU), Monthly Active Users (MAU), total registered users, or peak concurrent users. '2B DAU' vs '2B total users' leads to vastly different infrastructure sizing - DAU directly impacts your real-time capacity planning.

* In social media systems, prioritize availability over consistency because users can tolerate seeing slightly stale data (eventual consistency), but cannot tolerate the service being down. This justifies using AP systems (from CAP theorem) like Cassandra over CP systems like traditional RDBMS with strong consistency.

**Core Entities**

* Facebook News Feed operates on a Follow model, not a Friend model. While Facebook has bidirectional friendship relationships, the News Feed content distribution is based on unidirectional follow relationships - meaning users see posts from accounts they follow, regardless of whether those accounts follow them back. This is why 'Follow' is a core entity but 'Friend' is not necessary for News Feed design.

**API**

* RESTful API endpoints should use resource nouns (e.g., POST /v1/posts) rather than action verbs (e.g., POST /v1/create). This follows REST principles where HTTP methods (GET, POST, PUT, DELETE) already convey the action being performed on the resource.

* For feed pagination at scale, use cursor-based pagination (e.g., GET /v1/feed?cursor=timestamp&pageSize=10) instead of offset-based pagination. Cursor-based pagination performs better because it doesn't require counting/skipping rows as the dataset grows, and it handles real-time data insertion more reliably without duplicates or missing items.

**High Level Design**

* For many-to-many relationships like follows, never store arrays in a single user record. Instead, create a separate junction table (e.g., Follow table with follower_id and followee_id pairs) to enable efficient querying in both directions and handle millions of relationships without scalability issues.

* Pagination requires a cursor mechanism to track position, not just a limit. Use timestamps or post IDs as cursors that clients pass with each request (e.g., GET /v1/feed?cursor=xxx&limit=50) so the server knows which posts have already been seen and can fetch the next batch.

* Database indexes on foreign key columns (like follower_id and user_id) are critical for query performance in relational databases, especially as the number of joins and filtering operations grows with scale.

* Fan-out on read (querying follows at read time, then fetching posts) becomes slow for users who follow many people because you're doing expensive joins and filtering at read time. Acknowledging this tradeoff shows systems thinking, even if you don't immediately optimize it.

**Deep Dives**

* Fan-out on write precomputes feeds when posts are created (writing to all followers' feed caches), enabling fast reads but causing write amplification. Fan-out on read queries followed users at read time, keeping writes simple but making reads slow. For social feeds with 500ms latency requirements, fan-out on write is typically necessary for non-celebrity users.

* Write amplification occurs when one write triggers many secondary writes. In social feeds, a celebrity with 10M followers posting would require 10M feed cache updates. The solution is a hybrid approach: use fan-out on write for normal users (< 100K followers) and skip precomputation for celebrities, instead merging their posts at read time.

* In a hybrid fan-out approach, the feed read must merge two data sources: (1) the precomputed feed from cache containing posts from normal users the reader follows, and (2) real-time queries for recent posts from celebrity accounts the reader follows. Without this merge step, celebrity posts would never appear in user feeds.

* Write-back caching means writes go to cache first and are asynchronously persisted to the database later (fast writes, risk of data loss). Write-through means writes go to both cache and database synchronously (slower writes, no data loss). Read-through means the cache automatically loads from the database on cache misses. These are distinct caching patterns with different tradeoffs.

​

#### My Answers

**Requirements**

**Q:** What are the non-functional requirements for this system?

> Availability > consistency Should be able to scale to handle 2B users Low latency- users should be able to perform requests fast High fault tolerance - System stays up even if things go wrong Security- Encrypt data to ensure data is protected Compliance- Ensure system complies with legal rules

**Core Entities**

**Q:** What are the core entities of the system?

> User Post Follow Friend

**API**

**Q:** What are the main REST APIs that this system will need?

> POST /v1/create -> statusCode body : { caption: string media: JPEG[] tags: User[] }\
> POST /v1/follow/{user} -> statusCode\
> GET /v1/feed?pagination=value -> Post[]

**High Level Design**

**Q:** How will users be able to create text posts?

> Id create a User box to represent the user.\
> The post service would handle the API call with the post details.\
> We would check the character length of the post to see if it is below the character limit.\
> The post service would make a call to the post database and the post would be stored in the Post Table

**Q:** How will users be able to friend/follow people?

> I would introduce an API gateway to route the user's API call to the correct service and handle user authentication. We can use AWS API gateway here. I would create a Follow service to consume the user's request. The follow service would call the database and specifically the Follow Table. The follow table store each entry as a pair of follower ID and Followee ID, which are the same data category as User ID

**Q:** How will users be able to view a feed of posts from people they follow? Don't worry about pagination yet.

> The client's request will be routed through the API gateway to the Feed Service. The Feed Service will then talk to the database's follow table to get all followee IDs where the client ID matches the follower ID. From there, the Feed Service will look at the Post Table and collect all Posts where the User ID matches the followee ID, because this will get all the posts of the users the client is following. The Feed Service would sort these posts by timestamp and then return to the user. For optimizations, we can introduces indexing on Follower ID and User ID for fast retrievals.

**Q:** How will users be able to page through their feed? (i.e., infinite scroll - scroll down to see older posts without reloading the entire page)

> Let's update our GET /v1/feed API with cursor as a query parameter to track what posts to return. With pagination, the feed service would sort and return the number of posts up the pagination max. It would also return a cursor so on the next feed request, the feed service would return the max posts older than the cursor.

**Deep Dives**

**Q:** How do we efficiently read feeds for users who follow thousands of accounts while maintaining low latency?

> Use a Redis read-through cache to store feeds that are being built in the background whenever a post is created. Update the feeds for the relevant users who are following the client who made the post. We can use queue to poll in requests to build precomputed feeds whenever a post is created. This is stored in the precomputed feed cache. We can use data sharding to split our Post Table by User ID and Follow Table by Follower ID for faster look-ups. Index Follower ID and User ID for fast look-ups. Use a Timeseries DB to sort Posts by timestamp, so that the Feed Service does not have to sort. Use Blob storage (AWS S3) to store Post media content. Store pre-signed URL within the Post Table, and client retrieves the post through the Blob storage. This is assuming we are storing media content.

**Q:** How do we avoid write amplification when a celebrity user with millions of followers creates a post that needs to update millions of other users feeds?

> So the Feed Service would only queue precomputed feed jobs if the celebrity has below a certain follower threshold.\
> Further, in the Follow Table, we can keep track of a last interacted timestamp. Then, we can precompute feeds for the users who most interact with client?

![Alt text](/Whiteboards/Google_News.png)
​