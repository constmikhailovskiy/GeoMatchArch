### Services & Database Choices

#### 👤 [Authentication & User Accounts](auth/SERVICE_AUTH.md)

Responsible for user registration, login, and session management.

- **Database Choice:** `Relational DB (PostgreSQL / MySQL)`
- Requires strong **ACID guarantees** for user records, password hashes, and session management.
- A simple schema with `JOIN` capabilities is sufficient.

---

#### 🖼️ [Profiles & Interests](profile/SERVICE_PROFILE.md)

Responsible for user profile management, including bio, interests, and photos.

- **Database Choice:** `Relational DB (PostgreSQL / MySQL)`
- Profile data (bio, interests, etc.) fits naturally into a normalized relational schema with many-to-many relationships (e.g., `user_interests`).
- Allows for easy enforcement of foreign-key constraints and efficient lookups via indexed joins.

---

#### 📍 [Real-Time Location & Route History](location/SERVICE_LOCATION.md)

Handles real-time location updates, route history, and geospatial queries.

- **Database Choice:** `Geo-enabled relational store (PostgreSQL + PostGIS)` for Hot Data (last 30 days) + `Object Store (S3)` for Cold Data.
- **PostGIS** allows for efficient storage and querying of location points using specialized GiST indexes.
- An LRU/TTL-like logic is managed at the application level to move data from hot to cold storage.

---

#### ☕ [Places](places/SERVICE_PLACES.md)

Manages a catalogue of places (e.g. coffee shops, galleries, restaurants, yoga studios, etc) with geospatial data.

- **Database Choice:** `PostGIS/PostgreSQL` + `Redis Cache`
- Query results for nearby places are cached in **Redis** to reduce database load.
- Can be populated with data from third-party APIs (e.g., Google Places) and stored locally in PostGIS.

---

#### 🔒 [Privacy Settings & Data Exports/Deletes](data-privacy/SERVICE_DATA_PRIVACY.md)

Manages user privacy settings, personalization & analytics consents and data export/delete requests.

- **Database Choice:** `Relational DB (PostgreSQL)`
- Privacy flags (`share_location`, `invisible_mode`) and data management requests are stored alongside user records.
- Relies on database features like cascade deletion to reliably handle data removal requests as required by GDPR.

---

#### 💬 [Direct Messaging (DMs / Chat)](messages/SERVICE_MESSAGES.md)

Handles real-time direct messaging between users, including message history.

- **Database Choice:** `NoSQL Wide-Column Store (Cassandra)`
- Optimized for **very high write throughput** and fast reads of recent messages in a conversation (fan-out reads).
- Horizontally scalable and partitioned by `conversation_id` to handle massive message volumes without complex joins.
- An LRU-like algorithm can be used at the application level to manage conversation history.

---

#### ❤️ [Matching & Recommendations](matches/SERVICE_MATCHES.md)

Responsible for finding and recommending users based on interests, location, and other criteria.

- **Database Choice:** `Search Engine (Elasticsearch)`
- Ideal for **full-text search** and complex queries on user profiles and interests.
- Supports **scoring** and **ranking** of matches based on user preferences and location proximity.
- Built-in replication and sharding capabilities allow for horizontal scaling as the user base grows.
- Can be integrated with a relational database for user data storage, using Elasticsearch as a secondary index for fast lookups.
- Supports complex queries like "find users with similar interests within 5 km".

#### 💳 Monetization & Payment Transactions

Handles payment processing, subscription management, and transaction history.

- **Database Choice:** `Relational DB (PostgreSQL)`
- Financial transactions demand **strict consistency**, auditability, and easy reporting, making an ACID-compliant DB essential.
- A relational ledger table (e.g., with `transaction_id`, `user_id`, `amount`) is easy to index and join with user data for reporting.

---

#### ⚡ Caching & Session Stores
- **Database Choice:** `In-Memory Key-Value Store (Redis)`
- Ideal for storing ephemeral, frequently accessed data with short TTLs (Time To Live).
- Example keys: `latest_location:<user_id>`, `matches:<user_id>`, `places:<geohash>:<category>`.

### Data Storages Strategy Brief Summary

