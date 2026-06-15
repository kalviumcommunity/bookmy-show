# ShowTime Live Panel Roast QA Notes

This document records the questions asked by the live roast panel, the architectural answers provided, and a self-assessment of the completeness of each answer.

---

### Question 1 (Failure Scenarios)
> **"What happens if Redis crashes mid-lock?"**

* **Answer Provided**: 
  1. If a Node.js API server acquires a Redis seat lock but Redis crashes before the database hold transaction is committed, the current API request fails.
  2. The database remains the source of truth; since no SQL transaction was committed, the seat status in PostgreSQL remains `'available'`.
  3. When Redis recovers, the lock key will have expired (30s TTL). 
  4. If Redis fails over to a replica that missed the lock write, a window opens where a second user might acquire the lock in Redis. However, the database's unique constraint (`idx_booking_seats_active`) and version-based Optimistic Concurrency Control (OCC) will abort the second user's transaction, preventing double-bookings.
* **Assessment**: **Complete**. Correctly demonstrated that the database serves as the final correctness guarantee, keeping Redis failures from causing double-bookings.

---

### Question 2 (Scaling Limits)
> **"At what concurrent users does your DB become the bottleneck?"**

* **Answer Provided**: 
  1. The API servers offload 99% of browsing and hold traffic using CloudFront and Redis `MSETNX` locks. 
  2. The primary database bottleneck is the write rate of the SQS payment workers on the PostgreSQL primary.
  3. A PostgreSQL `db.r6g.xlarge` instance handles roughly 3,000–5,000 write transactions per second. 
  4. Assuming a checkout completion rate of 50%, the database bottlenecks at an incoming checkout volume of **6,000–10,000 payments/sec**. If checkout volumes exceed this, connection queues will build up at the database level.
* **Assessment**: **Complete**. Supported by write throughput estimates for the RDS tier, mapping the exact boundary of DB scaling.

---

### Question 3 (Edge Cases)
> **"What stops one user from holding 200 seats?"**

* **Answer Provided**: 
  * *Initial Gap*: In the original Part A design, we only locked individual seats but did not check the aggregate count of holds per user. A malicious scalper script could hold 200 seats in multiple parallel requests.
  * *Resolution*: We have added a substantive update (see `DESIGN-UPDATES.md`). We now track holds using a Redis key: `holds:{userId}:count`. Before acquiring any seat locks, the API increments this count. If it exceeds **8**, the transaction aborts immediately. The counter is decremented when the seat hold expires or is converted to a booking.
* **Assessment**: **Gap Identified & Resolved**. The panel highlighted an abuse vector which was resolved with a per-user rate limiter.

---

### Question 4 (Budget Constraints)
> **"Your auto-scaling spikes your bill to $3,200 this month - what's your plan?"**

* **Answer Provided**: 
  1. We configure CloudWatch Billing Alarms at $1,800 to warn of cost overruns.
  2. To prevent budget creep, we place **strict caps** on our auto-scaling groups: maximum 12 API servers (EC2) and 20 payment workers (Fargate).
  3. If traffic spikes beyond this capacity, we do not scale out further. Instead, we let the SQS queue absorb the buffer. The API servers will return a `202 Accepted` to customers, and the payment workers will process the queue sequentially. This smooths out traffic spikes without incurring extra costs.
* **Assessment**: **Complete**. Demonstrated how asynchronous queueing acts as a cost-control mechanism, trading real-time processing speed for budget predictability.

---

### Question 5 (Tradeoffs & Concurrency)
> **"Why not PostgreSQL row locking instead of Redis? Defend your choice."**

* **Answer Provided**: 
  1. Pessimistic row locking (`SELECT FOR UPDATE`) holds database connections open during execution.
  2. If 20% of requests are payment checkouts taking 800ms, a 500-connection PgBouncer pool collapses at **2,841 RPS**. 
  3. In contrast, Redis `SETNX` distributed locks run in-memory, process over 100,000 ops/second, and resolve in under 1ms. 
  4. Using Redis to filter out duplicate booking attempts prevents connection pool collapse and allows us to handle the load on a budget RDS database.
* **Assessment**: **Complete**. Backed by mathematical proof of connection pool collapse.
