# Scalability Considerations — Online Vehicle Rental Platform

---

## 1. HORIZONTAL VS VERTICAL SCALING

| Strategy | Description | When to Use |
|---|---|---|
| **Vertical Scaling** | Upgrade CPU/RAM of a single server | Quick short-term fix; has a physical ceiling |
| **Horizontal Scaling** | Add more server instances behind a load balancer | Preferred — scales linearly with traffic |

**Decision:** All microservices are **stateless** (no data stored in memory between requests — sessions are in Redis). This means any instance can handle any request, making horizontal scaling easy and safe.

```
Traffic doubles
     |
     v
Vertical  -> upgrade one big server (expensive, limited)
Horizontal -> add 2 more instances behind load balancer (cheap, unlimited)
```

---

## 2. LOAD BALANCING

```
All incoming traffic
        |
        v
  Load Balancer (Nginx / AWS ALB)
  Algorithm: Round Robin
        |
   .----+------.-------.
   |    |      |       |
  Inst1 Inst2 Inst3  Inst4   <-- Booking Service instances
```

- **Health checks** every 10 seconds — unhealthy instances removed automatically
- **Sticky sessions NOT used** — services are stateless; sessions live in Redis
- **Auto Scaling** — new instances added automatically when CPU exceeds 70%
- **New instance ready** within 2 minutes of trigger

---

## 3. CACHING STRATEGY (REDIS)

### Why Cache?
Every time a user searches for vehicles in Mumbai, the query hits PostgreSQL.
With 10,000 concurrent users doing the same search — that's 10,000 DB queries per minute.
Redis cache reduces this to 1 DB query per minute for the same data.

### Cache-Aside Pattern (used by all services)

```
Request arrives at Service
        |
        v
Check Redis cache
        |
   .----+-------.
   |             |
  HIT           MISS
   |             |
   v             v
Return data   Query PostgreSQL
immediately        |
                   v
              Store result in Redis
              with TTL (expiry)
                   |
                   v
              Return data
```

### Cache TTL (Time-To-Live) Per Data Type

| Data | TTL | Invalidation Trigger |
|---|---|---|
| Vehicle search results (by city) | 60 seconds | Vehicle added, updated, or removed |
| Vehicle detail page | 5 minutes | Vehicle info updated |
| Vehicle availability calendar | 30 seconds | Booking confirmed or cancelled |
| User session / JWT blacklist | 24 hours | User logs out |
| Rate limit counters | 1 minute | Sliding window reset |
| Static page data | 10 minutes | Manual cache clear |

---

## 4. DATABASE SCALING

### Read Replicas

```
Write operations (INSERT, UPDATE, DELETE)  -->  PostgreSQL PRIMARY
Read  operations (SELECT)                  -->  PostgreSQL READ REPLICA

90% of all database operations are reads.
(searching vehicles, viewing bookings, checking availability)
Read replicas absorb this traffic and free the primary for writes.
```

### Connection Pooling (PgBouncer)

Each microservice maintains a **connection pool** to the database.
Without pooling — 1000 users → 1000 simultaneous DB connections → PostgreSQL crashes.
With pooling — 1000 users → max 20 connections reused → PostgreSQL stable.

```
1000 user requests
       |
       v
  Connection Pool (PgBouncer)
  Max 20 connections at a time
       |
       v
  PostgreSQL (receives max 20 connections, handles efficiently)
```

### SQL Indexes for Performance

```sql
-- Most frequent query: search vehicles by city and status
CREATE INDEX idx_vehicles_city_status
  ON vehicles(city, status);

-- Filter by vehicle type and price
CREATE INDEX idx_vehicles_type_rate
  ON vehicles(vehicle_type, daily_rate);

-- User viewing their booking history
CREATE INDEX idx_bookings_user_id
  ON bookings(user_id);

-- Availability check (most critical — prevents double booking)
CREATE INDEX idx_bookings_vehicle_dates
  ON bookings(vehicle_id, start_datetime, end_datetime);

-- Blocked dates lookup
CREATE INDEX idx_availability_vehicle_dates
  ON vehicle_availability(vehicle_id, blocked_from, blocked_to);

-- Partial index — only index available vehicles (smaller, faster)
CREATE INDEX idx_active_vehicles
  ON vehicles(city, vehicle_type)
  WHERE status = 'AVAILABLE';
```

---

## 5. PREVENTING DOUBLE-BOOKING (DISTRIBUTED LOCK)

**The Problem:**
Two users open the same vehicle page at the same time and both try to book it for the same dates.
Without a lock — both bookings could succeed, creating a conflict.

**The Solution — Redis Distributed Lock (SETNX):**

