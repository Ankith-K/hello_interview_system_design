# Design a Ride-Sharing Service Like Uber - Practice Notes

### Design a Ride-Sharing Service Like Uber Guided Practice - April 18, 2026

<!-- EXCALIDRAW_PRACTICE practiceId="cmiibuvl701f208ad6s73ufnw" -->

#### Key Takeaways

**Requirements**

* Ride-sharing systems require strong consistency for matching operations because double-booking (assigning the same driver to multiple riders) creates poor user experience and operational chaos. This is a case where consistency takes priority over availability in the CAP theorem.

* Low latency requirements in ride-sharing are user-facing and time-sensitive - riders expect matches within seconds, not minutes. This drives architectural decisions like geographic partitioning, in-memory caching, and proximity-based algorithms.

* Payment data in distributed systems needs stronger durability guarantees than operational data. Use techniques like write-ahead logging (WAL), synchronous replication, or dedicated payment services with ACID transactions to prevent financial data loss.

**Core Entities**

* In ride-sharing systems, the Ride (or Trip) entity is essential as it represents the actual service transaction. It connects Rider and Driver entities and stores critical operational data like pickup/dropoff locations, timestamps, route details, and status (requested, in-progress, completed, cancelled). Without this entity, you cannot track the lifecycle of a ride or maintain historical records.

* Separating Fare as its own entity (rather than embedding it in Ride) enables flexible pricing models, A/B testing of fare calculations, historical fare analysis, and easier auditing. This separation follows the Single Responsibility Principle and allows the fare calculation logic to evolve independently from ride management.

**API**

* Ride-sharing systems require a driver location update endpoint (e.g., POST /drivers/{driverId}/location) because the matching algorithm needs real-time driver positions to assign nearby drivers to ride requests. Without this, the system cannot determine which drivers are available or closest to riders.

* For REST API design, use GET for retrieving existing resources and POST for creating new resources or performing calculations. A fare estimation endpoint should be POST /fare (not GET) because you're computing a new estimate based on input parameters, not retrieving an existing fare.

* When designing endpoints for status updates (like accepting/rejecting rides), include both the resource identifier in the path (e.g., /rides/{rideId}) and the action/status in the request body (e.g., {'status': 'accepted'}). This follows RESTful conventions and makes the API more maintainable than generic endpoints like /accept-reject-ride.

**High Level Design**

* Driver Location Services need a dedicated data store (like Redis with geospatial indexing) to persist and query location data efficiently. Without this persistence layer, the service cannot perform nearby driver lookups—location data can't just exist in memory without a storage mechanism.

* Redis geospatial indexes (using commands like GEORADIUS) enable efficient proximity queries for location-based matching. This is critical for ride-sharing systems that need to find nearby drivers within a radius in O(log N) time rather than scanning all driver locations.

* Services that check data from other domains (like Driver Location Service checking driver availability) must have explicit database connections shown in architecture diagrams. If a service needs to query the Ride Table for availability status, that connection must be drawn and explained.

* Server-sent events (SSE) enable one-way push notifications from server to client, making them ideal for notifying drivers of new ride requests. Unlike WebSockets (bidirectional), SSE is simpler when you only need server-to-client updates with occasional client-to-server API calls for responses.

**Deep Dives**

* Distributed locks must include a TTL (time-to-live) that automatically expires after a timeout period (e.g., 30-60 seconds). Without TTL, if a driver never responds due to network issues or app crashes, the lock would never release and that driver becomes permanently unavailable in the system.

* Redis geospatial queries use geohashing under the hood, which converts 2D coordinates into a single dimension that enables efficient proximity searches. Commands like GEORADIUS leverage this technique to find nearby drivers without scanning all locations.

* Distributed locks for coordination (like preventing duplicate ride requests) should be managed entirely in-memory by systems like Zookeeper, not persisted in databases like PostgreSQL. Database locks would be too slow and defeat the purpose of fast, distributed coordination across multiple service instances.

​

#### My Answers

**Requirements**

**Q:** What are the non-functional requirements of the system?