| Domain | Database Type | Replication | Sharding | Transactions |
| :--- | :--- | :--- | :--- | :--- |
| 👤 **Auth & User Accounts** | `Relational (PostgreSQL)` | Yes | No | **Yes** (ACID) |
| 🖼️ **Profile Management** | `Relational (PostgreSQL)` | Yes | No | **Yes** (ACID) |
| 📍 **Location History** | `PostGIS` + `S3 Object Store` | Yes | **Yes**<br>(by `userID`) | No |
| ❤️ **Matching** | `Elasticsearch` | Yes (Built-in) | **Yes** (Built-in) | No |
| ☕ **Places Catalogue** | `PostGIS` + `Redis` | Yes | No | **Yes** (for DB writes) |
| 🔒 **Privacy Settings** | `Relational (PostgreSQL)` | Yes | No | **Yes** (ACID) |
| 💬 **Direct Messaging** | `NoSQL (Cassandra)` | Yes (Built-in) | **Yes** (Built-in)<br>(by `conversation_id`)| No (Row-level only)|
| 💳 **Payment Transactions** | `Relational (PostgreSQL)` | Yes | No | **Yes** (ACID) |
| ⚡ **Cache & Session Store**| `In-Memory (Redis)` | Yes | Yes (Clustering) | No |

### Cross-Service Communication

The architecture employs two primary communication patterns, chosen based on the specific requirements of the interaction:

- *Synchronous Communication* (Request/Response) for immediate needs.
- *Asynchronous Communication* (Event-Driven) for decoupling, resilience, and high-throughput tasks.

Here's a high-level overview diagram of how each service communicates with others:

[Service Communication Diagram](../diagrams/geomatch_cross_service_communication.png)

---

#### 👤 Auth Service

The Auth service is the gatekeeper for user identity and is designed for high consistency.

-   **Incoming Communication:**
    -   Receives **synchronous** `POST` requests from the **API Gateway** for `/register`, `/login`, and `/refresh`. The client waits for an immediate response (success with tokens, or an error).

-   **Outgoing Communication:**
    -   **During registration**, it makes **synchronous** internal API calls (gRPC/REST) to:
        1.  `Profile Service`: To create a new user profile record.
        2.  `Privacy Service`: To create a record with default privacy settings.
    -   It also **asynchronously consumes** the `UserDeletionRequested` event from Kafka to delete user credentials.

-   **Rationale:**
    -   Synchronous outgoing calls during registration are crucial to ensure the process is **atomic**. If a profile cannot be created, the entire user registration must fail to prevent creating "orphan" accounts.
    -   Asynchronous consumption of deletion events decouples the service, ensuring it can handle cleanup without being directly called by the `Privacy Service`.

---

#### 🖼️ Profile Service

The Profile service is the source of truth for user-descriptive data and broadcasts changes to the system.

-   **Incoming Communication:**
    -   Receives **synchronous** `GET`/`PUT`/`POST` requests from the **API Gateway** for managing a user's own profile.
    -   Receives **synchronous** internal `POST` requests from the **Auth Service** during registration.

-   **Outgoing Communication:**
    -   **Does not make direct calls** to other services.
    -   It **asynchronously publishes** changes to the `Profile DB`. A **Change Data Capture (CDC)** tool monitors the database's transaction log and publishes a `ProfileUpdated` event to a Kafka topic, which the Matching Service consumes.

-   **Rationale:**
    -   This is a classic event-driven approach that completely decouples the `Profile Service` from its consumers. It doesn't need to know which other services (like `Matching`) are interested in profile updates. This makes the system highly extensible and resilient.

---

#### 📍 Location Service

This service uses a hybrid communication model to handle high-throughput writes and user-facing reads.

-   **Incoming Communication (Producer API):**
    -   Receives high-volume **asynchronous-style** `POST /location/ping` requests from the **API Gateway**. The service immediately returns a `202 Accepted` response.
    -   Receives **synchronous** `GET /location/history` requests from the **API Gateway**.

-   **Outgoing Communication (Producer API):**
    -   **Asynchronously publishes** a `location_ping` event to Kafka for every incoming ping.

