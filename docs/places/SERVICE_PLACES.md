## Places Service

### Replication
- **Requirement:** Required.
- **Replication Type:** **Asynchronous with a Single Leader**.
- **Rationale:**
    - The Places service is very **read-heavy**, meaning read operations will far outnumber write operations. Replication is essential for two main reasons:
        1.  **High Availability:** If the primary node fails, a replica can take over its role.
        2.  **Read Scalability:** We can distribute thousands of "find nearby places" requests across multiple replica nodes to avoid overloading the primary node.
    - **Asynchronous** replication is appropriate because we don't need strong consistency. There is no need to slow down the infrequent write operations by waiting for confirmation from replicas. An update to the list of places doesn't need to be instantaneous across the system; a delay is acceptable.

---

### Sharding
- **Requirement:** Should be planned for, considering the need to scale to different countries.
- **Rationale:** Assuming the service will become global and the database will grow significantly, **geo-sharding** (partitioning by geographic region, e.g., by country) should be considered.
    - **Benefits of Geo-Sharding:**
        - **Drastically Reduced Latency:** Data is stored physically closer to the users who access it.
        - **Efficient Queries:** Each read request operates on a much smaller, more relevant dataset, improving performance.
        - **Fault Isolation:** A data center issue in one region will not affect users in other regions.

---

### Transactions (ACID)
- **Requirement:** Required.
- **Rationale:**
    - **Atomicity (A):** Critical. When a new Point of Interest (POI) is added to the catalogue, all its attributes (name, coordinates, category, address) must be saved as a single, atomic operation to ensure data integrity.
    - **Consistency (C):** Necessary. This ensures that no operation violates business rules or constraints. For example, we can't accidentally add a place without a name or with a non-existent category.
    - **Isolation (I):** Necessary, especially for management by content editors via an admin panel. It ensures that concurrent edits of the same place do not conflict and corrupt the record. A level of **`Read Committed`** is sufficient to prevent one administrator from seeing the unsaved changes of another.
        - `Repeatable Read` is unjustified because there are no complex, long-running transactions that would require re-reading the same data and expecting it not to have changed.
        - `Serializable` is unnecessary overkill as there are no critical race conditions to prevent, unlike with unique user registrations.
    - **Durability (D):** Necessary. Data entered into the POI catalogue must be stored permanently (until intentionally changed or deleted).

---

### Consistency Model
- **Requirement:** Eventual Consistency.
- **Rationale:** This service does not require immediate updates. If a new cafe opens and is added to the database, it is perfectly acceptable for users to see it in the app with a delay of a few seconds or even longer. There is no need for real-time data synchronization across the entire system, which would be required by Strong Consistency at the cost of performance.

---

### Consensus
- **Requirement:** Required.
- **Rationale:** It's needed for the same reason as the other services: to enable reliable **automatic failover**. If the primary database node for the Places service fails, the cluster management system must use a consensus algorithm (like Raft) to elect a new leader from among the replicas. This guarantees the service remains available despite node failures.