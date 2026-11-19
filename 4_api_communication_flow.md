# API Communication Flow — Online Vehicle Rental Platform

---

## 1. REST API ENDPOINTS

### User Service — `/api/v1/auth` and `/api/v1/users`

| Method | Endpoint | Description | Auth Required |
|---|---|---|---|
| `POST` | `/api/v1/auth/register` | Register a new user | No |
| `POST` | `/api/v1/auth/login` | Login — returns JWT access token | No |
| `POST` | `/api/v1/auth/logout` | Invalidate session | Yes |
| `POST` | `/api/v1/auth/refresh` | Refresh expired access token | Yes |
| `GET` | `/api/v1/users/me` | Get own profile | Yes |
| `PUT` | `/api/v1/users/me` | Update profile | Yes |
| `POST` | `/api/v1/users/me/license` | Upload driving license | Yes |

> **Why no auth on register and login?**
> Register = user has no account yet. Login = this IS the step that creates the token.
> You cannot require a token to get a token — that would be impossible.

---

### Vehicle Service — `/api/v1/vehicles`

| Method | Endpoint | Description | Auth Required |
|---|---|---|---|
| `GET` | `/api/v1/vehicles` | Search vehicles (city, dates, type) | No |
| `GET` | `/api/v1/vehicles/:id` | Get vehicle details | No |
| `GET` | `/api/v1/vehicles/:id/availability` | Get availability calendar | No |
| `POST` | `/api/v1/vehicles` | List a new vehicle | Yes (OWNER) |
| `PUT` | `/api/v1/vehicles/:id` | Update vehicle details | Yes (OWNER) |
| `DELETE` | `/api/v1/vehicles/:id` | Remove vehicle listing | Yes (OWNER) |

---

### Booking Service — `/api/v1/bookings`

| Method | Endpoint | Description | Auth Required |
|---|---|---|---|
| `POST` | `/api/v1/bookings` | Create a booking | Yes |
| `GET` | `/api/v1/bookings/:id` | Get booking details | Yes |
| `GET` | `/api/v1/bookings/me` | List my bookings | Yes |
| `PUT` | `/api/v1/bookings/:id/cancel` | Cancel a booking | Yes |

---

### Payment Service — `/api/v1/payments`

| Method | Endpoint | Description | Auth Required |
|---|---|---|---|
| `POST` | `/api/v1/payments` | Process payment for a booking | Yes |
| `GET` | `/api/v1/payments/:id` | Get payment details | Yes |
| `POST` | `/api/v1/payments/:id/refund` | Issue a refund | Yes |
| `GET` | `/api/v1/payments/:id/receipt` | Download payment receipt | Yes |

---

### Review Service — `/api/v1/reviews`

| Method | Endpoint | Description | Auth Required |
|---|---|---|---|
| `POST` | `/api/v1/reviews` | Submit a review for a completed booking | Yes |
| `GET` | `/api/v1/vehicles/:id/reviews` | Get all reviews for a vehicle | No |

---

## 2. END-TO-END BOOKING FLOW — With Sample JSON

### STEP 1 — Search Vehicles

```
REQUEST
  GET /api/v1/vehicles?city=Mumbai&startDate=2024-02-01&endDate=2024-02-04&type=SUV

RESPONSE — 200 OK
{
  "vehicles": [
    {
      "id": "v-001",
      "make": "Toyota",
      "model": "Innova",
      "type": "SUV",
      "dailyRate": 2500,
      "city": "Mumbai",
      "avgRating": 4.6,
      "totalReviews": 38,
      "imageUrl": "https://cdn.example.com/innova.jpg"
    }
  ],
  "total": 12,
  "page": 1
}
```

---

### STEP 2 — Create Booking

```
REQUEST
  POST /api/v1/bookings
  Authorization: Bearer <JWT_TOKEN>

  {
    "vehicleId": "v-001",
    "startDatetime": "2024-02-01T09:00:00Z",
    "endDatetime": "2024-02-04T18:00:00Z",
    "pickupLocation": "Mumbai Airport T2",
    "dropoffLocation": "Mumbai Airport T2"
  }

RESPONSE — 201 Created
{
  "bookingId": "b-456",
  "status": "PENDING",
  "vehicle": {
    "make": "Toyota",
    "model": "Innova",
    "licensePlate": "MH-01-AB-1234"
  },
  "totalDays": 3,
  "baseAmount": 7500,
  "taxAmount": 1350,
  "totalAmount": 8850,
  "paymentDeadline": "2024-01-28T10:15:00Z"
}
```

---

### STEP 3 — Process Payment

