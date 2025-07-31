## Matching Service

### Replication
- **Requirement:** Critically Necessary.
- **How it works:** Replication is a **fundamental and built-in** part of Elasticsearch's architecture.
    - **Rationale:** It's required for **High Availability** (if one node fails, the service continues to operate) and for **read scalability**. Elasticsearch can use replica shards to process search queries, which allows the system to handle a high load. Data is organized into **indices**, which are composed of **primary shards**, and each primary shard can have one or more **replica shards**.

---

### Sharding
- **Requirement:** Critically Necessary.
- **How it works:** Sharding is also a **fundamental and automatic** part of the architecture.
    - **Rationale:** The index containing the data of millions of users for matching will be too large for a single server. Sharding allows this index to be scaled horizontally across many nodes. You simply define the number of primary shards when creating the index, and Elasticsearch automatically distributes the data among them.

---

### Transactions (ACID)
- **Requirement:** Not Required.
- **Rationale:** Elasticsearch is not an ACID-compliant database; it's a search engine optimized for fast lookups, not complex transactions.
    - **Atomicity** is guaranteed only at the **single-document level**. Updating a single user's profile in the index is atomic, which is sufficient for this service's needs.

---

### Consistency Model
- **Requirement:** Eventual Consistency.
- **Rationale:** This is the natural and ideal model for a search index.
    - **The Flow:** When a user updates their profile, the change is first committed to the `Profile DB` (strong consistency), then published via Kafka to the `Matching Service`, which updates the document in Elasticsearch.
    - **The Delay:** Elasticsearch has a built-in delay (the `refresh_interval`, typically 1 second by default) between when a document is indexed and when it becomes available for search.
    - **The Trade-off:** This means changes to a user's profile will appear in search results with a slight delay. This is a perfectly acceptable trade-off for the extremely high performance of complex search queries, which are the core of this service.

---

### Consensus
- **Requirement:** Required.
- **Rationale:** Yes, consensus is a critical part of how an Elasticsearch cluster operates.
    - **How it works:** An Elasticsearch cluster needs to **agree** on its state: which nodes are active, how shards are distributed, and most importantly, **which node is the master node**. The master node is responsible for managing the state of the entire cluster.
    - Elasticsearch uses its own consensus algorithm (similar to Raft) to reliably elect a master node and manage the cluster, allowing the system to recover from failures automatically.

---

### Monitoring & Observability

- **Requirement:** Critically Necessary.
- **Rationale:** Monitoring is essential for maintaining the health and performance of the `Matching Service`.
    - **Tools:** Use Elasticsearch's built-in monitoring features or integrate with external tools like Kibana or Grafana for visualization and alerting.
    - **[Key Metrics to Monitor](SERVICE_MATCHES_MONITORING.md)**