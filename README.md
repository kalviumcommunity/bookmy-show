# ShowTime Ticketing System - Architecture Design (Part A)

Welcome to the ShowTime architecture design repository. This project outlines the high-concurrency, budget-constrained design for a BookMyShow competitor capable of selling out the highest-demand concerts (like Coldplay's India tour) without double-bookings, under strict budget constraints.

## System Constraints & Tradeoffs Analysis

Designing ShowTime requires balancing three competing forces, forming an architectural trilemma:
1. **Scale**: 500,000 concurrent users at $T+0$ (noon).
2. **Correctness**: Zero double-bookings.
3. **Cost**: Strict budget limit of $2,000/month on AWS.
4. **Latency**: API response time under 500ms.

---

### Constraint 1: 5 Lakh Concurrent Users at 12:00:00 Noon

#### Peak RPS Calculations
* **Peak Instantaneous Load (Worst-Case)**: If all 500,000 users click "Book Now" at exactly 12:00:00 noon within a 1-second window, the system must handle **500,000 RPS**.
* **Sustained Peak Load (Realistic)**: If the 500,000 users trigger their booking requests over a 10-second window, the peak sustained load is **50,000 RPS**. Distributed over the first minute, it is **8,333 RPS**.
* For this design, we prepare for a sustained load of **50,000 RPS** at the API gateway, with a target API latency of under 500ms.

#### Core Bottlenecks
* **Database Connection Pool Exhaustion**: A traditional relational database (like PostgreSQL) handles a limited number of concurrent connections (typically 500–1,000 max connections, even with PgBouncer). If every request opens a transaction and waits for a row lock or external API call (like payment), the database connection pool exhausts in milliseconds, causing a cascading failure of the entire backend.
* **Lock Contention**: When 500,000 users try to book the same hot seat (e.g., seat A-12), database row-level locking (`SELECT FOR UPDATE`) causes threads to queue up. This serialized execution destroys throughput and increases latency past the 500ms threshold.

---

### Constraint 2: Zero Acceptable Double-Bookings

#### Technical Definition of a Double-Booking
A double-booking occurs if:
1. Two different bookings in the `bookings` table with status `confirmed` are linked to the same `seat_id` in the `booking_seats` table.
2. The system transitions the same physical seat in the `seats` table from `available` to `booked` or `held` for two different users simultaneously due to a race condition (read-modify-write hazard).

#### Prevention Mechanisms
* **Database Constraints**: A database-level unique constraint on `booking_seats(seat_id, event_id)` or unique indexes on `seats` ensuring a seat can only be tied to one active booking.
* **Pessimistic Row-Level Locking (`SELECT FOR UPDATE`)**: Locks the seat row during the transaction so other transactions must wait until the lock is released.
* **Optimistic Concurrency Control (OCC)**: Using a `version` integer column on the `seats` table. Updates only succeed if the version hasn't changed.
* **Distributed Locks**: Using Redis `SETNX` (Set if Not Exists) to acquire a lock for a specific seat in memory before hitting the database.

---

### Constraint 3: $2,000/Month AWS Budget

#### Resource Allocation Breakdown ($1,840/month)
To handle 50,000 RPS while respecting the budget, we distribute costs as follows:

| Component | AWS Resource | Details | Monthly Cost (USD) |
| :--- | :--- | :--- | :--- |
| **API Servers** | 6 × EC2 `t3.xlarge` | 4 vCPU, 16GB RAM. Runs Node.js cluster. | $719.00 |
| **Primary Database** | 1 × RDS PostgreSQL `db.r6g.xlarge` | 4 vCPU, 32GB RAM. Handles writes. | $262.00 |
| **Read Replicas** | 2 × RDS PostgreSQL `db.r6g.large` | 2 vCPU, 16GB RAM. Offloads availability reads. | $262.00 |
| **Cache & Locks** | 3 × ElastiCache Redis `cache.r6g.large` | 3-node Redis cluster. Handles distributed locks & caching. | $359.00 |
| **Payment Workers** | ECS Fargate Tasks | Average 10 tasks running (0.25 vCPU, 0.5GB RAM). | $73.00 |
| **Networking & Queues** | ALB, SQS, CloudFront, Route53 | Load balancing, global CDN, and message queues. | $165.00 |
| **TOTAL** | | | **$1,840.00** |

#### Budget Downscale Tradeoffs (The $500/Month Challenge)
If the budget were cut to $500/month, the architecture must change drastically:
1. **Eliminate ElastiCache Redis**: Save $359/mo. We must fall back to in-memory caching on API servers for static content (venue/event layouts) and handle all locks directly in PostgreSQL using row-level locks or optimistic locking.
2. **Downgrade RDS Instance**: Replace the `db.r6g.xlarge` and read replicas with a single `db.t4g.medium` (2 vCPU, 4GB RAM) with no replicas (~$50/mo vs $524/mo).
3. **Reduce API Servers**: Downsize to 2–3 × `t3.medium` instances (~$90/mo).
4. **Co-locate Workers**: Run the async payment workers inside the same API server EC2 instances to avoid Fargate costs.
5. **Waiting Room Throttling**: Since a $500/mo infrastructure cannot handle 50,000 RPS, we must implement a **Virtual Waiting Room** (like CloudFront integrations or queue-it style rate limiters) at the CDN level. This throttles incoming user requests to 1,000 RPS, letting users in in batches, keeping the DB safe from collapse.

---

## Repository Structure

* [docs/SCHEMA.md](file:///c:/Users/NIKHIL%20REDDY/Desktop/Design%20BMS/docs/SCHEMA.md): Database schema definitions, constraints, indexes, and design justifications.
* [docs/CONCURRENCY.md](file:///c:/Users/NIKHIL%20REDDY/Desktop/Design%20BMS/docs/CONCURRENCY.md): Concurrency analysis, locking math, and the hybrid locking strategy.
* [docs/CACHE.md](file:///c:/Users/NIKHIL%20REDDY/Desktop/Design%20BMS/docs/CACHE.md): Redis cache keys, TTL settings, and event-driven invalidation logic.
* [docs/QUEUE.md](file:///c:/Users/NIKHIL%20REDDY/Desktop/Design%20BMS/docs/QUEUE.md): Asynchronous order workflow, worker state machine, and failure handling.
* [docs/ARCHITECTURE.md](file:///c:/Users/NIKHIL%20REDDY/Desktop/Design%20BMS/docs/ARCHITECTURE.md): High-level system architecture layout.
* [docs/DESIGN-DECISIONS.md](file:///c:/Users/NIKHIL%20REDDY/Desktop/Design%20BMS/docs/DESIGN-DECISIONS.md): Key design decisions.
* [docs/DESIGN-UPDATES.md](file:///c:/Users/NIKHIL%20REDDY/Desktop/Design%20BMS/docs/DESIGN-UPDATES.md): Log of adjustments and review feedback.
