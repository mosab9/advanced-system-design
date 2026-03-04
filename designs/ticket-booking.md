# Design a Ticket Booking System (BookMyShow/Ticketmaster)

Reservation system with inventory management and concurrency handling.

---

## 1. Requirements

### Functional
- Browse events/shows
- View seat availability
- Select seats and book
- Process payment
- Generate e-ticket

### Non-Functional
- Handle concurrent bookings (no double-booking)
- High availability
- Low latency seat selection
- Handle traffic spikes (popular events)

---

## 2. High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│   ┌─────────────┐     ┌─────────────────────────────────────────┐  │
│   │   Client    │────▶│           API Gateway                   │  │
│   └─────────────┘     │    (Auth, Rate Limiting)                │  │
│                       └──────────────────┬──────────────────────┘  │
│                                          │                          │
│         ┌────────────────────────────────┼────────────────────┐    │
│         │                                │                    │    │
│         ▼                                ▼                    ▼    │
│   ┌───────────┐               ┌───────────────┐       ┌─────────┐ │
│   │  Event    │               │   Booking     │       │ Payment │ │
│   │ Catalog   │               │   Service     │       │ Service │ │
│   └───────────┘               └───────────────┘       └─────────┘ │
│                                      │                            │
│                                      ▼                            │
│                               ┌───────────────┐                   │
│                               │  Inventory    │                   │
│                               │  Service      │                   │
│                               │(Seat Locking) │                   │
│                               └───────────────┘                   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 3. The Double-Booking Problem

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Concurrency Challenge                           │
│                                                                     │
│   User A and User B both want Seat #42                             │
│                                                                     │
│   Without proper locking:                                          │
│                                                                     │
│   Time    User A                  User B                           │
│   ────    ──────                  ──────                           │
│   T1      Check seat #42: FREE                                     │
│   T2                              Check seat #42: FREE             │
│   T3      Book seat #42 ✓                                          │
│   T4                              Book seat #42 ✓ (DOUBLE BOOK!)   │
│                                                                     │
│   Solutions:                                                       │
│   1. Pessimistic locking (DB lock)                                │
│   2. Optimistic locking (version number)                          │
│   3. Temporary hold (Redis with TTL)                              │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 4. Seat Locking Strategy

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Temporary Hold Pattern                          │
│                                                                     │
│   Step 1: User selects seat → Create temporary hold (10 min)      │
│                                                                     │
│   Redis: SETEX seat:event123:seat42 600 user_456                  │
│   (Key expires in 600 seconds)                                     │
│                                                                     │
│   Step 2: User completes payment within 10 min                     │
│   → Convert hold to permanent booking                              │
│   → Delete Redis key, update DB                                    │
│                                                                     │
│   Step 3: User abandons / timeout                                  │
│   → Redis key expires automatically                                │
│   → Seat becomes available again                                   │
│                                                                     │
│   Checking availability:                                           │
│   - Check Redis for holds                                          │
│   - Check DB for permanent bookings                                │
│   - Seat available if not in either                               │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 5. Booking Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Booking Flow                                    │
│                                                                     │
│   1. User views available seats                                    │
│      GET /events/{id}/seats                                        │
│      → Return seat map with availability                          │
│                                                                     │
│   2. User selects seats                                            │
│      POST /events/{id}/hold                                        │
│      Body: { seats: ["A1", "A2"] }                                │
│      → Try to acquire locks (Redis SETNX)                         │
│      → Success: Return hold_id, expires_at                        │
│      → Failure: Return "seats unavailable"                        │
│                                                                     │
│   3. User proceeds to payment                                      │
│      POST /bookings                                                │
│      Body: { hold_id: "...", payment_method: "..." }              │
│      → Verify hold still valid                                    │
│      → Process payment                                             │
│      → Create booking record                                       │
│      → Release hold, mark seats as booked                         │
│      → Generate e-ticket                                           │
│                                                                     │
│   4. Timeout handling                                              │
│      → Redis TTL expires                                           │
│      → Seats automatically released                               │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 6. Data Model

```sql
-- Events
CREATE TABLE events (
    id              UUID PRIMARY KEY,
    name            VARCHAR(200),
    venue_id        UUID,
    event_date      TIMESTAMP,
    total_seats     INT,
    available_seats INT
);

-- Seats
CREATE TABLE seats (
    id              UUID PRIMARY KEY,
    event_id        UUID,
    section         VARCHAR(20),
    row             VARCHAR(10),
    seat_number     VARCHAR(10),
    price           DECIMAL(10, 2),
    status          VARCHAR(20),  -- AVAILABLE, HELD, BOOKED
    version         INT DEFAULT 0, -- For optimistic locking
    UNIQUE (event_id, section, row, seat_number)
);

-- Bookings
CREATE TABLE bookings (
    id              UUID PRIMARY KEY,
    user_id         UUID,
    event_id        UUID,
    status          VARCHAR(20),  -- PENDING, CONFIRMED, CANCELLED
    total_amount    DECIMAL(10, 2),
    created_at      TIMESTAMP
);

-- Booking items (seats in booking)
CREATE TABLE booking_items (
    booking_id      UUID,
    seat_id         UUID,
    price           DECIMAL(10, 2),
    PRIMARY KEY (booking_id, seat_id)
);
```

---

## 7. Handling High Traffic (Flash Sales)

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Handling Traffic Spikes                         │
│                                                                     │
│   Popular event: 10,000 seats, 100,000 users trying to book       │
│                                                                     │
│   Strategies:                                                      │
│                                                                     │
│   1. Virtual waiting room                                          │
│      - Queue users before they can select seats                   │
│      - Let users in gradually                                      │
│      - Show estimated wait time                                    │
│                                                                     │
│   2. Pre-registration                                              │
│      - Users register interest before sale                        │
│      - Randomly assign time slots                                  │
│                                                                     │
│   3. Rate limiting                                                 │
│      - Limit requests per user/IP                                 │
│      - CAPTCHA for suspicious activity                            │
│                                                                     │
│   4. Read replicas                                                 │
│      - Serve seat availability from replicas                      │
│      - Writes go to primary                                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 8. Seat Map Caching

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Seat Map Optimization                           │
│                                                                     │
│   Cache seat status in Redis:                                      │
│                                                                     │
│   Key: seatmap:event123                                            │
│   Value: Bitmap or Set of booked/held seats                       │
│                                                                     │
│   For visual seat map:                                             │
│   - Pre-render seat map images for common states                  │
│   - Update incrementally on changes                               │
│   - Use WebSocket for real-time updates                           │
│                                                                     │
│   Optimization:                                                    │
│   - Don't fetch all seats, paginate by section                    │
│   - Cache "available count" per section                           │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 9. Interview Tips

**Key Points:**
- Seat locking strategy (pessimistic vs optimistic vs temporary hold)
- Handling concurrent bookings (no double-booking)
- Payment integration with timeout handling
- Traffic spike handling (virtual waiting room)
- Seat map caching and real-time updates

**Questions to Ask:**
- Expected traffic patterns?
- Seat selection (choose specific seat vs best available)?
- Refund/cancellation policy?
- Multi-event or single venue?
