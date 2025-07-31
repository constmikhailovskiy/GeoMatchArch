## Data Privacy Service

### Replication
- **Requirement:** Critically Necessary.
- **Replication Type:** **Single-Leader, Synchronous (Quorum-Based)**.
- **Rationale:** A user's privacy settings and consent records are legally sensitive and must be handled with the highest integrity.
    - **Synchronous replication** is required to guarantee that a change (e.g., a user revoking consent for location sharing) is durably saved to multiple servers before being confirmed. This eliminates the risk of data loss due to a single server failure.
    - The write volume for this service is low, so the performance impact of synchronous replication is negligible and a worthwhile trade-off for the strong durability guarantee.

---

### Sharding
- **Requirement:** Not Necessary.
- **Rationale:** The data volume for this service is extremely small. Even with 10 million users, the total size of privacy settings (a few flags and timestamps per user) will be minimal. A single replicated PostgreSQL instance can handle this load and volume without any issues. The complexity of sharding is completely unjustified.

---

### Transactions (ACID)
- **Requirement:** Critically Necessary.
- **Rationale:** The management of consent and privacy settings must be transactional to ensure data integrity and auditability.
    - **Atomicity (A):** Ensures that when a user changes multiple settings at once, the entire operation succeeds or fails as a single unit.
    - **Consistency (C):** Enforces data rules, such as linking privacy settings to a valid `userID`.
    - **Isolation (I):** Prevents concurrent operations from interfering with each other. A `Read Committed` level is sufficient to ensure that reads only see successfully saved data.
    - **Durability (D):** Guarantees that once a user's consent choice is recorded, it is permanent and will survive any system failure. This is a fundamental legal requirement under regulations like GDPR.

---

### Consistency Model
- **Requirement:** Strong Consistency.
- **Rationale:** This is non-negotiable. When a user revokes consent or changes a privacy setting, that change must take effect **immediately** across the entire system. There can be no delay.
- **Eventual Consistency** is unacceptable as it could lead to a user's privacy choices being violated during the replication lag (e.g., their location continuing to be shared after they have opted out).

---

### Consensus
- **Requirement:** Required.
- **Rationale:** As with the other critical PostgreSQL-based services, a consensus algorithm is required to manage the **automatic failover** of the replicated database cluster. It ensures that a new leader is elected reliably and without data corruption if the primary node fails, thus maintaining the high availability of this critical service.