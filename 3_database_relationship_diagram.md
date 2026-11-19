# Database Relationship Diagram — Online Vehicle Rental Platform

---

## TABLES & WHAT THEY STORE

### USERS
Stores everyone who uses the platform — renters, vehicle owners, and admins.
```
id            UUID        Primary Key
name          VARCHAR     Full name
email         VARCHAR     Unique — used for login
phone_number  VARCHAR     Unique
password_hash VARCHAR     Bcrypt hashed password
role          ENUM        RENTER | OWNER | ADMIN
is_verified   BOOLEAN     Email verified?
is_active     BOOLEAN     Account active?
created_at    TIMESTAMP
updated_at    TIMESTAMP
```

---

### DRIVER_LICENSES
Stores the driving license document of a user. Separate from USERS to keep signup lightweight.
```
id                  UUID     Primary Key
user_id             UUID     FK -> USERS.id  (1-to-1)
license_number      VARCHAR  Unique
issued_country      VARCHAR
expiry_date         DATE
document_url        VARCHAR  URL to file stored in AWS S3
verification_status ENUM     PENDING | VERIFIED | REJECTED
```

---

### VEHICLES
Every vehicle listed on the platform by an owner.
```
id             UUID     Primary Key
owner_id       UUID     FK -> USERS.id
make           VARCHAR  e.g. "Toyota"
model          VARCHAR  e.g. "Innova"
year           INTEGER
vehicle_type   ENUM     SEDAN | SUV | HATCHBACK | BIKE | TRUCK | VAN | LUXURY
fuel_type      ENUM     PETROL | DIESEL | ELECTRIC | CNG | HYBRID
license_plate  VARCHAR  Unique
daily_rate     DECIMAL
hourly_rate    DECIMAL
status         ENUM     AVAILABLE | BOOKED | MAINTENANCE | INACTIVE
city           VARCHAR
latitude       DECIMAL
longitude      DECIMAL
avg_rating     FLOAT    Updated on each review
total_reviews  INTEGER
created_at     TIMESTAMP
```

---

### VEHICLE_IMAGES
Photos of a vehicle. One vehicle has many photos.
```
id            UUID     Primary Key
vehicle_id    UUID     FK -> VEHICLES.id
image_url     VARCHAR  URL in AWS S3 / CDN
is_primary    BOOLEAN  Main display photo
display_order INTEGER  Ordering of photos in gallery
```

---

### VEHICLE_AVAILABILITY
Date ranges when a vehicle is blocked (booked or under maintenance).
```
id           UUID   Primary Key
vehicle_id   UUID   FK -> VEHICLES.id
booking_id   UUID   FK -> BOOKINGS.id  (nullable — could be manual block)
blocked_from DATE
blocked_to   DATE
reason       VARCHAR  e.g. "BOOKING" or "MAINTENANCE"
```

---

### BOOKINGS
The central table of the entire system. Every reservation made on the platform.
```
id                  UUID       Primary Key
user_id             UUID       FK -> USERS.id
vehicle_id          UUID       FK -> VEHICLES.id
start_datetime      TIMESTAMP
end_datetime        TIMESTAMP
total_days          INTEGER
base_amount         DECIMAL    dailyRate x totalDays
tax_amount          DECIMAL    18% GST
total_amount        DECIMAL    base + tax
status              ENUM       DRAFT | PENDING | CONFIRMED | ACTIVE | COMPLETED | CANCELLED | EXPIRED
pickup_location     VARCHAR
dropoff_location    VARCHAR
cancellation_reason TEXT       (nullable)
created_at          TIMESTAMP
updated_at          TIMESTAMP
```

---

### PAYMENTS
Payment record for each booking. One booking = one payment record.
```
id               UUID     Primary Key
booking_id       UUID     FK -> BOOKINGS.id  (1-to-1)
user_id          UUID     FK -> USERS.id
amount           DECIMAL
currency         VARCHAR  e.g. "INR"
method           ENUM     CARD | UPI | NET_BANKING | WALLET
status           ENUM     PENDING | SUCCESS | FAILED | REFUNDED
gateway_txn_id   VARCHAR  Transaction ID from Stripe/Razorpay
payment_gateway  VARCHAR  "STRIPE" or "RAZORPAY"
paid_at          TIMESTAMP
refund_amount    DECIMAL  (nullable)
refund_at        TIMESTAMP (nullable)
```

---

### REVIEWS
User reviews submitted after a completed rental.
```
id           UUID     Primary Key
booking_id   UUID     FK -> BOOKINGS.id  (1-to-0 or 1 — review is optional)
reviewer_id  UUID     FK -> USERS.id
vehicle_id   UUID     FK -> VEHICLES.id
rating       INTEGER  1 to 5
comment      TEXT
is_visible   BOOLEAN  Admin can hide inappropriate reviews
created_at   TIMESTAMP
```

---

### NOTIFICATIONS
Every email, SMS, or push notification sent to a user.
```
id        UUID      Primary Key
user_id   UUID      FK -> USERS.id
type      ENUM      BOOKING_CONFIRMED | BOOKING_CANCELLED | PAYMENT_SUCCESS | REFUND_ISSUED | REMINDER
channel   VARCHAR   "EMAIL" | "SMS" | "PUSH"
message   TEXT      Content of notification
metadata  JSONB     Extra data (booking ID, amount etc.)
is_read   BOOLEAN
sent_at   TIMESTAMP
```

---

## ENTITY RELATIONSHIP DIAGRAM

