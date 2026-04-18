# Design a URL Shortener Like Bit.ly - Practice Notes

### Design a URL Shortener Like Bit.ly Guided Practice - April 18, 2026

<!-- EXCALIDRAW_PRACTICE practiceId="cmgtnv8b006ll08adm0j64inn" -->

#### Key Takeaways

**Requirements**

* **Uniqueness is a core functional requirement for URL shorteners**: Each short URL must map to exactly one long URL to prevent incorrect redirections. Without this guarantee, users could be sent to the wrong destination, breaking the fundamental contract of the service.

* **Non-functional requirements describe HOW the system behaves, not WHAT it does**: Latency, availability, and scalability are non-functional (system qualities). URL length handling, character support, and data validation are functional requirements (features/capabilities). For example, 'support URLs up to 2048 characters' is functional, while 'redirect requests in <100ms' is non-functional.

* **Quantify latency requirements for URL shorteners**: Instead of 'fast redirects,' specify measurable targets like 'P99 latency <100ms for redirects' or 'P95 latency <50ms.' URL shorteners are latency-critical since users expect instant redirects, and specific numbers make requirements testable and drive architectural decisions.

**Core Entities**

* In URL shortener data models, avoid redundant entities: the 'Short URL' entity should store both system-generated codes and user-defined aliases (custom short URLs) in the same table, using a field like 'is_custom' or 'alias_type' to distinguish them. This prevents data duplication and simplifies lookups.

* URL shortener entities should maintain a one-to-many relationship between User and Short URLs (one user can create many short URLs), and a one-to-one relationship between Short URL and Long URL (each short code maps to exactly one destination, though the same long URL can have multiple short codes).

* Essential fields for a URL shortener's Short URL entity include: short_code (unique identifier), long_url (destination), user_id (creator), created_at (timestamp), expiration_date (optional TTL), and click_count (analytics). This covers both core functionality and common feature requirements.

**API**

* URL shorteners should use path parameters (GET /{shortCode}) rather than query parameters (GET /longURL?shortURL) because the short code IS the resource identifier, and this allows the shortened URL to be clean and directly accessible (e.g., bit.ly/abc123).

* Long URLs should be passed in the POST request body, not as query parameters, because URLs can exceed the maximum query string length (typically 2048 characters in browsers) and may contain special characters that require encoding.

* URL shorteners should return HTTP 301 (permanent redirect) or 302 (temporary redirect) status codes when accessing a short URL. Use 301 for better caching and SEO, or 302 if you need to track every click without browser caching.

* RESTful API design uses resource-based naming like /urls/{shortCode} rather than action-based naming like /longURL, making the API more intuitive and following standard REST conventions where URLs represent resources, not actions.

**High Level Design**

* In a URL shortener, the service returns an HTTP redirect status code (301 or 302) to the client browser, NOT making the request itself to the destination. This is critical because having the server make the request would break user sessions, cookies, and create unnecessary load on your system.

* Use HTTP 302 (temporary redirect) instead of 301 (permanent redirect) for URL shortening when you need click tracking. 301 responses are permanently cached by browsers, which prevents your service from tracking metrics on subsequent visits to the same short URL.

* URL shortener database schemas should include user_id (to track ownership), creation_timestamp, and expiration_date fields. These enable user management, analytics, and automatic cleanup of expired URLs, making the system production-ready rather than just a proof-of-concept.

**Deep Dives**

* URL shorteners need a specific generation algorithm: either (1) a global counter with Base62 encoding for guaranteed uniqueness, or (2) hash functions (like MD5/SHA256) with Base62 encoding plus collision detection and retry logic. Base62 uses [a-zA-Z0-9] to create compact, human-readable URLs.

* Base62 encoding is essential for URL shorteners because it converts numeric IDs into short strings using 62 characters (a-z, A-Z, 0-9). A 6-character Base62 string provides 62^6 = ~56 billion unique URLs, while a 7-character string provides ~3.5 trillion possibilities.

* For collision handling in hash-based URL generation, implement a write constraint at the database level (unique index on short_url) and retry with a modified input (e.g., append a counter or salt) when collisions occur. This ensures uniqueness without complex distributed locking.

* Read-heavy systems like URL shorteners (10k redirects/sec) benefit from aggressive caching with high TTL values since shortened URLs rarely change. Cache hit ratios of 80-90% are typical, reducing database load by an order of magnitude.

​

#### My Answers

**Requirements**

**Q:** What are the non-functional requirements for this system?

> Handle long lengths of URLs Availability > consistency Scalable, handles many requests Fast return of URLs

**Core Entities**

**Q:** What are the core entities of the system?

> User Long, original URL Short, new URL Alias

**API**

**Q:** What are the main API endpoints that this system will need?

> POST /shorten?longURL -> shortURL: string body : { alias: string? }\
> GET /longURL?shortURL -> longURL: string

**High Level Design**

**Q:** How will users shorten a URL? Don't worry about the algorithm for generating new short URLs. We will get to that later.

> User connects to Load Balance and API gateway to address horizontal scaling and authentication. URL service shortens URL Post gres relational DB stores one URL table with attributes short URL, long URL, alias Introduce cache to get long URLs that are retrieved often. Introduce indexing strategy to store URL's as key-value pair (key - short URL, value - long URL) for fast retrievals Introduce distributed locking to make sure pair cannot be used multiple times.

**Q:** How will users be redirected from a short URL to the original long URL?

> User enters short URL URL service checks if short URL is in DB If so, gets the long URL (key-value pair), and executes an HTTP request to correct domain/website If not, returns some form of error to user

**Deep Dives**

**Q:** How can we ensure that each shortened URL is short and unique?

> Introduce locking while shortening a URL to ensure key-value pair is not used twice. Introduce key-value pair in data base on short URL's for unique keys

**Q:** How can we ensure that redirects are fast?

> *No answer recorded*

**Q:** How can your system scale to support 10k redirects (reads) per second?

> Add horizontal scaling to support multiple machines processing URLs in parallel.

![Alt text](/Whiteboards/Bitly.png)

​