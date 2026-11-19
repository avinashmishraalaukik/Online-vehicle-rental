# HLD & LLD Documentation — Online Vehicle Rental Platform

---

## PART A — HIGH-LEVEL DESIGN (HLD)

### 1. System Overview

The Online Vehicle Rental Platform allows users to browse, book, and pay for vehicles. Vehicle owners can list their vehicles. The system is built as a set of independent microservices communicating via REST (synchronous) and Kafka (asynchronous).

---

### 2. Core Modules

| Module | Responsibility |
|---|---|
| **User Module** | Registration, login, profile management, license upload |
| **Vehicle Module** | List vehicles, manage availability, upload images |
| **Booking Module** | Create reservation, manage booking lifecycle |
| **Payment Module** | Process payments, handle refunds |
| **Search Module** | Filter vehicles by city, date, type, price |
| **Notification Module** | Send email/SMS on booking events |

---

### 3. HLD Request Flow

```
User searches for a vehicle
        |
        v
API Gateway -> Search Service -> Returns list of available vehicles
        |
User selects vehicle & dates
        |
        v
API Gateway -> Booking Service
        |-- Checks availability      (calls Vehicle Service)
        |-- Locks the vehicle        (Redis distributed lock)
        |-- Creates booking record   (status = PENDING)
        '-- Returns booking ID + payment details to user
        |
User submits payment
        |
        v
API Gateway -> Payment Service
        |-- Charges card via Stripe
        |-- Updates booking status = CONFIRMED
        '-- Publishes BookingConfirmed event to Kafka
                |
                v
        Notification Service consumes event
                '-- Sends confirmation email + SMS to user
```

---

### 4. Technology Stack

| Layer | Technology | Why |
|---|---|---|
| Frontend | React.js (web), React Native (mobile) | Component-based, fast rendering |
| API Gateway | Nginx | Lightweight, high-performance reverse proxy |
| Backend Services | Node.js (Express) | Non-blocking I/O, great for APIs |
| Message Broker | Apache Kafka | High-throughput async event streaming |
| Primary Database | PostgreSQL | ACID transactions for financial data |
| Cache | Redis | In-memory, sub-millisecond reads |
| Search | ElasticSearch | Full-text search with filters |
| File Storage | AWS S3 | Durable, scalable object storage |
| CDN | AWS CloudFront | Global edge network for fast delivery |
| Payment | Stripe / Razorpay | Trusted payment gateway |
| Email | SendGrid | Reliable transactional email |
| SMS | Twilio | Programmable SMS at scale |

---

### 5. Non-Functional Requirements

| Requirement | Target |
|---|---|
| Availability | 99.9% uptime |
| Search latency | < 300ms |
| Booking creation | < 500ms |
| Payment processing | < 2 seconds |
| Concurrent users | Up to 50,000 |
| Data consistency | Strong consistency for bookings and payments |
| Data durability | No loss of booking or payment data |

---

### 6. Architecture Style — Monolithic vs Microservices

| Dimension | Monolithic | Microservices (chosen) |
|---|---|---|
| Deployment | Single unit | Each service deploys independently |
| Scaling | Scale entire app | Scale only the bottleneck service |
| Failure | One crash affects all | Failures are isolated per service |
| Team ownership | One team | Each team owns one service |
| Complexity | Simple at start | Higher operational overhead |

> This platform uses **Microservices** because different parts have different scaling needs.
> (e.g. Search gets 10x more traffic than Payments)

---

---

## PART B — LOW-LEVEL DESIGN (LLD)

### 1. Layered Architecture (MVC Pattern)

```
+----------------------------------+
|       Presentation Layer          |   -> React / React Native frontend
+----------------------------------+
|        Controller Layer           |   -> REST API route handlers
+----------------------------------+
|         Service Layer             |   -> Business logic
+----------------------------------+
|        Repository Layer           |   -> Database access (SQL queries)
+----------------------------------+
|         Database Layer            |   -> PostgreSQL / Redis
+----------------------------------+
```

Each layer has a **single responsibility** — a change in the DB schema does not break UI code.

---

### 2. Booking Service — Internal Structure

```
BookingController        <- HTTP route handlers
  POST /bookings         -> createBooking()
  GET  /bookings/:id     -> getBookingById()
  GET  /bookings/me      -> getMyBookings()
  PUT  /bookings/:id/cancel -> cancelBooking()

BookingService           <- Business logic
  createBooking(dto)
    |-- validateDates()               check start < end, min 1 day
    |-- checkVehicleAvailability()    call Vehicle Service (REST)
    |-- acquireDistributedLock()      Redis SETNX on vehicleId + dates
    |-- calculateTotalCost()          days x dailyRate + 18% tax
    |-- createPaymentIntent()         call Payment Service
    '-- save(booking)                 BookingRepository -> PostgreSQL

  confirmBooking(bookingId)
    |-- verifyPaymentSuccess()
    |-- updateStatus(CONFIRMED)
    |-- blockVehicleDates()           call Vehicle Service
    '-- publishEvent(BookingConfirmed) -> Kafka

  cancelBooking(bookingId, reason)
    |-- validateCancellationWindow()  >24h before = full refund
    |-- initiateRefund()              call Payment Service
    |-- updateStatus(CANCELLED)
    |-- releaseDates()                call Vehicle Service
    '-- publishEvent(BookingCancelled) -> Kafka

BookingRepository        <- Data access layer
  save(booking)
  findById(id)
  findByUserId(userId)
  updateStatus(id, status)
```

