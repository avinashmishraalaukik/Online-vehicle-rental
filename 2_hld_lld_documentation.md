# HLD & LLD Documentation — Online Vehicle Rental Platform

---

## PART A — HIGH-LEVEL DESIGN (HLD)

### 1. System Overview

Cloud-native microservices platform for vehicle rentals. **Renters** search & book, **Owners** list & track earnings, **Admins** monitor & manage. REST handles real-time calls; Kafka handles async background events.

---

### 2. Core Modules

| Module | Responsibility | Scaling |
|---|---|---|
| **User** | Registration, login, JWT, license upload | Stateless — horizontal |
| **Vehicle** | Listings, availability, image upload | Redis-cached — horizontal |
| **Booking** | Reservations, lifecycle, state machine | Redis distributed locks |
| **Payment** | Charge, refund, receipts | PCI-isolated service |
| **Search** | Filter by city, date, type, price | ElasticSearch cluster |
| **Notification** | Email + SMS on booking/payment events | Kafka consumer group |
| **Review** | Ratings after completed bookings | High read, low write |
| **Analytics** | Revenue dashboards, booking stats | Read replicas + batch |

---

### 3. HLD Request Flow

```
[1] SEARCH  →  API Gateway → Search Service (ElasticSearch) → paginated results

[2] BOOK
  API Gateway → Booking Service
    |── Validate dates & license
    |── Check availability     (REST → Vehicle Service)
    |── Acquire Redis lock     (SETNX on vehicleId+dates, TTL 600s)
    |── Calculate cost         (days × dailyRate + 18% GST)
    |── Save booking           (status = PENDING) → PostgreSQL
    └── Return bookingId + paymentDeadline

[3] PAY
  API Gateway → Payment Service
    |── Charge via Stripe/Razorpay
    |── Update booking → CONFIRMED
    └── Publish BookingConfirmed → Kafka
              ├── Notification Service  (email + SMS)
              └── Vehicle Service       (block dates)

[4] USE & RETURN
  Rental start date  → booking = ACTIVE
  Vehicle returned   → booking = COMPLETED → user submits review
```

---

### 4. Technology Stack

| Layer | Technology | Why |
|---|---|---|
| Frontend | React.js + React Native | Component-based, shared codebase |
| API Gateway | Nginx / Kong | JWT auth, rate limiting, routing |
| Backend | Node.js (Express) | Non-blocking I/O for high concurrency |
| Message Broker | Apache Kafka | High-throughput fault-tolerant async events |
| Database | PostgreSQL | ACID — mandatory for financial data |
| Cache / Locks | Redis | Sub-ms reads, distributed locking, sessions |
| Search | ElasticSearch | Full-text, geospatial, millisecond latency |
| File Storage | AWS S3 + CloudFront | Durable storage, global edge delivery |
| Payment | Stripe / Razorpay | PCI-DSS compliant, cards/UPI/wallets |
| Notifications | SendGrid + Twilio | Transactional email & programmable SMS |
| Infra | Docker + Kubernetes | Isolation, auto-scaling, rolling deploys |
| Monitoring | Prometheus + Grafana + ELK | Metrics, alerts, centralized logs |

---

### 5. Non-Functional Requirements

| Requirement | Target | How |
|---|---|---|
| Availability | 99.9% uptime | Multi-AZ, circuit breakers, health checks |
| Search latency | < 300 ms | ElasticSearch + Redis query cache |
| Booking creation | < 500 ms | Async confirmations via Kafka |
| Payment | < 2 seconds | Direct Stripe integration |
| Concurrent users | 50,000 | Kubernetes HPA — horizontal pod autoscaling |
| Data consistency | Strong (bookings/payments) | PostgreSQL ACID + Redis distributed lock |
| Data durability | Zero loss | PostgreSQL WAL + S3 backups + Kafka replication |

---

### 6. Architecture Style — Why Microservices

