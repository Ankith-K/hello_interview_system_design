# Design a Local Delivery Service like Gopuff - Practice Notes

### Design a Local Delivery Service like Gopuff Guided Practice - April 18, 2026

<!-- EXCALIDRAW_PRACTICE practiceId="cmjcd70eo01bf08adcz1k01ns" -->

#### Key Takeaways

**Requirements**

* In e-commerce systems, ordering/purchasing operations require strong consistency to prevent overselling - two customers cannot buy the same physical inventory item. This is a business-critical constraint that distinguishes ordering from other operations like search or browsing, where eventual consistency is acceptable.

* Non-functional requirements should be operation-specific in complex systems. While 'availability > consistency' may be true for search and browsing in e-commerce, the opposite is true for checkout/ordering where strong consistency prevents inventory conflicts and maintains data integrity.

* Quantify latency requirements to make them actionable (e.g., search queries should return in <100ms, product page loads in <500ms). Vague statements like 'fast response times' don't provide clear engineering targets for implementation.

**Core Entities**

* In e-commerce and delivery systems, separate 'Item' (the product definition with name, description, price) from 'Inventory' or 'Item Instance' (the actual physical units available at specific locations). This separation allows you to track stock levels per location and handle real-time availability checks.

* Fulfillment Centers need an associated Inventory entity to track which physical items exist at each location. Without this, you cannot answer critical questions like 'Can this order be fulfilled from the nearest warehouse?' or 'How many units of Product X are available in Location Y?'

* The Inventory entity typically includes: item_id (reference to Item), fulfillment_center_id (location), quantity (stock level), and possibly reservation_status (for items in pending orders). This enables both stock management and efficient order routing.

**API**

* In REST APIs, location data like latitude/longitude should be query parameters (e.g., GET /items?lat=37.7&lon=-122.4) rather than in the request body for GET requests. Query parameters are cacheable, bookmarkable, and align with HTTP semantics where GET request bodies are discouraged.

* RESTful resource naming uses plural nouns representing collections (POST /orders) rather than action-oriented names (POST /order-items). This follows the principle that HTTP verbs (POST, GET, PUT, DELETE) already express the action, so the endpoint should identify the resource.

* Pagination parameters (limit, offset or cursor) should be included in list/query endpoints to handle large result sets efficiently. Without pagination, responses can become too large, causing timeouts and poor user experience as the dataset grows.

**High Level Design**

* Distributed locks need timeout mechanisms with exponential backoff to prevent deadlocks when multiple orders try to acquire locks on overlapping items. Without timeouts, two transactions waiting for each other's locks will hang indefinitely.

* Partial failures in distributed systems require rollback or compensation logic. If inventory is decremented but order creation fails, you must restore the inventory count to prevent inventory loss and maintain data consistency.

* Database transactions with SELECT FOR UPDATE can replace distributed locks in single-database systems. Postgres ACID transactions automatically handle both race conditions and partial failures through atomic commit/rollback, simplifying the design compared to external distributed locks.

* When integrating expensive external services (like GPS/mapping APIs), filter data first to minimize calls. Check if inventory exists and filter by approximate distance before making real-time traffic API calls, as calling for every item or distant fulfillment center wastes money and adds latency.

**Deep Dives**

* Using exact latitude/longitude coordinates as cache keys results in poor hit rates because floating-point numbers rarely match exactly. Instead, round coordinates to a grid (e.g., 0.01 degree precision) or use geohash prefixes to create geographic buckets that multiple nearby users will share.

* Read replicas distribute database read load across multiple servers, preventing the primary database from being overwhelmed. This is critical for read-heavy systems where even with caching, cache misses at high scale generate substantial database traffic.

* PostGIS geospatial indexing enables efficient proximity-based queries (e.g., finding nearest fulfillment centers) by using specialized spatial data structures instead of calculating distances for every record in the database.

* Geographic database sharding is particularly effective when users typically query their immediate area - it reduces the data each query scans and naturally distributes load based on user location patterns.

​

#### My Answers

**Requirements**

**Q:** What are the non-functional requirements of the system?

> Availability > consistency- System should be available at all times and can handle some inconsistency due to non-critical business logic. Scalability- System should scale with traffic. Latency- System should respond fast to user requests. Fault tolerance- System should still function in disaster events. Security- Store user payment information.

**Core Entities**

**Q:** What are the core entities of the system?

> Customer Item Order Fulfillment center

**API**

**Q:** What are the external REST APIs that your system will expose in order to satisfy the functional requirements?

> GET /query-items?deliveryTime=1 -> Item[] body : { latitude: float longitude: float }\
> POST /order-items -> Order body: { [ item: Item quantity: int ...this is a list ] }

**High Level Design**

**Q:** How will customers be able to query the availability of items within a *fixed distance* (e.g. 30 miles)?

> The user would talk to our system via the /query-items API endpoint. This would talk to the Item Query Service, which would talk to our Postgres relational DB to get all fulfillment centers within a 30 mile distance between the user location and fulfillment center location (we know both these values). From there, the service, would query the inventory and items for the selected fulfillment centers.

**Q:** How can we extend the design so that we only return items which can be delivered within *1 hour*?

> Knowing the locations of the user and fulfillment center locations, the system can leverage a third-party GPS service like Google Maps API to get drive time based on current traffic indicators. We would call this external service for every fulfillment center IF the inventory is non-zero.

**Q:** How will customers be able to place orders?

> The user will talk to our system via the /order-items endpoint and pass in their order information (which items and how much). This endpoint will talk to the Order service, which will lock the entries where the item ID matches the user's ordered items so that another user cannot order an item where the quantity at the fulfillment center may not be accurate. The order service will update the inventory table accurately. To minimize inventory outages, the service will attempt to equalize the quantity of items it is ordering from various fulfillment centers. This can also be deprioritized if distance is more important (such as in peak rush hour), The Order service will also query the Item table to get the price, and based on quantity, get a payment total. The order service will communicate with an external payment service using the User's payment token.

**Q:** How can we ensure users cannot order items which are unavailable? How do we avoid race conditions?

> The Order service will use distributed locks (via Zookeeper) to lock the entries in the inventory table which are relevant to the user's order request. This would be locked once the order service decides which fulfillment center to use and how much (quantity) for each item. By locking, we ensure another user cannot access an item in a fulfillment center which may not have quantity. The timeout for the lock would from when the order service decides which fulfillment centers to use to fulfill the order to payment confirmation, or 1 minute, whichever is shorter. If the order cannot be completed, the order service will revert the inventory table data. The lock is still effective so other users would not have manipulated the data causing inconsistency.

**Deep Dives**

**Q:** How can we ensure availability lookups are fast and available?

> We can use a cache (Redis) with a write-aside strategy and least-frequently used eviction policy. Since the query is pulling items based on location, we can create a key-value pair with key as the user's location and value as a list of the fulfillment centers. We can create a general location box and not use exact latitude, longitude as this fluctuates. This way, the query service can shortcut finding which fulfillment centers are within our location criteria. This would also ensure user's who query the items the most would have quick reads, thanks to our eviction policy. We can also implement a time-to-live of 5 minutes to keep our cache small. We can index on an inventory's fulfillment center ID to get relevant inventories fast. We can index on an item's ID to get relevant items fast. We can also introduce geospatial indexing via PostGIS to organize Fulfillment center table data by location and allow the system to query relevant fulfillment centers faster. Lastly, we can shard our DB by a region attribute we add to our fulfillment centers, because we typically will be getting fulfillment centers in the area.

![Alt text](/Whiteboards/Tinder.png)

​

​