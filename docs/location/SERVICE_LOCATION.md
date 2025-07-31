## Real-Time Location & Route History Service

### Replication
- **Requirement:** Critically Necessary.
- **Rationale:** A tailored replication approach is needed for each component of this service.
    - **For Kafka:** Built-in topic replication is critical for **durability**. It guarantees that messages are not lost, even if a Kafka broker fails.
    - **For the "Hot" PostGIS Database:** Replication is required for **high availability of reads**. If a leader node fails, users can still view their recent routes thanks to a failover to a replica.
    - **For Cold Storage (S3):** Replication is handled automatically by AWS "under the hood" and requires no additional action from us.
- **Replication Type (for PostGIS):** **Asynchronous, with a Single Leader**.
    - **Asynchronous** replication is the ideal choice because data is already safely persisted in Kafka; we don't need synchronous database write guarantees. This approach does not slow down the high-volume ingestion process.
    - A **single-leader** architecture is simpler and more predictable for this task. All write operations go to one centralized place (the leader node of each shard), which simplifies the system's logic. Leaderless or multi-leader models introduce complexities with conflict resolution that are unnecessary for a simple, high-volume stream of location data.

---

### Sharding
- **Requirement:** Critically Necessary.
- **Rationale:** The data volume (terabytes per year) and ingestion velocity are too large for a single server. Sharding is the only way to scale the system horizontally.
    - **Sharding Strategy:** The most sensible approach is to shard by **`userID`**. This ensures all geopoints belonging to a single user are always stored on the same node (shard).
    - **Benefits:**
        - The most common read query ("get route history for user X") becomes very fast as it only queries a single shard.
        - The write load is evenly distributed across the cluster.
        - Routing logic is simple: the application can use a hash of the `userID` to determine the correct shard.
    - **Rejected Strategies:** Sharding by time would create "hot spots" (all new data written to one shard), and sharding by geolocation would "smear" a single user's data across multiple shards, making history queries slow and complex.
    - **For S3 / Cold Storage:** Partitioning is achieved by using a structured file naming convention (e.g., by date) to optimize future analytical queries.

---

### Transactions (ACID)
- **Requirement:** Definitely Not Required.
- **Rationale:** The key requirement for this service is handling a massive stream of writes (~15k/sec) with minimal latency. Strict ACID guarantees introduce locking and would critically slow down the entire ingestion pipeline.
- We must prioritize **Availability and Latency** over strict data integrity for this category of data. We are prepared to lose a small percentage of geopoints because we can still construct a user's route from the remaining data.
    - **Atomicity** is harmful here. If a batch write of 100 points fails because of one bad record, an atomic transaction would reject all 100. It's better to lose one point than 100.
    - **High levels of Isolation** would be catastrophic for this write-heavy service, as the resulting database locks would create massive queues and performance degradation.

---

### Consistency Model
- **Requirement:** Eventual Consistency.
- **Rationale:** There is no business need for immediate visibility of a user's latest geopoint. Users will view their routes retrospectively. Functionality that notifies users about proximity to others also does not require sub-second accuracy. A small delay is not critical.
- **Strong Consistency** is unsuitable for the same reasons as the ACID model: it would make the system extremely slow (requiring confirmation from a majority of nodes for every single geopoint) and fragile.

---

### Consensus
- **Requirement:** Required.
- **Rationale:** This service is composed of distributed systems (like Kafka and a sharded database cluster) that must operate reliably as a whole.
- **Where it's needed:**
    - **In the Kafka Cluster:** To elect leaders for topic partitions and maintain the overall health and state of the cluster.
    - **In the PostGIS Cluster Management System:** To enable reliable automatic failover. If a leader node of a shard fails, the remaining nodes must reach a consensus to elect a new leader to continue operations.
- Without consensus in these underlying tools, the system would descend into chaos, leading to downtime and data loss.