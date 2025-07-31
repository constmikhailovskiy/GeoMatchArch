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

- Users: 10 million total / 1.5 million DAU max
- Location pings:
  - Size per ping: ~100 B (16 B for coordinates + metadata)
  - Volume per day: 1.5 M DAU × 100 pings = 150M pings/day
    - 150M / 86 400s ≈ 1,736 writes per second, peak ~15k w/ps
  - Route-history reads: 1.5M DAU × 2 requests = 3 million reads/day
    - 3M / 86 400 ≈ 35 r/ps, peak ~200 r/ps
  - Read-write ratio (location): ~1 billion writes: 20 million reads ≈ 50:1
 
- Matching requests:
  - Volume per day: 1.5M DAU × 20 match requests = 30 million reads/day
    - 30M / 86,400s ≈ 347 r/ps, peak ~3 000 rps

- Places of Interest requests:
  - Volume per day: 1.5M DAU × 10 = 15 million/day
    - 15M / 86,400s ≈ 174 r/ps, peak ~1,5k r/ps

- Direct Messages:
  - Each active user sends: ~10 messages/day avg.
  - Average message payload (text + metadata) ≈ 1 KB.
  - Each message is read by the recipient(s) once on average.
  - Writes per day: 1.5 M DAU × 10 = 15 million messages/day
  - 15 M / 86 400 ≈ 174 writes/sec (peak ~1,500 wps)
  - Reads per day: 1.5 M DAU × (assume 20 reads) = 30 million reads/day
    - 30 M / 86 400 ≈ 347 rps (peak ~3,000 rps)
  - Read-write ratio: ~15 M writes: 30M reads ≈ 1:2

