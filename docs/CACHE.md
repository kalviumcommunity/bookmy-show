# ShowTime Cache Design with TTL & Invalidation

Caching is vital to achieving sub-500ms API response times under a 50,000 RPS peak load. However, caching dynamic seat states can cause stale data reads and booking conflicts. This document details which data is cached, key naming conventions, TTL values, invalidation strategies, and what must never be cached.

---

## Cache Entry Configuration

| Cache Target | Redis Key Format | TTL (Time-To-Live) | Invalidation Trigger | Strategy |
| :--- | :--- | :--- | :--- | :--- |
| **Event Details** | `event:{event_id}` | `3600s` (1 Hour) | Admin updates event details (status, time, etc.) | Cache-Aside (Delete on update) |
| **Static Seat Map Layout** | `seatmap:{event_id}` | `86400s` (24 Hours) | Event cancellation / layout modification | Cache-Aside (Delete on update) |
| **Seat Availability Count** | `availability:{event_id}:{category}` | `30s` (Self-healing) | Seat status change (`available` $\leftrightarrow$ `held` / `booked`) | Write-Through (Atomic `DECR`/`INCR` + 30s TTL refresh) |

---

## Detail Breakdown

### 1. Event Details
* **Description**: Contains metadata like event name, date, start time, status, and venue details.
* **Why Cache?**: This data is requested on every browsing, listing, and checkout API request. The data is static during the ticket sale.
* **TTL (1 hour)**: Safely handles long-term caching.
* **Invalidation**: On database updates to an event, the admin backend deletes the Redis key `event:{event_id}`. The next user request triggers a DB query to rebuild the cache.

### 2. Static Seat Map Layout
* **Description**: Stores the structural seating configuration (e.g., sections, rows, coordinates, pricing bands).
* **Why Cache?**: Layout arrays are large JSON objects. Querying them from PostgreSQL on every page load consumes significant DB CPU and bandwidth. Since the physical layout never changes, it is cached for 24 hours.
* **Invalidation**: Only deleted on event cancellation or layout modification.

### 3. Seat Availability Count per Event per Category
* **Description**: Total count of remaining tickets in each tier (e.g., VIP, Premium, General). This count is shown to users browsing categories.
* **Why Cache?**: This is the highest-read path during a ticket sale. Without caching, thousands of users reloading the checkout page would run expensive aggregate counts (`COUNT(*)`) in PostgreSQL, exhausting connections.
* **Staleness Tolerance**: A 30-second stale count (e.g., showing 10 tickets remaining when only 8 are left) is acceptable. The final checkout step will enforce correctness.
* **Preventing Cache Stampede**: Deleting the availability key on every single seat hold triggers a cache stampede (hundreds of threads trying to run `COUNT(*)` on DB at the same time to rebuild cache). To prevent this, we use **atomic increments/decrements in Redis**.

---

## 🚨 What We EXPLICITLY Do Not Cache

### Individual Seat Status (e.g., `seat_status:{seat_id}`)
* **The Risk**: Caching whether seat A-12 is `available`, `held`, or `booked` introduces a window of staleness. If the cache is even 1 second stale, two users will see seat A-12 as `available`. They will both select it and proceed to checkout, triggering high-frequency lock attempts and database errors.
* **Alternative**: The real-time availability of individual seats is determined dynamically by querying the Redis **seat locks** (for holds) or PostgreSQL primary (for confirmed bookings) on demand. Caching is used for layout configurations and overall counts, but **never** for fine-grained state representation.

---

## Cache Invalidation & Update Flows

### 1. Event & Seat Map (Cache-Aside Pattern)

```javascript
async function getEventDetails(eventId) {
  const cacheKey = `event:${eventId}`;
  
  // 1. Try to fetch from Redis
  let data = await redis.get(cacheKey);
  if (data) {
    return JSON.parse(data);
  }
  
  // 2. Cache miss - Query DB
  const event = await db.query('SELECT * FROM events WHERE id = $1', [eventId]);
  if (!event) return null;
  
  // 3. Write back to Redis with 1-hour TTL
  await redis.set(cacheKey, JSON.stringify(event), 'EX', 3600);
  return event;
}
```

### 2. Seat Availability (Atomic Write-Through Pattern)

To avoid database cache stampedes under heavy traffic, the cache count is decremented/incremented atomically in Redis.

#### Flow A: Getting Availability Count

```javascript
async function getSeatAvailability(eventId, category) {
  const cacheKey = `availability:${eventId}:${category}`;
  
  // 1. Try to read from cache
  let count = await redis.get(cacheKey);
  if (count !== null) {
    return parseInt(count, 10);
  }
  
  // 2. Cache miss (or key expired) - run DB query
  // We use a distributed lock or single-flight loader to ensure only ONE worker hits the DB.
  const lockKey = `lock:availability_rebuild:${eventId}:${category}`;
  const lock = await redis.set(lockKey, 'locked', 'NX', 'EX', 5);
  
  if (lock) {
    try {
      const dbResult = await db.query(
        `SELECT COUNT(*) as count FROM seats 
         WHERE event_id = $1 AND category = $2 AND status = 'available'`,
        [eventId, category]
      );
      count = dbResult.rows[0].count;
      
      // Save to cache with 30s TTL
      await redis.set(cacheKey, count, 'EX', 30);
    } finally {
      await redis.del(lockKey);
    }
  } else {
    // Wait slightly and retry cache lookup
    await new Promise(resolve => setTimeout(resolve, 50));
    return getSeatAvailability(eventId, category);
  }
  
  return parseInt(count, 10);
}
```

#### Flow B: Updating Counts on Seat Hold (Atomic Decrement)

```javascript
async function holdSeats(eventId, userId, seatIds, category) {
  // 1. Acquire Redis seat locks (as detailed in CONCURRENCY.md)
  const locksAcquired = await acquireSeatLocks(seatIds);
  if (!locksAcquired) throw new Error("SEAT_ALREADY_LOCKED");

  // 2. Atomic decrement of the availability count in Redis
  const countKey = `availability:${eventId}:${category}`;
  await redis.decrby(countKey, seatIds.length);

  try {
    // 3. Perform SQL transaction
    await db.query('BEGIN');
    // ... DB Hold operations ...
    await db.query('COMMIT');
  } catch (error) {
    await db.query('ROLLBACK');
    // If DB transaction fails, increment the cache back to restore count
    await redis.incrby(countKey, seatIds.length);
    throw error;
  }
}
```
   This dual approach guarantees high performance while keeping the client-facing seat count in sync.
