# System Architecture Diagram — Online Vehicle Rental Platform

---

## REQUEST FLOW — Two Paths

```
Every user request takes ONE of two paths:

PATH 1 — Static Content (HTML, CSS, JS, Images)
─────────────────────────────────────────────────
  USER
   |
   v
  CDN (CloudFront)  ---- Cache HIT? ---YES---> Return file instantly
                              |
                             NO (Cache MISS)
                              |
                              v
                           AWS S3 (fetch file -> cache it -> return)


PATH 2 — Dynamic Requests (search, book, pay, login)
─────────────────────────────────────────────────────
  USER
   |
   v
  CDN (passes through — no cache for dynamic data)
   |
   v
  API GATEWAY
  [1] Validates JWT token
  [2] Checks rate limit
  [3] Routes to correct service
   |
   v
  LOAD BALANCER
  (distributes traffic across service instances)
   |
   v
  MICROSERVICE (Booking / Vehicle / Payment / User / Search)
   |
   |---> [1] Check REDIS CACHE first
   |           |
   |      Cache HIT? --YES--> Return data instantly (no DB needed)
   |           |
   |          NO (Cache MISS)
   |           |
   v           v
   '---> [2] Query POSTGRESQL (fetch -> store in Redis -> return)
```

+------------------------------------------------------------------+
|                           CLIENTS                                |
|               Web App    Mobile App    Admin Panel               |
+----------------------------+-------------------------------------+
                             |
                             v
+------------------------------------------------------------------+
|                      CDN (CloudFront)                            |
|            Serves: HTML, CSS, JS, Vehicle Images                 |
|       Static files served from nearest edge location             |
+----------------------------+-------------------------------------+
                             | Dynamic requests pass through
                             v
+------------------------------------------------------------------+
|                        API GATEWAY                               |
|      Validates JWT Token  |  Rate Limiting  |  Routing          |
+----------------------------+-------------------------------------+
                             |
                             v
+------------------------------------------------------------------+
|                      LOAD BALANCER                               |
|          Distributes traffic across service instances            |
+-----+----------+----------+----------+----------+---------------+
      |          |          |          |          |
      v          v          v          v          v
 +---------+ +--------+ +--------+ +--------+ +----------+
 |                     v
+------------------------------------------------------------------+
|                 REDIS CACHE  (checked first)                     |
|   Vehicle listings | User sessions | Availability calendar       |
|                                                                  |
|   Cache HIT  --> return data immediately (no DB query needed)    |
|   Cache MISS --> go to PostgreSQL                                |
+----------------------------+-------------------------------------+
                             | Cache MISS only
                             v
+------------------------------------------------------------------+
|                     POSTGRESQL DATABASE                          |
|        Users | Vehicles | Bookings | Payments | Reviews          |
|                                                                  |
|   Fetched data is stored back in Redis for next request          |
+------------------------------------------------------------------+


ASYNC EVENTS (happen in background — non-blocking)
───────────────────────────────────────────────────
  Booking Service ---publishes---> KAFKA ---consumed by---> Notification Service
  Payment Service ---publishes---> KAFKA ---consumed by---> Analytics Service

  Notification Service sends: Email (SendGrid) + SMS (Twilio)


FILE STORAGE
─────────────
  Vehicle images uploaded by owners ---stored---> AWS S3
  S3 URLs cached and served via CDN to users
```

---

## WHERE REDIS FITS — Clarified

```
  Request arrives at Service
          |
          v
  Service checks REDIS first
          |
   .------+------.
   |             |
  HIT           MISS
   |             |
   v             v
Return        Query
cached        PostgreSQL
data               |
                   v
              Store result
              in Redis
              (for next time)
                   |
                   v
              Return data
```

> Redis is INSIDE the service layer — after API Gateway, before PostgreSQL.
> The API Gateway still processes every request first (auth, rate limit, routing)
> then the service checks Redis before hitting the database.

---

## COMPONENT SUMMARY

| Component | Position | Job |
|---|---|---|
| **CDN** | Before API Gateway | Serves static files from edge — never reaches backend |
| **API Gateway** | Entry point for all dynamic requests | Auth · Rate limiting · Routing |
| **Load Balancer** | After API Gateway | Spreads traffic across service instances |
| **Microservices** | Core logic layer | Business rules — booking, payment, search etc. |
| **Redis** | Inside service layer (before DB) | Cache frequent data — avoids repeated DB queries |
| **PostgreSQL** | Last resort for data | Persistent storage — only hit on cache miss |
| **Kafka** | Async side channel | Decouples services — fire events without waiting |
| **AWS S3** | File storage | Stores vehicle images — served via CDN |

---

## MICROSERVICES OVERVIEW

| Service | Port | Responsibilities |
|---|---|---|
| **User Service** | 3001 | Register, login, JWT, profile, license upload |
| **Vehicle Service** | 3002 | Vehicle catalog, availability, image upload |
| **Booking Service** | 3003 | Create/cancel reservations, state machine |
| **Payment Service** | 3004 | Charge cards, refunds, receipts |
| **Search Service** | 3005 | ElasticSearch-backed vehicle search |
| **Notification Service** | 3006 | Email via SendGrid, SMS via Twilio |
| **Review Service** | 3007 | Submit and fetch vehicle reviews |
| **Analytics Service** | 3008 | Booking stats, revenue reports |