> Consistency > availability - Ensure rider driver matches are accurate Scalability- Needs to handle a high volume of rider and driver traffic. Latency- System responds fast to rider and driver requests Fault tolerance- System continues to function in disasters. Durability- We can have some data loss on driver rider history. We cannot lose data on payments. Security- Ensure payment details, driver and rider personal information is secure. Contains sensitive details related to personal information.

**Core Entities**

**Q:** What are the core entities of the system?

> Rider Driver Ride Fare

**API**

**Q:** What are the external APIs that your system will expose in order to satisfy the functional requirements?

> GET /fare -> Ride body : { startLatitude: float, startLongitude: float, endLatitude: float, endLongitude: float }\
> POST /request-ride/{rideID} -> Ride\
> POST /update-driver-location -> statusCode body : { latitude: float, longitude: float }\
> POST /accept-reject-ride/{rideID} -> Ride or None body: { accept: boolean }\
> GET /ride-details/{rideID} -> Ride

**High Level Design**

**Q:** How would you give users a estimated fare based on their start location and destination?

> We have a Driver and Rider as the clients of the system. Authenticated via JWT token. API Gateway authenticates user and routes to appropriate service. The Fare Estimation Service calculates fare amount based on pickup, dropoff, and nearby driver locations. It creates a Ride entry with status estimate, with empty details for driverID The Match Service connects a rider and driver and updates Ride Table The Driver Location Service tracks the location of active drivers by using some cache (later we will implement) to store locations pinged from driver client GPS Uber DB is a PostgreSQL relational DB because the data for our core entities are structured and consistent. We used normalization so the system can refer to attributes via reference ID's. There is a Driver, Rider, and Ride Table that contains the relevant attributes. External Payment Service via payment token. We will handle optimizations later.

**Q:** How will riders be able to request a ride based on the estimated fare?

> The Fare Estimation Service actually creates a Ride with a status of estimated. If the rider requests the ride from here, it uses this same ride (via the Ride ID), to make the request via the POST /request-ride API, which talks to the Match Service

**Q:** How does your system match riders to the best driver for their ride?

> Once a Ride is requested, the Match Service talks to the Driver Location Service to get a list of Drivers closest to the pickup location and willing to drive to the dropoff location (using max radius). The Driver Location Service would check the Ride Table to see if the driverID is currently available (if status is completed) or not in Ride Table. The Driver Location Service uses a Location cache to store the driver's location coordinates. Each location entry in the cache has a time-to-live of 5 seconds and uses FIFO eviction policy so we always have fresh location data. We can enable geospatial indexing in the cache to get relevant location data faster. Use Redis cache.

**Q:** How does your system notify matched drivers and allow them to accept/decline rides?

> Once the Match Service has the sorted list of Drivers from the Driver Location Service (sorted by proximity to pickup and maxRadius), we use server-sent events to ping the Driver client with a notification of the Ride details. From the Ride details, the Driver can accept or reject the ride via the POST /accept-reject-ride API.

**Deep Dives**

**Q:** How can you handle the high write throughput from drivers sending location updates every couple seconds and efficiently perform proximity searches for matching?

> Add a Load Balancer to handle high traffic and horizontally scale our services. As stated earlier, we can use a Redis cache to store location coordinates with a TTL of 5 seconds and eviction policy of FIFO to keep only the freshest, relevant location info. We can also enabled geospatial indexing on the cache to retrieve only relevant locations faster. This will allow us to read from the cache fast, and therefore make matches faster. We cannot store location data in the DB because of the high traffic and it would be expensive to write that much data to our DB.

**Q:** How do we guarantee each driver receives at most one ride request at a time?

> Great question. We can use a lock on a driver if they sent a notification from the Match Service. The lock will store the Driver ID. The Match Service will check if a Driver ID is locked in the Zookeeper lock. This ensures that the Match Service will not send a notification to a locked. When the Driver accepts or rejects the request, we unlock the Driver. Lets also add a TTL of 30 seconds in case the Driver client crashes or is unresponsive, the lock releases by itself.\
> I also added indexing on driver ID in the Ride Table because we read it heavily for availability (from status)

​

​