```
USERS
  |
  |---(1:1)-----> DRIVER_LICENSES
  |
  |---(1:Many)--> VEHICLES
  |                   |
  |                   |---(1:Many)--> VEHICLE_IMAGES
  |                   |
  |                   '---(1:Many)--> VEHICLE_AVAILABILITY
  |                                         ^
  |                                         |
  |---(1:Many)--> BOOKINGS -----------------' (confirmed booking blocks dates)
  |                   |
  |                   |---(1:1)-------> PAYMENTS
  |                   |
  |                   '---(1:0 or 1)--> REVIEWS
  |
  '---(1:Many)--> NOTIFICATIONS
```

---

## RELATIONSHIPS SUMMARY

| Relationship | Type | Description |
|---|---|---|
| USERS → DRIVER_LICENSES | One-to-One | Each user has exactly one license record |
| USERS → VEHICLES | One-to-Many | An owner (user) can list many vehicles |
| USERS → BOOKINGS | One-to-Many | A user can make many bookings over time |
| USERS → REVIEWS | One-to-Many | A user can write many reviews |
| USERS → NOTIFICATIONS | One-to-Many | A user receives many notifications |
| VEHICLES → BOOKINGS | One-to-Many | A vehicle can have many bookings (across different dates) |
| VEHICLES → VEHICLE_IMAGES | One-to-Many | A vehicle has many photos |
| VEHICLES → VEHICLE_AVAILABILITY | One-to-Many | A vehicle has many blocked date ranges |
| VEHICLES → REVIEWS | One-to-Many | A vehicle receives many reviews from different renters |
| BOOKINGS → PAYMENTS | One-to-One | Each booking has exactly one payment record |
| BOOKINGS → REVIEWS | One-to-Zero-or-One | A booking may generate one review (optional) |
| BOOKINGS → VEHICLE_AVAILABILITY | One-to-Many | A confirmed booking blocks date ranges |

---

## KEY CONSTRAINTS

```
-- Unique Constraints
USERS.email
USERS.phone_number
VEHICLES.license_plate
DRIVER_LICENSES.license_number

-- Foreign Keys
VEHICLES.owner_id              -> USERS.id
BOOKINGS.user_id               -> USERS.id
BOOKINGS.vehicle_id            -> VEHICLES.id
PAYMENTS.booking_id            -> BOOKINGS.id
REVIEWS.booking_id             -> BOOKINGS.id
REVIEWS.reviewer_id            -> USERS.id
REVIEWS.vehicle_id             -> VEHICLES.id
VEHICLE_AVAILABILITY.vehicle_id -> VEHICLES.id
VEHICLE_AVAILABILITY.booking_id -> BOOKINGS.id  (nullable)
VEHICLE_IMAGES.vehicle_id      -> VEHICLES.id
NOTIFICATIONS.user_id          -> USERS.id
DRIVER_LICENSES.user_id        -> USERS.id
```

---

## PERFORMANCE INDEXES

```sql
-- Speed up vehicle search by city and status (most common query)
CREATE INDEX idx_vehicles_city_status
  ON vehicles(city, status);

-- Speed up vehicle type + price filtering
CREATE INDEX idx_vehicles_type_rate
  ON vehicles(vehicle_type, daily_rate);

-- Speed up booking history queries per user
CREATE INDEX idx_bookings_user_id
  ON bookings(user_id);

-- Speed up availability checks (critical for preventing double-booking)
CREATE INDEX idx_bookings_vehicle_dates
  ON bookings(vehicle_id, start_datetime, end_datetime);

-- Speed up availability block lookups
CREATE INDEX idx_availability_vehicle_dates
  ON vehicle_availability(vehicle_id, blocked_from, blocked_to);

-- Only index available vehicles (partial index — smaller, faster)
CREATE INDEX idx_active_vehicles
  ON vehicles(city, vehicle_type)
  WHERE status = 'AVAILABLE';
```

---

## ENUMs

```
UserRole:          RENTER, OWNER, ADMIN
VehicleType:       SEDAN, SUV, HATCHBACK, BIKE, TRUCK, VAN, LUXURY
FuelType:          PETROL, DIESEL, ELECTRIC, CNG, HYBRID
VehicleStatus:     AVAILABLE, BOOKED, MAINTENANCE, INACTIVE
BookingStatus:     DRAFT, PENDING, CONFIRMED, ACTIVE, COMPLETED, CANCELLED, EXPIRED
PaymentMethod:     CARD, UPI, NET_BANKING, WALLET
PaymentStatus:     PENDING, SUCCESS, FAILED, REFUNDED
LicenseStatus:     PENDING, VERIFIED, REJECTED
NotificationType:  BOOKING_CONFIRMED, BOOKING_CANCELLED, PAYMENT_SUCCESS, REFUND_ISSUED, REMINDER
```

---

## REAL-WORLD STORY — HOW ALL TABLES CONNECT

```
1. Priya signs up              -> 1 row in USERS
2. She uploads her license     -> 1 row in DRIVER_LICENSES
3. She searches SUVs in Mumbai -> queries VEHICLES + VEHICLE_AVAILABILITY
4. She books Toyota Innova     -> 1 row in BOOKINGS (status = PENDING)
5. She pays Rs. 8850           -> 1 row in PAYMENTS (status = SUCCESS)
                                  BOOKINGS updated (status = CONFIRMED)
                                  VEHICLE_AVAILABILITY gets new blocked row
6. She returns vehicle         -> BOOKINGS updated (status = COMPLETED)
7. She submits a review        -> 1 row in REVIEWS (rating = 5)
8. Throughout the process      -> 3 rows in NOTIFICATIONS
                                  (confirmed, reminder, payment receipt)
```
