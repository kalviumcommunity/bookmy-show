# ShowTime System Architecture

This document presents the complete infrastructure and request lifecycle design for the **ShowTime** ticketing application. It details how the application handles 50,000 sustained RPS while preventing double-bookings and keeping running costs under $2,000/month.

---

## Complete System Architecture Diagram

```text
                                [ User Client ]
                                       │
                    ┌──────────────────┴──────────────────┐
                    │ HTTPS Request (GET Layout / POST Booking)
                    ▼
┌─────────────────────────────────────────────────────────────────┐
│                   CloudFront CDN (Global Edge)                  │
│  - Static Asset Cache Hit: Serves cached static seat maps       │
│  - Dynamic Pass-through: Routes bookings requests to ALB        │
└──────────────────────────────────┬──────────────────────────────┘
                                   │ Dynamic API Traffic
                                   ▼
┌─────────────────────────────────────────────────────────────────┐
│                 Application Load Balancer (ALB)                 │
│  - SSL/TLS Termination                                          │
│  - Rate Limit Protection (Rule: Max 100 requests/IP/minute)     │
│  - Target Group Health Checks: 10s interval, threshold 2        │
└──────────────────────────────────┬──────────────────────────────┘
                                   │ Load-Balanced HTTPS
                                   ▼
┌─────────────────────────────────────────────────────────────────┐
│            Node.js API Servers (Auto-scaling Group)             │
│  - Validates user sessions and payload structure                │
│  - Reads static cache / checks hold limits in Redis             │
│  - Initiates seats holding database transaction                 │
│  - Publishes booking events to SQS Payment Queue                │
└───────────┬──────────────────────┬──────────────────────┬───────┘
            │                      │                      │
    [*1] Redis Lock Check          │ SQL Write            │ Async SQS Message
    - Reads cached capacity        │ (Write-Only Path)    │ - bookingId
    - Checks holds:user:count      │                      │ - paymentToken
    - MSETNX seat locks (30s TTL)  │                      │ - price
            ▼                      ▼                      ▼
┌─────────────────────────┐  ┌──────────────┐  ┌──────────────────────────┐
│   Redis Cluster (Elasti)│  │ PostgreSQL   │  │   SQS Payment Queue      │
│  - (a) Cache: Event tier│  │   Primary    │  │ - Message: bookingId,    │
│        counts (30s TTL) │  │ - Processes  │  │   token, total amount    │
│  - (b) Seat Locks:      │  │   writes     │  │ - Visibility Timeout: 30s│
│        MSETNX (30s TTL) │  │   only       │  │   ([*2])                 │
│  - (c) User Hold Limit: │  │ - Holds seats│  │ - DLQ: routed after 3    │
│        max 8 holds ([*5])│ │   (OCC [*6]) │  │   processing failures    │
└─────────────────────────┘  └──────┬───────┘  └───────────┬──────────────┘
                                    │                      │
                                    │ Replication          │ Read message
                                    ▼                      ▼
          ┌─────────────────────────┴────────┐  ┌──────────────────────────┐
          │                                  │  │   Payment Worker(s)      │
          ▼                                  ▼  │   (ECS Fargate Tasks)    │
┌──────────────────┐               ┌────────┴─┐ │ - Pulls messages from SQS│
│ PostgreSQL Read  │               │   Postgre│ │ - Calls Payment Gateway  │
│    Replica 1     │               │   Replica│ │ - Updates DB to confirm  │
│ - Browsing &     │               │ - Browsed│ │ - Publishes confirmation │
│   availability   │               │   seat   │ │ - Deletes SQS message    │
│   queries ([*3]) │               │   details│ └──────────┬───────────────┘
└──────────────────┘               └──────────┘            │
                                                           │ Success notification
                                                           ▼
                                                ┌──────────────────────────┐
                                                │        SNS Topic         │
                                                │ - Fan-out message trigger│
                                                └──────────┬───────────────┘
                                                           │ Trigger
                                                           ▼
                                                ┌──────────────────────────┐
                                                │   SES & SMS Providers    │
                                                │ - Dispatches tickets     │
                                                └──────────────────────────┘
```

---

## Diagram Annotations (Mapping to Part A Design Decisions)

### `[*1]` Redis MSETNX Distributed Lock (see [CONCURRENCY.md](file:///c:/Users/NIKHIL%20REDDY/Desktop/Design%20BMS/docs/CONCURRENCY.md))
* **Decision**: We lock seat combinations in memory using Redis `MSETNX` prior to opening a database transaction.
* **Justification**: At a peak sustained load of 50,000 RPS, duplicate attempts to book hot seats are rejected at the caching layer in under 3ms. Only 1 request for the seat gets through to the database, protecting PostgreSQL from lock queues.

### `[*2]` SQS Visibility Timeout & DLQ (see [QUEUE.md](file:///c:/Users/NIKHIL%20REDDY/Desktop/Design%20BMS/docs/QUEUE.md))
* **Decision**: SQS Visibility Timeout is set to `30 seconds` with a Max Receive Count of `3` before DLQ routing.
* **Justification**: Since external payment gateway SDK calls take up to 15 seconds, a 30s visibility window ensures a worker is never double-allocated the same booking before processing completes. Sticking messages are offloaded to a Dead Letter Queue (DLQ) to prevent queue blockages.

### `[*3]` Database Write vs Read Separation (see [SCHEMA.md](file:///c:/Users/NIKHIL%20REDDY/Desktop/Design%20BMS/docs/SCHEMA.md))
* **Decision**: All seat status and booking mutations are routed exclusively to the PostgreSQL Primary, while browsing and category availability query reads route to the 2x Read Replicas.
* **Justification**: Prevents expensive `COUNT(*)` browsing reads from starving connections on the primary write database, maximizing transactional write capacity.

### `[*4]` Optimistic Concurrency Control (OCC) (see [SCHEMA.md](file:///c:/Users/NIKHIL%20REDDY/Desktop/Design%20BMS/docs/SCHEMA.md))
* **Decision**: The final update to `'booked'` in the database compares the seat `version` column.
* **Justification**: Eliminates long-duration pessimistic row locks during network calls, guaranteeing transaction safety with zero database deadlock risk.

### `[*5]` [POST-ROAST UPDATE] Per-User Seat Hold Counter (see [DESIGN-UPDATES.md](file:///c:/Users/NIKHIL%20REDDY/Desktop/Design%20BMS/docs/DESIGN-UPDATES.md))
* **Decision**: Implemented a Redis tracker key `holds:{userId}:count` checked before acquiring seat locks. Max limit of 8 holds.
* **Justification**: Prevents cart-hoarding scripts and scalping bots from locking thousands of seats concurrently.

### `[*6]` [POST-ROAST UPDATE] Dynamic Hold TTL Scheduler (see [DESIGN-UPDATES.md](file:///c:/Users/NIKHIL%20REDDY/Desktop/Design%20BMS/docs/DESIGN-UPDATES.md))
* **Decision**: The database seat hold `held_until` is reduced from 10 minutes to 3 minutes during active sale windows.
* **Justification**: Maximizes seat turnaround, returning abandoned seats back to the pool rapidly to prevent seat starvation.
