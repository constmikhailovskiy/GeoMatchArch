# GeoMatchArch

Geo Match is a dating platform that tracks users' routes and local places, helping users find friends and partners based on their interests and commonalities.

This is a high-load service available to users on multiple platforms: both mobile and web.

## Requirements
### Functional
- User Authentication & Authorization
- Profile Management with Interest Tags
- Real-Time Location Tracking & Route History
- Basic Matching by Interests & Top Locations
- Direct Messages
- Local Places Catalogue Based on Current Coordinates
- Personal Data & Privacy Controls
- Monetization & Payment Processing 

### Non-Functional

- The system must handle high volumes of concurrent users sending location updates (GPS pings) and requesting match queries without degradation in response time
- The architecture should allow horizontal scaling (e.g., partitioning location writes via a message queue and auto‐scaling matching service instances) so that:
- Location‐ingest latency remains low (under 100 ms per ping)
- Match‐query latency stays under 200 ms, even under peak load
- Core services must maintain an uptime of at least 99.9% and be resilient to component failures.
- All sensitive data (user credentials, location traces, profile info) must be protected both in transit and at rest. The system must comply with GDPR/CCPA principles
- Low Location‐ingest latency (< 100 ms)
- Match‐query latency under 200 ms, even at peak

### Limits
- Max 5 failed login attempts per 15 min (per IP/user) before a short lockout.
- Up to 20 tags per user.
- Max search radius = 20 km for Matches
- Max 5 km radius for places lookup
- Max 2,000 characters per message.
- Max 5 attachments per conversation; max 10 MB per file.
- Rate limit: max 20 sends/sec per user; max 500 messages/day.

## Back of Envelope

### Key Assumptions
- **Users:** 10 million total / 1.5 million DAU (Daily Active Users) max

---

### Traffic Estimates (Back of the Envelope)

#### 📍 Location Pings
- **Size per ping:** ~100 B (16 B for coordinates + metadata)
- **Volume per day:** 1.5M DAU × 100 pings = 150M pings/day
- **Write Rate:** 150M / 86,400s ≈ 1,736 writes/sec (peak ~15k wps)
- **Route-history reads:** 1.5M DAU × 2 requests = 3 million reads/day
- **Read Rate:** 3M / 86,400s ≈ 35 reads/sec (peak ~200 rps)
- **Read-Write Ratio:** ~50:1 (Write Heavy)

#### ❤️ Matching Requests
- **Volume per day:** 1.5M DAU × 20 requests = 30 million reads/day
- **Read Rate:** 30M / 86,400s ≈ 347 reads/sec (peak ~3,000 rps)

#### ☕ Places of Interest Requests
- **Volume per day:** 1.5M DAU × 10 requests = 15 million/day
- **Read Rate:** 15M / 86,400s ≈ 174 reads/sec (peak ~1,500 rps)

#### 💬 Direct Messages
- **Writes per day:** 1.5M DAU × 10 messages = 15 million messages/day
- **Write Rate:** 15M / 86,400s ≈ 174 writes/sec (peak ~1,500 wps)
- **Reads per day:** 1.5M DAU × 20 reads = 30 million reads/day
- **Read Rate:** 30M / 86,400s ≈ 347 rps (peak ~3,000 rps)
- **Read-Write Ratio:** ~1:2 (Read Heavy)

#### 💳 Payment Service
- **Subscribers:** 10% of DAU → 150K subscribers active/day
- **Write Rate (Transactions):** ~0.06 tps (peak ~1 tps on billing days)
- **Read Rate (Billing History):** ~0.24 rps (peak ~2 rps)
- **Admin Queries:** Negligible load (~500 queries/month)

---

### Database Storage Estimates

#### Location Data
- **Daily:** 150M pings × 100 B ≈ 15 GB/day
- **Monthly:** 15 GB × 30 ≈ 450 GB/month
- **Yearly:** 450 GB × 12 ≈ **5.4 TB/year**

#### Profiles & Interests
- **Total:** 10M users × 1 KB ≈ **10 GB** (one-time growth)

#### Messages
- **Daily:** 15M messages × 1 KB ≈ 15 GB/day
- **Yearly:** ≈ 5.4 TB/year
- **With Overhead:** (+20% for indexes) ≈ **6 TB/year**

#### Transactions & Billing
- **Subscription logs:** 300 MB/month
- **One-off purchases:** 450 MB/month
- **With Overhead:** ≈ 900 MB/month (≈ **11 GB/year**)

#### Indexes & Auxiliary Tables
- **Total:** ~**50 GB/year**

### Caching (Redis)
- `latest_location`: `<user_id>`: TTL 90s → ~1.5 M keys/day
- `matches`: `<user_id>`: TTL 30s → ~1.5 M keys/day
- `places`: `<geohash>:<category>`: TTL 5 min → A few thousand keys depending on traffic
- `recent_chats`: `<user_id>`: TTL 5 min (or per conversation) → ~1.5 M keys/day (assuming caching each user’s recent DMs briefly)

---

### Combined Storage Estimates
- **Location data:** ~5.4 TB/year
- **Profiles & interests:** ~10 GB (one-time)
- **Messages:** ~6 TB/year (including overhead)
- **Transactions & billing logs:** ~11 GB/year
- **Indexes & auxiliary tables:** ~50 GB/year

---

### Optional Optimizations
- Store aggregated "top routes" instead of raw points for older data to reduce DB size (e.g., compress points into polygons).
- Archive direct messages older than 1 year to cold storage or delete according to user policy.

### Load Balancers

**Present:** Yes

1.  **Edge/API Gateway LB**
    -   Fronts all client traffic (web, mobile).
    -   Terminates TLS, enforces throttling, and routes requests to the proper microservice cluster.

2.  **Database LBs (for Read Replicas)**
    -   Distributes read queries across multiple database replicas to improve performance and availability.

### Load Balancer Traffic Estimates

#### Average Load
- **Writes/sec:** 1,736 (location) + 174 (messages) ≈ **1,910 wps**
- **Reads/sec:** 35 (history) + 347 (matches) + 174 (places) + 347 (message reads) + 0.24 (billing) ≈ **903 rps**
- **Combined Average:** ~2,800 requests/sec

---

#### Peak Load
- **Writes:** ~15,000 (location) + 1,500 (messages) ≈ **16,500 wps**
- **Reads:** ~200 (history) + 3,000 (matches) + 1,500 (places) + 3,000 (message reads) + ~2 (billing) ≈ **7,702 rps**
- **Combined Peak:** ~24,200 requests/sec

---

### Routing Strategy

- **Chosen Variant:** Weighted Routing

> We know ahead of time that some pods will be more powerful, so we will configure weighted routing. This allows heavier-duty instances to shoulder a larger share of the traffic, while smaller pods still receive requests but at a lower rate.
