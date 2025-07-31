## Authentication & User Accounts Service

### Replication
- **Requirement:** Critically Necessary.
- **Rationale:** User account and authorization data is essential for the entire system's operation. The loss of this data would make it impossible for users to access the application. Therefore, ensuring data is replicated is a critical requirement.
- **Replication Type:** **Single-Leader, Synchronous (Quorum-Based)**.
    - A centralized, single-leader model (where one node accepts writes and replicates them to read-only followers) is highly desirable for managing user data.
    - **Synchronous replication** is crucial because we need reliable confirmation that user data has been written to a majority of nodes. This type provides the best balance of **Availability** and **Consistency**.
    - We can sacrifice some write speed because the intensity of write operations (logins, account updates) is not as high as content-interaction workloads (reading other data).
    - The **quorum-based** approach further reduces write latency by not requiring confirmation from *all* nodes, only a majority (>50%), which provides a strong guarantee of durability while maintaining good performance.

---

### Sharding
- **Requirement:** Not Necessary.
- **Rationale:** The total data volume for 10 million user accounts is measured in gigabytes, not terabytes, and can be easily handled by a modern, replicated PostgreSQL server.
- Sharding would introduce significant complexity, especially in ensuring the uniqueness of logins (email). In a sharded system, no single shard has complete information about all existing emails, making a simple `UNIQUE` constraint ineffective across the entire dataset.

---

### Transactions (ACID)
- **Requirement:** Critically Necessary.
- **Rationale:**
    - **Atomicity (A):** Essential for operations like user registration, which involves creating records in multiple tables (e.g., an auth record and a profile record). The entire chain of operations must succeed or fail as a single unit to prevent inconsistent "ghost" accounts.
    - **Consistency (C):** The database must enforce its rules, such as preventing the creation of two accounts with the same email, even if the requests are simultaneous. This prevents data corruption.
    * **Isolation (I):** Guarantees that concurrent operations (like a user changing their password from two different devices at the same time) do not interfere with each other and corrupt the data.
    * **Durability (D):** Guarantees that once a user's data is successfully saved, it will not be lost, regardless of system crashes or failures.
- **Isolation Level:** A combined approach of **`Read Committed` + `Serializable`** is optimal.
    - **`Read Committed`** is the ideal balance for most frequent session and account-related operations (login, fetching settings, updates).
    - **`Serializable`** should be explicitly used for the user registration process to prevent race conditions when checking for unique emails, even though it may have higher latency.

---

### Consistency Model
- **Requirement:** Strong Consistency.
- **Rationale:** For a service that manages identity and sensitive data, any compromise on consistency is unacceptable.
    - When a user registers, they must be able to log in immediately.
    - When a user changes their password, any subsequent login attempt must be validated against the **new** password instantly.
    - **Eventual Consistency