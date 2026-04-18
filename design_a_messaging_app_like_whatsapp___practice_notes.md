# Design a Messaging App Like WhatsApp - Practice Notes

### Design a Messaging App Like WhatsApp Guided Practice - April 18, 2026

<!-- EXCALIDRAW_PRACTICE practiceId="cmgubz08d09eb08adf6gk9q1g" -->

#### Key Takeaways

**Requirements**

* Messaging platforms like WhatsApp prioritize availability over consistency (AP in CAP theorem) because users need to send/receive messages even during network partitions. Eventual consistency is acceptable since temporary message delays are better than the app being unavailable.

* Guaranteed message deliverability is a core requirement for messaging systems - messages must eventually reach recipients even during temporary network failures. This typically requires persistent queues, retry mechanisms, and acknowledgment protocols.

* Fault tolerance means the system continues functioning even when components fail. For WhatsApp, this requires redundant servers, database replication, and graceful degradation so users can still message even if some services are down.

**Core Entities**

* In messaging systems like WhatsApp, include a 'Client' or 'Device' entity to track multiple devices per user (phone, desktop, web). This is essential for syncing message read/delivery status across devices and managing push notifications to the correct endpoints.

* Multi-device support requires tracking device-specific state separately from user state. Each client needs its own connection identifier, last-seen timestamp, and delivery acknowledgments to ensure messages sync properly when a user switches between devices.

**API**

* Chat APIs need both sending AND receiving mechanisms. Common patterns include: WebSockets (bidirectional), Server-Sent Events/SSE (server-to-client push), or long polling. Without a receive mechanism, users cannot get real-time messages or retrieve messages sent while offline.

* Server-Sent Events (SSE) with Publisher-Subscriber pattern enables real-time message delivery where the server pushes updates to clients over a persistent HTTP connection. This is simpler than WebSockets when you only need server-to-client communication.

* Message history retrieval is a core requirement for chat systems to support the 30-day offline message access use case. This typically requires GET endpoints that fetch past messages from persistent storage, separate from the real-time delivery mechanism.

* Large data endpoints (like chat history) should include pagination parameters (limit, offset, or cursor-based) to prevent returning thousands of messages in a single response, which would cause performance issues and poor user experience.

**High Level Design**

* Track message delivery status with a 'status' field (e.g., 'sent', 'delivered', 'read') in your Message table to ensure reliable offline message delivery and prevent duplicate sends. Without this, comparing timestamps alone can't guarantee which messages were actually received by the client.

* Use presigned URLs with blob storage (like AWS S3) for media attachments in chat systems. This approach offloads binary data processing from your application servers, reduces bandwidth costs, and provides secure, time-limited access to media files without exposing your storage credentials.

* Server-Sent Events (SSE) is an effective choice for real-time message delivery in chat applications because it provides unidirectional server-to-client push notifications over HTTP, which is simpler than WebSockets when you don't need bidirectional communication.

* Store a 'lastTimeSeen' timestamp on users to enable offline message retrieval. When users reconnect, query messages with timestamps greater than their lastTimeSeen value to deliver missed messages, ensuring continuity in the chat experience.

**Deep Dives**

* Multi-client support in messaging systems requires a Device Table with a one-to-many relationship to Users. Each device gets a unique identifier, status (online/offline), device type, and last active timestamp. This allows the system to track which devices belong to each user and route messages accordingly.

* Message delivery in multi-device systems uses a fan-out pattern: when a message arrives for a user, the system queries the Device Table to find all active devices for that user, then delivers the message to each device separately via their respective persistent connections (like SSE or WebSocket).

* Message delivery status must be tracked per device, not per user. This ensures accurate read receipts and synchronization across devices - a message might be delivered to a user's phone but not their laptop, and the system needs to track both states independently.

​

#### My Answers

**Requirements**

**Q:** What are the non-functional requirements of the system?

> Availability > consistency- Ensures messages are sent/received even if not consistent (rare) Latency- Messages should send and receive in less than 200 ms Scalability- System should handle high messaging traffic at peak hours (business, night, weekdays) Security- System should encrypt messages and ensure users are authorized. Compliance- System messages should comply with legal rules. Fault-tolerance- System needs to have backup servers to send/receive messages in case of outages. Durability- Ensure system does not lose messages and sends/receives with little data loss.

**Core Entities**

**Q:** What are the core entities of the system?

> User Message Group Chat Media

**API**

**Q:** What are the external APIs that your system will expose in order to satisfy the functional requirements?

> Assumes user is authenticated and we do not pass user in every call. POST /v1/groupchat -> status code body { phoneNumbers: string[], max 100 message: string, mediaPresignedURL: file }\
> POST /v1/send/{phoneNumber} -> status code body { message: string, mediaPresignedURL: file }\
> The GET APIs are Server-Sent Events using Publisher-Subscriber model.\
> GET /v1/receive -> { message: string senderPhoneNumber: string mediaPresignedURL: file }\
> GET /v1/receiveChat -> { message: string phoneNumbers: string, max 100 mediaPresignedURL: file }\
> GET /v1/receiveHistory -> { chat : { messages: string[] medias: file[] timestamps: numbers[] }[] }

**High Level Design**

**Q:** Design a simple system that allows users to create chats and add participants.

> The user calls API /v1/groupchat\
> The API calls the chat service\
> The Chat service communicates with a postgres relational database to add the message and the phone numbers to the Chats table. This creates a Chat row with a list of phone numbers and list of messages.

**Q:** Update the design to allow users to send and receive messages.

> User calls /v1/send.\
> The Send Message Service updates the Message Table. The Receive Message Service picks up on the recipient added to Message Table and sends the message back to the recipient user using the stream Server-Sent Event /v1/receive.

**Q:** Extend the design to allow users to receive messages later if their client is offline

> Introduced a User Table to save phone number, last time seen, messages, and chats. This table gets updated by chat service and send message service now.\
> Message Table also has timestamp attribute added now.\
> When the user is back online, the Receive Message Service will filter all messages from the User Table that are past the lastTimeSeen attribute using the timestamp attribute in Message.\
> Receive Message Service will then send the messages to the user through a stream Server-Sent Event /v1/receiveHistory\
> Messages in the Chats Table and User Table can be sorted using a timeseries DB now.

**Q:** How can we send/receive media attachments in messages?

> We can introduce a AWS S3 Blob storage solution.\
> On send, the User will request a media file to the Send Message service, and the database will store a presigned URL and store the media file itself in the Blob.\
> On receive, the Service will receive a presigned url from the Receive Message Service, and will use the URL to get media file from Blob.

**Deep Dives**

**Q:** What do we do to handle multiple clients for a given user?

> Our Receive Message Service can use the Device Table to understand for a recipient user, which messages to send to which device based on their status.

![Alt text](/Whiteboards/WhatsApp.png)
​