# ShowTime Design Decisions

This document logs the non-obvious architectural choices made for the **ShowTime** ticketing platform. Each decision is structured to show the context, options considered, justification, accepted tradeoffs, and conditions that would trigger a revision.

---

## Decision 1: Concurrency Strategy (Redis Distributed Locking with PostgreSQL OCC)

**Context:** The system must handle a peak sustained load of 50,000 RPS (500,000 concurrent requests at noon). Double-bookings are legally unacceptable (must be exactly 0), and running infrastructure costs must remain under $2,000/month.

**Options considered:**
1. **PostgreSQL Row-Level Locking (`SELECT FOR UPDATE`)**: Safe, transactional, and natively handled by the database. However, this holds active database connections while queuing lock requests. At 500 max connections, the pool exhausts at **2,841 RPS**, failing the concurrency requirement.
2. **Hybrid locking with Redis `SETNX` & PostgreSQL OCC (Chosen)**: API servers acquire a temporary lock in Redis (`SETNX`) with a 30s TTL. If the lock is held, the request is rejected immediately in under 3ms. If acquired, the worker executes a PostgreSQL transaction using version-based Optimistic Concurrency Control (OCC) to write the hold.

**Why chosen:** Redis acts as an in-memory write filter. 99.9% of concurrent duplicate requests for the same popular seats fail at the cache layer without touching PostgreSQL. This protects the primary database from lock contention and connection pool exhaustion, allowing us to support 50,000 RPS on a modest database instance (`db.r6g.xlarge` costing $262/mo) and a small Redis cluster ($359/mo).

**Tradeoffs accepted:** Increases architectural complexity by introducing a Redis dependency. If the Redis cluster fails, the system loses its front-line filter, and write traffic falls back to PostgreSQL, which will cause transaction aborts and increased latencies rather than clean, fast in-memory rejections.

**Revision trigger:** If the monthly AWS budget is reduced below $500, we would decommission ElastiCache and implement pure database-level row locking, enforcing a CDN-level waiting room to throttle traffic to 1,000 RPS.

---

## Decision 2: Cache Invalidation Approach (Event-Driven Write-Through for Seat Counts)

**Context:** The category availability counts must be displayed to users browsing categories under a 50,000 RPS load. We must avoid cache stampedes on the database while keeping counts reasonably accurate.

**Options considered:**
1. **TTL-Only Expiry**: Seat availability counts are cached for a short window (e.g., 30s). When the TTL expires, the cache is deleted, and the next reader queries the DB. Under 50,000 RPS, this triggers a **cache stampede**, where thousands of concurrent readers query the DB to run `COUNT(*)` aggregate queries, crashing the database.
2. **Event-Driven Write-Through via Redis Increments/Decrements (Chosen)**: When a seat is held, the API atomically decrements the Redis count (`DECRBY`). If a hold expires or a booking fails, it increments the count (`INCRBY`). A safety TTL of 30s is attached, and on cache miss, a single-flight loader lock ensures only one worker queries the database to rebuild the count.

**Why chosen:** This guarantees that the cached count remains updated in real-time without ever triggering expensive database queries under load. Database traffic is completely flat and shielded.

**Tradeoffs accepted:** Code complexity increases. If a database transaction rolls back, the API must execute a compensating Redis increment to keep counts in sync. Network failures could cause temporary count drift, which is corrected by the 30-second TTL refresh.

**Revision trigger:** If the database tier is scaled out with multiple read replicas (under a higher budget), we could switch to read-replica queries with a short TTL, simplifying cache synchronization logic.

---

## Decision 3: UUID vs SERIAL for Booking IDs

**Context:** Generating unique identifiers for millions of booking transactions.

**Options considered:**
1. **SERIAL Integers**: Auto-incrementing integers (4 or 8 bytes). Highly index-friendly with sequential inserts. However, they are predictable, allowing malicious scrapers to iterate URLs (e.g., `/bookings/10001`) to enumerate every booking in the system. They also cannot be safely generated client-side.
2. **UUID v4 (Chosen)**: Random 128-bit unique identifiers.

**Why chosen:** UUIDs prevent enumeration attacks. Crucially, they can be generated client-side by the API server before inserting into the database, allowing the UUID to serve as an **idempotency key**. If the API server crashes after SQS publishing, the client can retry the request using the same UUID, preventing duplicate booking entries.

**Tradeoffs accepted:** UUIDs occupy 16 bytes (vs 4/8 bytes for SERIAL), which increases index sizes. Their random nature causes B-tree index fragmentation in PostgreSQL, degrading write performance at high volumes.

**Revision trigger:** If write performance degrades due to index size and page splits, we would switch to **UUID v7**, which preserves randomness while being chronologically sortable, eliminating B-tree fragmentation.

---

## Decision 4: SQS Visibility Timeout Value (30 Seconds)

**Context:** SQS visibility timeout must be set to prevent duplicate worker processing while keeping message processing delays to a minimum.

**Options considered:**
1. **Short Timeout (e.g., 5 seconds)**: If a payment gateway API call takes 10 seconds, the message will become visible in SQS again. A second worker will pick it up and charge the user's card again, creating a double-payment.
2. **Long Timeout (e.g., 10 minutes)**: If a payment worker crashes mid-transaction, the booking message is locked. The customer remains stuck in a pending state for 10 minutes before the system retries.
3. **Medium Timeout of 30 Seconds (Chosen)**: A 30-second window is selected.

**Why chosen:** The maximum API latency of payment gateways (Razorpay/PayU) is 10–15 seconds. A 30-second visibility timeout (double the gateway timeout) ensures that a single worker has enough time to complete the transaction and query status before the message is released, preventing double-processing.

**Tradeoffs accepted:** If a payment worker crashes immediately, the message remains invisible and un-processed for 30 seconds, causing a slight delay in booking resolution for that specific customer.

**Revision trigger:** If the payment gateway provider increases their API timeout limits or if we add multi-provider failover routing inside the worker (increasing worker processing time), we would increase the visibility timeout to 60 seconds.