| Dimension | Monolithic | Microservices ✅ |
|---|---|---|
| Deployment | Single unit | Each service deploys independently |
| Scaling | Scale entire app | Scale only the bottleneck service |
| Fault isolation | One crash = all down | Notification down ≠ Booking down |
| Team ownership | Shared codebase | Each team owns one service |

> Search gets **10× more traffic** than Payments — different scaling, latency, and consistency needs demand independent services.

---

### 7. Security Architecture

```
TLS 1.3 everywhere  →  API Gateway (JWT validate + rate limit 100 req/min/user)
  →  Input validation (Joi/Zod schemas)
  →  Parameterized SQL (no injection risk)
  →  bcrypt passwords (cost factor 12)
  →  Stripe tokenization (card numbers never stored)
  →  AWS Secrets Manager (no hardcoded secrets)
  →  Prometheus alerts on anomalous traffic (DDoS detection)
```

---

### 8. Service Communication

```
REST (synchronous):        Booking ──► Vehicle (availability), Payment (intent)
                           Payment ──► Stripe API

Kafka (async, fire & forget):
  Booking Service  ──►  BookingConfirmed   ──►  Notification, Vehicle Service
  Booking Service  ──►  BookingCancelled   ──►  Notification, Vehicle Service
  Payment Service  ──►  PaymentSuccess     ──►  Booking Service
  Payment Service  ──►  RefundIssued       ──►  Notification Service
```

---
---

## PART B — LOW-LEVEL DESIGN (LLD)

### 1. Layered Architecture

```
Presentation  →  React / React Native (no business logic)
Controller    →  Parse HTTP request → call service → return response + middleware
Service       →  ALL business logic, orchestration, event publishing
Repository    →  CRUD only — SQL queries, no logic
Database      →  PostgreSQL (persistent) + Redis (cache + locks + sessions)
```

One responsibility per layer — a DB schema change never breaks Controller code.

---

### 2. Booking Service — Internal Design

```
BookingController  (HTTP only — no logic)
  POST /bookings            → createBooking()
  GET  /bookings/:id        → getBookingById()
  GET  /bookings/me         → getMyBookings()
  PUT  /bookings/:id/cancel → cancelBooking()
  PUT  /bookings/:id/extend → extendBooking()

BookingService  (all business logic)
  createBooking(dto)
    1. validateDates()           start < end, min 1 day, max 30 days
    2. verifyUserLicense()       call User Service — license uploaded & verified
    3. checkAvailability()       call Vehicle Service (REST)
    4. acquireRedisLock()        SETNX lock:vehicle:{id}:{dates} TTL=600s → 409 if fails
    5. calculateCost()           base = days × rate | tax = base × 0.18 | total = base + tax
    6. createPaymentIntent()     call Payment Service → returns clientSecret
    7. saveBooking()             status = PENDING → PostgreSQL

  confirmBooking(bookingId)
    verifyPayment() → updateStatus(CONFIRMED) → blockDates() → releaseRedisLock()
    → publishEvent(BookingConfirmed) → Kafka

  cancelBooking(bookingId, reason)
    validateOwnership() → check window (≥24h=full refund, <24h=50% refund)
    → initiateRefund() → updateStatus(CANCELLED) → releaseDates()
    → publishEvent(BookingCancelled) → Kafka

  extendBooking(bookingId, newEnd)
    validateNewEnd() → checkAvailability(currentEnd, newEnd)
    → chargeExtra() → updateEndDatetime() → publishEvent(BookingExtended)

BookingRepository  (data access only)
  save() | findById() | findByUserId() | updateStatus() | findExpiredPendingBookings()
```

---

### 3. Booking State Machine

```
DRAFT → PENDING → PAYMENT_PROCESSING → CONFIRMED → ACTIVE → COMPLETED
                        ↓
                  PAYMENT_FAILED → retry → PENDING

PENDING    ──► EXPIRED     (no payment in 15 min — cron job)
CONFIRMED  ──► CANCELLED   (≥24h before: full refund | <24h: 50% refund)
ACTIVE     ──► CANCELLED   (owner cancels: 100% refund + owner penalty)
CONFIRMED  ──► EXTENDED    (renter extends: additional charge applied)
```