```
REQUEST
  POST /api/v1/payments
  Authorization: Bearer <JWT_TOKEN>

  {
    "bookingId": "b-456",
    "method": "CARD",
    "cardToken": "tok_visa_xxxx"
  }

RESPONSE — 200 OK
{
  "paymentId": "p-789",
  "status": "SUCCESS",
  "amount": 8850,
  "currency": "INR",
  "gatewayTxnId": "ch_stripe_xyz",
  "paidAt": "2024-01-28T10:12:00Z",
  "receiptUrl": "/api/v1/payments/p-789/receipt"
}
```

---

### STEP 4 — System Actions (Background — user does not wait for these)

```
Kafka event published:
{
  "event": "BookingConfirmed",
  "bookingId": "b-456",
  "userId": "u-123",
  "vehicleId": "v-001",
  "totalAmount": 8850
}

Notification Service consumes event and sends:
  Email -> "Your Toyota Innova is booked for Feb 1-4! Amount paid: Rs. 8850"
  SMS   -> "Booking #b-456 confirmed. Pickup: Mumbai Airport T2 on Feb 1."

Vehicle Service consumes event and:
  Inserts row in VEHICLE_AVAILABILITY (blocks Feb 1-4 for vehicle v-001)
```

---

## 3. INTER-SERVICE COMMUNICATION

### Synchronous — REST (request waits for response)

```
API Gateway ---------> Booking Service
Booking Service ------> Vehicle Service    (check availability, block dates)
Booking Service ------> Payment Service    (create payment intent)
Payment Service ------> Stripe API         (charge card)
Vehicle Service ------> Google Maps API    (geo-coding for location)
```

### Asynchronous — Kafka (fire and forget — no waiting)

```
Producer              Event                   Consumer(s)
─────────────────     ─────────────────────   ──────────────────────────
Booking Service  -->  BookingConfirmed     --> Notification Service
Booking Service  -->  BookingConfirmed     --> Vehicle Service (block dates)
Booking Service  -->  BookingCancelled     --> Notification Service
Booking Service  -->  BookingCancelled     --> Vehicle Service (release dates)
Payment Service  -->  PaymentSuccess       --> Booking Service (confirm booking)
Payment Service  -->  RefundIssued         --> Notification Service
```

> **Why Kafka for these events?**
> Booking Service should NOT wait for email to be sent before responding to the user.
> It publishes the event and immediately returns a response — email happens in the background.

---

## 4. AUTHENTICATION FLOW

```
Step 1: POST /api/v1/auth/login
        Body: { "email": "priya@gmail.com", "password": "mypassword" }

Step 2: User Service validates credentials
        - Does email exist in DB?
        - Does password match stored hash? (bcrypt compare)
        - Is account verified?

Step 3: If valid -> generate two tokens:
        Access Token  (JWT, signed with secret, expires in 15 minutes)
        Refresh Token (random string, expires in 7 days, stored in Redis)

Step 4: Client stores:
        Access Token  -> in memory (JavaScript variable)
        Refresh Token -> in HTTP-only cookie (not accessible by JS)

Step 5: Every protected API request includes:
        Authorization: Bearer <access_token>

Step 6: API Gateway validates JWT on every request
        - Valid signature?
        - Not expired?
        If invalid -> 401 Unauthorized (request blocked here, never reaches service)

Step 7: When Access Token expires (after 15 mins):
        POST /api/v1/auth/refresh
        -> Refresh Token validated against Redis
        -> New Access Token issued

Step 8: On logout:
        Refresh Token deleted from Redis
        -> User is fully logged out even if old Access Token hasn't expired yet
```

---

## 5. STANDARD ERROR RESPONSE FORMAT

All services return errors in the same consistent format:

```json
{
  "error": {
    "code": "VEHICLE_NOT_AVAILABLE",
    "message": "The selected vehicle is not available for the chosen dates.",
    "statusCode": 409,
    "timestamp": "2024-01-28T10:00:00Z",
    "requestId": "req-abc-123"
  }
}
```

### HTTP Status Code Reference

| Status Code | Meaning | Example |
|---|---|---|
| `200` | Success | Vehicle list returned |
| `201` | Resource created | Booking created |
| `400` | Bad request / validation error | Missing required field |
| `401` | Unauthorized | Missing or invalid JWT token |
| `403` | Forbidden | Renter trying to delete a vehicle |
| `404` | Resource not found | Booking ID does not exist |
| `409` | Conflict | Vehicle already booked for those dates |
| `422` | Unprocessable entity | Business rule violation |
| `429` | Too many requests | Rate limit exceeded |
| `500` | Internal server error | Unexpected server crash |
