# Design a News Aggregator like Google News - Practice Notes

### Design a News Aggregator like Google News Guided Practice - April 18, 2026

<!-- EXCALIDRAW_PRACTICE practiceId="cmgtp5shp07aa09ado3dfnqqn" -->

#### Key Takeaways

**Requirements**

* Non-functional requirements should include specific, measurable targets (e.g., 'support 10M concurrent users' or 'p99 latency under 200ms') rather than generic statements like 'needs to scale' - this gives concrete design constraints and helps you choose appropriate technologies.

* News aggregators face specific scaling challenges like traffic spikes during breaking news events (10-100x normal load), which requires designing for burst capacity through auto-scaling, CDN caching, and read replicas rather than just steady-state throughput.

* When specifying latency requirements, distinguish between different operations: feed pulling/ingestion latency (backend, can be 200ms-1s), user-facing read latency (should be <100ms), and article refresh rates (can be minutes) - each has different architectural implications.

**Core Entities**

* In system design entity identification, distinguish between entities (data objects to be stored like User, Article, Publisher) and the system itself. The platform/application is not an entity - it's the container that manages entities. Only list persistable data objects that have attributes and relationships.

**API**

* REST API query parameters must be separated by ampersands (&), not multiple question marks. The correct format is `/api/resource?param1=value1&param2=value2&param3=value3`. Only the first parameter follows a question mark; subsequent parameters use ampersands to maintain proper URL encoding standards.

**High Level Design**

* RSS feed aggregation requires a scheduled background worker system that polls publisher feeds independently of user requests. This ensures fresh content is continuously collected and available, rather than fetching on-demand which would create unacceptable latency for users.

* RSS feeds are XML documents that require dedicated parsing logic to extract structured data (title, content, publication date, URL, media). Your system needs an RSS parser component to transform XML into database records, not just raw HTTP requests to publishers.

* Managing thousands of publisher feeds requires tracking metadata: RSS feed URL, last poll timestamp, and polling frequency per publisher. This prevents wasting resources on unchanged feeds and ensures high-velocity publishers are checked more frequently than slow-updating ones.

* Geospatial indexing (like PostGIS for PostgreSQL) enables efficient location-based queries for regional news feeds. Store articles with latitude/longitude coordinates and use spatial indexes to quickly filter articles within a geographic radius or bounding box based on user location.

**Deep Dives**

* **Cursor-based pagination using timestamp + article ID** prevents inconsistent results when new content is published during scrolling. Simple offset/counter pagination causes users to see duplicates or miss articles when the dataset changes, while cursor-based pagination maintains a stable reference point.

* **Change Data Capture (CDC) with pre-computed feeds** is superior to TTL-based caching for real-time applications. Instead of waiting for cache expiration, CDC events trigger immediate updates to pre-computed feeds stored as Redis sorted sets, providing consistent sub-100ms response times and avoiding thundering herd problems during TTL expiration.

* **Publisher webhooks with fallback polling** is the optimal content ingestion strategy. Webhooks allow publishers to push content immediately upon publication (seconds vs minutes), while maintaining RSS polling and web scraping as fallbacks for non-cooperative publishers ensures complete coverage.

* **Server-sent events (SSE) are for server-to-client communication**, not for polling external publishers. SSE enables servers to push updates to connected clients, while polling is a client-initiated request pattern - these are fundamentally different communication models.

​

#### My Answers

**Requirements**

**Q:** What are the non-functional requirements?

> Availability > consistency Needs to scale Latency- should pull article feeds fast

**Core Entities**

**Q:** What are the core entities of the system?

> User Source publishers and RSS feeds Article News Aggregator Platform

**API**

**Q:** What are the external REST APIs that your system will expose in order to satisfy the functional requirements?

> GET /v1/newsfeed?latitude?longitude?pagination -> Article[]

**High Level Design**

**Q:** How will your system collect and store news articles from thousands of publishers worldwide via their RSS feeds?

> The aggregator service talks to source publishers to get relevant articles periodically. This is not user-driven. The aggregator service polls from source publishers on its own. It uses server-sent events in the form of long polling.\
> Article data is stored in database article table. Article table holds publisher, location, article data, presigned URL, and timestamp.\
> Publisher data is stored in publishers table. Keep list of publishers and last checked to know which publisher to poll from.\
> The aggregator service uses server-sent events to poll news periodically from source publishers\
> The DB uses geospatial indexing strategy to sort articles by location.\
> We can also use a timeseries DB to store articles by times since we want to show recent articles.\
> Use CDN to cache news geographically. CDN stores most recent news for eviction policy\
> Use blob storage to store media content associated with article- images, videos\
> Use

**Q:** How will your system show users a regional feed of news articles?

> Use a CDN to cache articles geographically. The CDN can also use a most recent eviction policy to keep the most recent articles in its cache.\
> Use the geospatial indexing strategy to sort article data by location.\
> Add user to system. User makes request through API.\
> Use API we defined earlier to call service\
> Introduce load balancer to scale user traffic and API gateway for security

**Deep Dives**

**Q:** How will users be able to efficiently scroll through the feed 'infinitely'?

> From the client side, keep a pagination counter and make a new API call with updated pagination counter to allow for infinite scroll.\
> Introduce user cache with each users current article to support returning the next articles.

**Q:** How do you ensure that users feeds load quickly (< 200ms)?

> Introduce a CDN to cache geographically relevant articles to the user.\
> Introduce a load balancer and horizontal scaling to parallely serve multiple requests.\
> Give articles in cache a time-to-live to ensure the most recent articles are in cache.\
> Use write-through cache, when DB is updated after aggregator service gets new articles. Update cache.\
> Use Redis cache because it supports write-back policy.\
> When the aggregator service gets new articles, temporarily invalidate the cache because it no longer has the newest articles.

**Q:** How do we ensure articles appear in feeds within 30 minutes of publication? Even for publishers that don't have RSS feeds.

> Instead of using a polling format, we can use streams where our aggregator service subscribes to the source publishers. Whenever a new article is published our aggregator would automatically receive it through the stream. Use Kafka for streams.\
> We can use web scraping to manually pull article data. This would be periodically, but every 30 minutes is the maximum since we want articles to appear in feeds within 30 minutes of publication.

**Q:** How do you efficiently deliver thumbnail images for millions of news articles in user feeds?

> I already addressed this earlier.\
> We should use a blob storage with a presigned URL. The blob storage would host the same image in different sizes for different devices.\
> The user would request the presigned URL from the server. The server returns the presigned URL, and the user fetches the thumbnail file from the Blob storage using the presigned URL.\
> Deliver images through CDN from S3.\
> Use AWS cloudfront for CDN.\
> Use AWS S3 for blob storage.\
> Our aggregator service would determine the platform user is using.

![Alt text](/Whiteboards/Google_News.png)

​

​