---

### 4. Core Domain Classes

```
User:     id, name, email(unique), phone, passwordHash, role(RENTER|OWNER|ADMIN),
          isVerified, licenseUrl, licenseVerified, createdAt

Vehicle:  id, ownerId→User, make, model, year, type(SEDAN|SUV|BIKE|LUXURY|EV...),
          fuelType, transmission, seatingCapacity, licensePlate(unique),
          dailyRate, status(AVAILABLE|BOOKED|MAINTENANCE), city,
          latitude, longitude, imageUrls[], avgRating, totalReviews

Booking:  id, userId→User, vehicleId→Vehicle, startDatetime, endDatetime,
          totalDays, baseAmount, taxAmount, totalAmount,
          status(DRAFT|PENDING|CONFIRMED|ACTIVE|COMPLETED|CANCELLED|EXPIRED|EXTENDED),
          pickupLocation, dropoffLocation, refundAmount, cancellationReason

Payment:  id, bookingId→Booking, userId→User, amount, currency, 
          method(CARD|UPI|NET_BANKING|WALLET),
          status(PENDING|PROCESSING|SUCCESS|FAILED|REFUNDED|PARTIAL_REFUND),
          gatewayTxnId, gatewayResponse(JSONB), paidAt, refundedAt

Review:   id, bookingId→Booking(unique), reviewerId→User, vehicleId→Vehicle,
          rating(1-5), comment(max 1000), ownerReply, isVisible, createdAt
```

---

### 5. Design Patterns Applied

**Repository Pattern** — Service depends on `IBookingRepository` interface, not the Postgres class. Swapping DB = zero changes in Service.

**Strategy Pattern** — `PaymentService` depends on `IPaymentGateway`. `StripeGateway` and `RazorpayGateway` both implement it. Switching gateways = one config line.

**Observer Pattern** — Booking Service publishes to Kafka. `NotificationService`, `VehicleService`, `AnalyticsService` consume independently. Adding a new consumer = zero changes upstream.

**Factory Pattern** — `NotificationFactory.create('EMAIL'|'SMS'|'PUSH')` returns the right notifier. NotificationService never hardcodes channel classes.

---

### 6. SOLID Principles

| Principle | How Applied |
|---|---|
| **Single Responsibility** | `BookingService` = booking logic only. `PaymentService` = payments only. Controller = HTTP only. |
| **Open / Closed** | New vehicle type → add enum value. New payment gateway → new class. Zero existing code changes. |
| **Liskov Substitution** | `StripeGateway` and `RazorpayGateway` are interchangeable via `IPaymentGateway`. |
| **Interface Segregation** | `IBookable`, `ISearchable`, `IPayable` are separate — no service implements unused methods. |
| **Dependency Inversion** | `BookingService` depends on `IPaymentGateway` + `IBookingRepository` — never concrete classes. |

---

### 7. Caching & Error Strategy

**Redis Caching:**
```
vehicle:{id}                  TTL 30m   Vehicle detail page
search:{city}:{type}:{dates}  TTL 5m    Search results
user:session:{userId}         TTL 15m   Session claims
refresh_token:{token}         TTL 7d    Refresh token store
```
Invalidate vehicle cache on: BookingConfirmed, Vehicle update, Review submitted.

**Error Handling:**
```
Validation errors (400)   → Joi/Zod middleware — stopped before Service layer
Business errors (409/422) → Custom error classes thrown by Service
External failures (503)   → Circuit breaker — opens after 5 failures in 30s
Kafka consumer errors     → Retry 3× with backoff → Dead Letter Queue → PagerDuty alert
All errors return:  { code, message, statusCode, timestamp, requestId }
```

---

### 8. Key Database Indexes

```
users:               email, phone
vehicles:            city, status, type, ownerId | COMPOSITE(city, status, type)
bookings:            userId, vehicleId, status, startDatetime
vehicle_availability: COMPOSITE(vehicleId, blockedDate)
payments:            bookingId, gatewayTxnId
```
