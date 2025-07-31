### Services & Database Choices

#### 👤 [Authentication & User Accounts](auth/SERVICE_AUTH.md)
- **Database Choice:** `Relational DB (PostgreSQL / MySQL)`
- Requires strong **ACID guarantees** for user records, password hashes, and session management.
- A simple schema with `JOIN` capabilities is sufficient.

---

#### 🖼️ Profiles & Interests
- **Database Choice:** `Relational DB (PostgreSQL / MySQL)`
- Profile data (bio, interests, etc.) fits naturally into a normalized relational schema with many-to-many relationships (e.g., `user_interests`).
- Allows for easy enforcement of foreign-key constraints and efficient lookups via indexed joins.

---

#### 📍 Real-Time Location & Route History
- **Database Choice:** `Geo-enabled relational store (PostgreSQL + PostGIS)` for Hot Data (last 30 days) + `Object Store (S3)` for Cold Data.
- **PostGIS** allows for efficient storage and querying of location points using specialized GiST indexes.
- An LRU/TTL-like logic is managed at the application level to move data from hot to cold storage.

---

#### ☕ Places
- **Database Choice:** `PostGIS/PostgreSQL` + `Redis Cache`
- Query results for nearby places are cached in **Redis** to reduce database load.
- Can be populated with data from third-party APIs (e.g., Google Places) and stored locally in PostGIS.

---

#### 🔒 Privacy Settings & Data Exports/Deletes
- **Database Choice:** `Relational DB (PostgreSQL)`
- Privacy flags (`share_location`, `invisible_mode`) and data management requests are stored alongside user records.
- Relies on database features like cascade deletion to reliably handle data removal requests as required by GDPR.

---

#### 💬 Direct Messaging (DMs / Chat)
- **Database Choice:** `NoSQL Wide-Column Store (Cassandra)`
- Optimized for **very high write throughput** and fast reads of recent messages in a conversation (fan-out reads).
- Horizontally scalable and partitioned by `conversation_id` to handle massive message volumes without complex joins.
- An LRU-like algorithm can be used at the application level to manage conversation history.

---

#### 💳 Monetization & Payment Transactions
- **Database Choice:** `Relational DB (PostgreSQL)`
- Financial transactions demand **strict consistency**, auditability, and easy reporting, making an ACID-compliant DB essential.
- A relational ledger table (e.g., with `transaction_id`, `user_id`, `amount`) is easy to index and join with user data for reporting.

---

#### ⚡ Caching & Session Stores
- **Database Choice:** `In-Memory Key-Value Store (Redis)`
- Ideal for storing ephemeral, frequently accessed data with short TTLs (Time To Live).
- Example keys: `latest_location:<user_id>`, `matches:<user_id>`, `places:<geohash>:<category>`.

### Database Strategy Summary

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