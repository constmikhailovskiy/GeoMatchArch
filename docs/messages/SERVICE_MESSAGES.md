## Direct Messages Service

### Replication
- **Requirement:** Critically Necessary.
- **Replication Type:** **Leaderless Model**.
- **Rationale:** Any node can accept a write request, which makes the system extremely resilient to failures and provides the high write availability needed for a high-intensity chat application where millions of users could be messaging simultaneously.

---

### Sharding
- **Requirement:** Critically Necessary.
- **Rationale:** The service must handle a massive volume of message data being written at an extremely high intensity.
    - The chosen database, **Cassandra**, provides sharding **out-of-the-box**. For this use case, sharding is done by the **`conversation_id`**.
    - This ensures that all messages from a single chat are stored on the same node/segment, making the process of loading chat history very fast, as it doesn't require querying multiple shards.
    - This key-based sharding (and its underlying hash function) also allows for the even distribution of chat data across different nodes, preventing "hot spots".

---

### Transactions (ACID)
- **Requirement:** Not Required (in its classic form).
- **Rationale:**
    - **Atomicity** is guaranteed only at the single-row (single message) level, which is sufficient. Sending a message is a simple, self-contained operation that does not require simultaneous changes in other parts of the database. The guarantee that each individual message is saved correctly is enough for the service to work reliably.
    - **Isolation** is also at the row-level. Since there are no complex, multi-row transactions, there is no need for complex isolation levels found in relational databases. A message-sending operation is inherently isolated by its nature.

---

### Consistency Model
- **Requirement:** Required (and available out-of-the-box with Cassandra).
- **Rationale:** Consistency is necessary to guarantee message delivery, ensure the correct order of messages in a chat, and reduce the risk of "phantom messages" (e.g., a message reappearing after being deleted).
    - In Cassandra, the consistency level can be tuned for each request.
    - For writing messages, using **`QUORUM`** is advisable. This means a write must be acknowledged by a majority of replicas, providing a good balance between reliability and speed.
    - `QUORUM` can also be used for reads to ensure data is up-to-date.

---

### Consensus
- **Requirement:** Not Required.
- **Rationale:** Consensus is a slow and expensive process, and for a chat application, **speed and availability are paramount**.
    - Cassandra uses a simpler and faster **last-write-wins (LWW)** approach based on timestamps to resolve conflicts.
    - True data conflicts are highly unlikely (multiple users cannot edit the same message at the same time), while the need for high-speed message delivery is critical.