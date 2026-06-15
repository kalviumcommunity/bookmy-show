# ShowTime Design Updates & Feedback Log

Based on feedback from the live panel roast, the ShowTime architecture has been revised to address security vulnerabilities and resource allocation edge cases. This document details the substantive changes made to the design.

---

## Update 1: Per-User Seat Hold Counter in Redis

**Triggered by:** Panel Question 3 - *"What stops one user from holding 200 seats?"*

### What Changed
We implemented a Redis-based rate limiter to track active seat holds per user.
* **Tracking Key**: `holds:{userId}:count` (integer counter).
* **Mechanism**: 
  1. Before acquiring Redis seat locks, the API server executes an atomic `INCRBY holds:{userId}:count {requested_seats}`.
  2. If the return value is greater than **8**, the API immediately decrements the counter back and returns a `429 Too Many Requests (Max hold limit exceeded)`.
  3. The counter is decremented when:
     - The seat hold is successfully converted to a booking (`status = 'booked'`).
     - The database sweep worker sweeps and expires the seat hold.
     - The booking transaction fails or is aborted.

```text
User Request (Hold 3 seats) 
         │
         ▼
[INCRBY holds:user123:count 3] ────► [Redis]
                                        │
                                 Is value > 8?
                                  /       \
                             Yes /         \ No
                                /           \
           [DECRBY count 3] ◄──┘             └──► Proceed to lock seats
         429 Max Limit Exceeded                    (MSETNX seat_lock:ID)
```

### Why This is Necessary
The original design only locked individual seats, leaving the system vulnerable to botnets and cart-hoarding scripts. A single malicious user could make parallel API requests to hold hundreds of seats, artificially locking out genuine buyers and starving ticket inventory.

### What it Costs
* **Latency**: Adds one additional atomic Redis command (~0.5ms) to the hold path.
* **Complexity**: Introduces operational risk of count drift. If an API server crashes or a transaction rollback path fails to decrement the Redis counter, a user could be blocked from booking until the key's TTL expires. A safety TTL of 15 minutes is applied to the counter key to ensure it automatically resets.

### What it Still Doesn't Solve
It does not prevent a sophisticated scalper from executing a Sybil attack by registering hundreds of fake user accounts (each holding 8 seats). To fully solve this, we must implement verification mechanisms at registration/login (such as Cloudflare Turnstile CAPTCHA and SMS verification).

---

## Update 2: Dynamic Seat Hold TTL Reduction During Active Sales

**Triggered by:** Panel Question 5 / 3 - *"Why not PostgreSQL row locking instead of Redis? Defend your choice." / seat holding recovery and lock starvation.*

### What Changed
We modified the seat hold duration (`held_until` timestamp) from a static 10-minute window to a dynamic window based on event traffic status.
* **Standard TTL**: 10 minutes (for low-demand browsing).
* **Active Sale TTL**: **3 minutes** (automatically triggered when an event transitions to `status = 'on_sale'`).
* **Sweep Frequency**: The background database sweep worker's interval is increased from once every 30 seconds to once every 10 seconds during active sales.

### Why This is Necessary
Under high-demand scenarios (like a Coldplay concert), a 10-minute hold window leads to high **lock starvation**. If 30% of users abandon their cart at the payment page, those seats remain locked and unusable for 10 minutes. By reducing the hold window to 3 minutes, abandoned seats are recycled back to the public pool 3.3x faster, maximizing ticket conversion and minimizing the time required to sell out the venue.

### What it Costs
* **User Experience**: Creates a higher-pressure checkout flow for the customer, who must complete payment within 3 minutes.
* **Database Load**: Increasing the database sweep worker's query frequency to 10 seconds increases index scan overhead on the PostgreSQL primary.
* **Network Failures**: If a user experiences a slow bank authorization OTP delivery that takes more than 3 minutes, their transaction will fail and their seat hold will be lost, resulting in support escalations.

### What it Still Doesn't Solve
If a user is double-charged because their payment cleared at $T+181\text{ s}$ (just after the 3-minute hold expired and the seat was booked by someone else), the payment worker will detect this discrepancy. The system will mark the booking as failed and trigger an automatic refund, but we cannot secure their seat, which has already been sold.