-   **Internal Communication (Consumer):**
    -   The **Location Consumer** component **asynchronously consumes** `location_ping` events from Kafka.
    -   After processing, the Consumer can **asynchronously publish** new, higher-level events (e.g., `user_location_pattern_updated`) back to Kafka for other services like `Matching` to consume.

-   **Rationale:**
    -   The async "fire-and-forget" pattern for location pings is essential for handling the massive write load without blocking the client.
    -   Synchronous reads for history are necessary to provide an immediate response to the user.

---

#### ❤️ Matching Service

This service is primarily an asynchronous consumer that builds a read-optimized model of user data.

-   **Incoming Communication:**
    -   Receives **synchronous** `GET /matches` requests from the **API Gateway**.
    -   **Asynchronously consumes** events from Kafka topics, including:
        -   `ProfileUpdated`: To get the latest user interests and bio.
        -   `user_location_pattern_updated`: To get aggregated location insights.

-   **Outgoing Communication:**
    -   Makes no direct calls to other services.
    -   It may **asynchronously publish** events like `NewMatchCreated` to Kafka, which a `Notification Service` could consume to send push notifications.

-   **Rationale:**
    -   This event-driven approach makes the `Matching Service` extremely resilient and scalable. It does not depend on the availability of the `Profile` or `Location` services to function. It operates on its own local, eventually consistent copy of the data stored in Elasticsearch, which is optimized for the complex queries it needs to perform.

---

#### 🔒 Privacy Service

This service manages user consent and orchestrates the account deletion process.

-   **Incoming Communication:**
    -   Receives **synchronous** `GET`/`PUT` requests from the **API Gateway** for managing privacy settings.
    -   Receives **synchronous** internal `POST` requests from the **Auth Service** during registration.
    -   Receives a **synchronous** `POST /privacy/delete-account` request to initiate user data deletion.

-   **Outgoing Communication:**
    -   Upon receiving a deletion request, it **asynchronously publishes** a `UserDeletionRequested` event to Kafka.

-   **Rationale:**
    -   Account deletion is a long-running, fan-out process. Using an asynchronous event is the only reliable way to ensure that all services are notified and can perform their cleanup tasks independently, without the `Privacy Service` needing to know about all of them or manage their potential failures.



#### ☕ Places Service

The Places service is a self-contained catalogue, primarily serving read requests.

-   **Incoming Communication:**
    -   Receives **synchronous** `GET` requests from the **API Gateway**, typically for `/places/nearby`. These requests include the user's current geographic coordinates.

-   **Outgoing Communication:**
    -   Generally, this service **does not communicate with other internal services**.
    -   It may make **synchronous** outgoing API calls to a **third-party service** (in our case, Google Maps API) to fetch new data if a place is not found in its local database cache.

-   **Rationale:**
    -   The service's role is simple: accept coordinates and return a list of places. This is a classic synchronous, request-response pattern. The user needs this information immediately to see what's around them, so an asynchronous flow would be inappropriate.

---

#### 💬 Direct Messages Service

This service uses a synchronous API for clients but an asynchronous, event-based approach for notifying users.

-   **Incoming Communication:**
    -   Receives **synchronous** `POST /messages` requests from the **API Gateway** when a user sends a message. The client waits for a confirmation that the message has been accepted by the server.
    -   Receives **synchronous** `GET /conversations/{id}` requests to fetch a chat's history.

-   **Outgoing Communication:**
    -   After successfully persisting a new message to its Cassandra database, the service **asynchronously publishes** a `NewMessageReceived` event to a Kafka topic.
    -   This event contains details like the `conversation_id`, `sender_id`, `recipient_id`, and `message_preview`.

-   **Rationale:**
    -   The core logic of sending and receiving messages must be fast and reliable, which is handled by the synchronous API.
    -   However, the task of **notifying the recipient** (e.g., via a push notification) should be decoupled. The `Messages Service`'s job is to save the message, not to manage push notifications. By publishing an event, it allows a separate `Notification Service` to handle the complex logic of sending notifications without blocking or slowing down the core messaging functionality. This makes the system more resilient and scalable.