```
User A requests vehicle V1 for Feb 1-4
User B requests vehicle V1 for Feb 1-4 at the same moment

Step 1: User A -> Booking Service tries to acquire lock:
        SETNX  lock:vehicle:v1:2024-02-01-04  "user-A"  EX 30
        Redis returns 1 (lock acquired by A — success)

Step 2: User B -> Booking Service tries same lock:
        SETNX  lock:vehicle:v1:2024-02-01-04  "user-B"  EX 30
        Redis returns 0 (lock already held — FAILED)

Step 3: User B receives:
        409 Conflict — "Vehicle just booked. Please choose different dates."

Step 4: User A completes booking successfully.
        Lock is released after booking is saved to PostgreSQL.
```

> `EX 30` means the lock expires in 30 seconds automatically.
> This prevents deadlocks if the server crashes while holding the lock.

---

## 6. AVAILABILITY & FAULT TOLERANCE PATTERNS

| Pattern | Purpose | Applied To |
|---|---|---|
| **Circuit Breaker** | If a downstream service fails repeatedly, stop calling it temporarily and return a fallback response instead of waiting | Payment Service -> Stripe API calls |
| **Retry with Exponential Backoff** | On failure, retry after 1s, then 2s, then 4s — avoids hammering a struggling service | All inter-service REST calls |
| **Idempotency Keys** | Same request can be retried safely without double-charging | Payment Service — each payment has a unique idempotency key |
| **Dead Letter Queue (DLQ)** | If a Kafka consumer fails to process an event after retries, the event is moved to DLQ for manual inspection | Notification Service, Analytics Service |
| **Health Check Endpoints** | Load balancer removes unhealthy pods from rotation | All services expose `GET /health` |
| **Multi-AZ Deployment** | Database runs in two availability zones — if one data center goes down, the other takes over automatically | PostgreSQL (AWS RDS Multi-AZ) |

### Circuit Breaker — How It Works

```
Normal state:
  Booking Service -> Stripe API  (all good)

Stripe starts failing:
  Failure 1: timeout -> log
  Failure 2: timeout -> log
  Failure 3: timeout -> CIRCUIT OPENS

Circuit OPEN state (next 60 seconds):
  Booking Service -> Circuit Breaker -> returns error immediately
                     (Stripe is NOT called at all)
  User sees: "Payment temporarily unavailable. Try again in a moment."

After 60 seconds:
  Circuit HALF-OPEN -> one test request to Stripe
  If successful -> CIRCUIT CLOSES -> normal operation resumes
  If still failing -> CIRCUIT stays OPEN -> try again later
```

---

## 7. PERFORMANCE TARGETS (SLA)

| Metric | Target | How Achieved |
|---|---|---|
| Vehicle search API (p95) | < 200ms | Redis cache + ElasticSearch |
| Booking creation (p95) | < 500ms | Redis lock + async payment intent |
| Payment processing (p95) | < 2 seconds | Stripe API + async confirmation |
| System uptime | 99.9% (< 9 hours downtime/year) | Multi-AZ, auto-failover, health checks |
| Peak concurrent users | 50,000 | Horizontal auto-scaling |
| Auto-scale trigger | CPU > 70% for 2 minutes | AWS Auto Scaling Group |
| New instance ready | Within 2 minutes | Pre-warmed AMIs / Docker images |
| DB query time (cached) | < 1ms | Redis in-memory lookup |
| DB query time (uncached) | < 50ms | Indexes + read replicas |

---

## 8. CONTENT DELIVERY FOR IMAGES (CDN)

Vehicle images must NOT be served from the application server — it wastes compute and is slow.

```
Vehicle owner uploads image
        |
        v
Vehicle Service -> stores file in AWS S3 (durable, cheap storage)
S3 URL returned and saved in VEHICLE_IMAGES table

User views vehicle listing
        |
        v
Browser requests image from CloudFront CDN
(serves from the nearest edge location — Mumbai user gets Mumbai edge server)
        |
   Cache HIT? --YES--> Served from edge (< 20ms latency)
        |
       NO
        |
        v
CloudFront fetches original from S3
-> caches at edge location
-> serves to user

Next user in Mumbai who requests same image:
  -> Served instantly from edge cache
  -> S3 and application server not touched at all
```

This keeps **all image traffic completely off the application servers**, reducing server load significantly.

---

## 9. SCALABILITY SUMMARY

```
Layer               Scaling Strategy
─────────────────   ──────────────────────────────────────────────
CDN                 Global edge network — infinite scale for static assets
API Gateway         Horizontal — stateless, add instances freely
Microservices       Horizontal — auto-scale each service independently
Redis Cache         Cache-aside — reduces DB load by 90% for reads
PostgreSQL          Read replicas for reads + connection pooling
Kafka               Partitioned topics — scale consumers horizontally
AWS S3              Infinite storage — managed by AWS
```
