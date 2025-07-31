## Profile Management & Interests Service

### Replication
- **Requirement:** Required.
- **Replication Type:** **Asynchronous with a Single Leader**.
- **Rationale:**
    - This is necessary for **High Availability** and **read-load distribution**. If the primary profile database fails, users will still be able to view profiles (perhaps with a minor lag), and the system will remain operational.
    - **Asynchronous replication** is more appropriate here because we don't require immediate system-wide synchronization. There's no need to pay the latency cost of synchronous replication. This approach ensures high performance for profile write operations (updates). The potential loss of a few seconds of data in a crash scenario is not critical for this type of information.

---

### Sharding
- **Requirement:** Not Necessary (at the predicted user scale).
- **Rationale:** Similar to the Authentication service, even for 10 million users, the profile data volume (including photo links, etc.) will be in the range of tens or hundreds of gigabytes, not terabytes. Sharding would significantly increase architectural complexity for no real benefit.
- Instead, **vertical scaling** (using a more powerful server) and employing **read replicas** are more than sufficient strategies for this service.

---

### Transactions (ACID)
- **Requirement:** Required.
- **Rationale:** Profile operations are transactional, so the ACID guarantees provided by a relational database are important.
    - **Atomicity (A):** Necessary. When a user changes their bio, adds several new interests, and removes old ones simultaneously, the entire operation must be a single unit. It's unacceptable for only a part of the changes to be saved.
    - **Consistency (C):** Necessary. The database must remain in a consistent state (e.g., interests must be linked to an existing user).
    - **Isolation (I):** Necessary. A level of **`Read Committed`** is sufficient. This guarantees that other services won't read a profile while it's in the middle of being updated and ensures a user won't see a partially-saved profile of another person. It's fast and doesn't create unnecessary locks.
        - `Repeatable Read` is not needed as it solves a problem (non-repeatable reads within a single, long transaction) that doesn't exist in this service's typical workflow.
        - `Serializable` is designed to prevent critical race conditions (like unique email registration) which are not present here. There is no business rule that two users can't have the same bio. Using it would severely slow down the service for no benefit.
    - **Durability (D):** Necessary. If a user saves changes to their profile, those changes must be permanent until they are explicitly modified again.

---

### Consistency Model
- **Requirement:** Hybrid: **Strong Consistency** (for the user) + **Eventual Consistency** (for other services).
- **Rationale:**
    - **Strong Consistency (for the user's own view):** When a user updates their profile, they expect to see those changes immediately if they refresh the page. This "read-your-own-writes" scenario must be guaranteed to prevent a confusing user experience.
    - **Eventual Consistency (for other services):** It's perfectly acceptable if the `Matching Service` sees the updated user profile with a few seconds of delay. This allows for efficient, asynchronous communication (via Kafka and CDC) without slowing down the `Profile Service` with direct requests to other system components.

---

### Consensus
- **Requirement:** Required.
- **Rationale:** Consensus is needed to reliably manage **automatic failover** for the replicated database cluster. The management system (e.g., Patroni) must decide which replica will become the new leader if the primary node fails. To avoid chaos and a "split-brain" scenario, all members of the cluster must reach a consensus on this choice.