---

### 3. Booking State Machine

```
  [User selects vehicle & dates]
           |
           v
        DRAFT
           | user submits booking form
           v
        PENDING <----------------------------+
           | payment initiated               | user retries
           v                                 |
   PAYMENT_PROCESSING                        |
           |             |                   |
           v             v                   |
      CONFIRMED    PAYMENT_FAILED -----------+
           |
           | rental start date reached
           v
         ACTIVE
           |
           | vehicle returned
           v
       COMPLETED


  PENDING   ----> EXPIRED    (no payment within 15 minutes)
  CONFIRMED ----> CANCELLED  (user cancels more than 24h before start)
  ACTIVE    ----> CANCELLED  (owner cancels — penalty applied)
```

---

### 4. Core Domain Classes

```
User
  id            : UUID
  name          : String
  email         : String  (unique)
  phone         : String
  passwordHash  : String
  role          : ENUM (RENTER, OWNER, ADMIN)
  isVerified    : Boolean
  createdAt     : DateTime

Vehicle
  id            : UUID
  ownerId       : UUID -> User
  make          : String        e.g. "Toyota"
  model         : String        e.g. "Innova"
  year          : Integer
  type          : ENUM (SEDAN, SUV, HATCHBACK, BIKE, TRUCK, VAN, LUXURY)
  fuelType      : ENUM (PETROL, DIESEL, ELECTRIC, CNG, HYBRID)
  licensePlate  : String (unique)
  dailyRate     : Decimal
  status        : ENUM (AVAILABLE, BOOKED, MAINTENANCE, INACTIVE)
  city          : String
  avgRating     : Float

Booking
  id            : UUID
  userId        : UUID -> User
  vehicleId     : UUID -> Vehicle
  startDatetime : DateTime
  endDatetime   : DateTime
  totalDays     : Integer
  totalAmount   : Decimal
  status        : ENUM (DRAFT, PENDING, CONFIRMED, ACTIVE, COMPLETED, CANCELLED, EXPIRED)
  pickupLocation: String
  createdAt     : DateTime

Payment
  id            : UUID
  bookingId     : UUID -> Booking
  userId        : UUID -> User
  amount        : Decimal
  currency      : String
  method        : ENUM (CARD, UPI, NET_BANKING, WALLET)
  status        : ENUM (PENDING, SUCCESS, FAILED, REFUNDED)
  gatewayTxnId  : String
  paidAt        : DateTime

Review
  id            : UUID
  bookingId     : UUID -> Booking
  reviewerId    : UUID -> User
  vehicleId     : UUID -> Vehicle
  rating        : Integer (1 to 5)
  comment       : String
  createdAt     : DateTime
```

---

### 5. SOLID Principles Applied

| Principle | Meaning | Applied in This Platform |
|---|---|---|
| **Single Responsibility** | A class does one thing only | `BookingService` only handles bookings. `PaymentService` only handles payments. Never mixed. |
| **Open/Closed** | Open for extension, closed for modification | Adding a new vehicle type (e.g. EV) extends the `Vehicle` model — no existing code changes needed. |
| **Liskov Substitution** | Subtypes are substitutable for base types | `ElectricVehicle` extends `Vehicle` and works in all booking flows without breaking anything. |
| **Interface Segregation** | Many small interfaces > one fat interface | `IBookable`, `ISearchable`, `IPayable` are separate interfaces — a service only implements what it needs. |
| **Dependency Inversion** | Depend on abstractions, not concretions | `BookingService` depends on `IPaymentGateway` interface, not `StripeService` directly. Switching gateways needs zero changes in `BookingService`. |

---

### 6. Separation of Concerns

```
Controller  -> ONLY handles HTTP request and response
Service     -> ONLY contains business logic
Repository  -> ONLY talks to the database
Model       -> ONLY defines the data structure

Example — creating a booking:

BookingController.createBooking(req, res)
   |
   '--> BookingService.createBooking(dto)       <- business logic
              |
              '--> BookingRepository.save(booking)  <- DB write
              |
              '--> VehicleService.checkAvailability()  <- another service
              |
              '--> PaymentService.createIntent()   <- another service
```

None of these layers know or care about what the others look like internally.

This is all about HLD and LLD
