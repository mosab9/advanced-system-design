# Design Uber (Ride-Sharing Platform)

Real-time location-based matching system.

---

## 1. Requirements

### Functional Requirements
- Riders request rides
- Drivers accept/decline requests
- Real-time location tracking
- ETA calculation
- Payment processing

### Non-Functional Requirements
- Low latency matching (< 1 second)
- High availability
- Real-time updates
- Handle location spikes (events, rush hour)

---

## 2. Key Challenges

```
1. Geospatial queries: "Find drivers within 5km"
2. Real-time matching: Assign optimal driver quickly
3. Location updates: Millions of updates/second
4. Supply-demand balancing: Surge pricing
```

---

## 3. High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Clients                                     │
│              (Rider App, Driver App)                                │
└───────────────────────────────┬─────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    WebSocket Gateway                                │
│              (Real-time bidirectional)                              │
└───────────────────────────────┬─────────────────────────────────────┘
                                │
        ┌───────────────────────┼───────────────────────┐
        │                       │                       │
        ▼                       ▼                       ▼
┌───────────────┐      ┌───────────────┐      ┌───────────────┐
│ Location      │      │  Matching     │      │   Trip        │
│ Service       │      │  Service      │      │   Service     │
└───────┬───────┘      └───────┬───────┘      └───────────────┘
        │                      │
        ▼                      ▼
┌───────────────┐      ┌───────────────┐
│ Location DB   │      │  Geospatial   │
│ (Time-series) │      │  Index        │
└───────────────┘      └───────────────┘
```

---

## 4. Location Tracking

### Geospatial Indexing (QuadTree / Geohash)

```
┌─────────────────────────────────────────────────────────────────────┐
│                      Geohash Approach                               │
│                                                                     │
│   World divided into grid cells                                    │
│   Each cell has unique string ID                                   │
│                                                                     │
│   Geohash: "9q8yy" (San Francisco)                                 │
│                                                                     │
│   Prefix sharing = proximity                                       │
│   "9q8yy" and "9q8yz" are neighbors                               │
│                                                                     │
│   ┌─────┬─────┬─────┐                                              │
│   │9q8yx│9q8yy│9q8yz│                                              │
│   ├─────┼─────┼─────┤                                              │
│   │9q8yv│9q8yw│9q8yq│   ◀── Geohash cells                         │
│   ├─────┼─────┼─────┤                                              │
│   │9q8yt│9q8yu│9q8yr│                                              │
│   └─────┴─────┴─────┘                                              │
│                                                                     │
│   Find nearby drivers:                                             │
│   1. Get rider's geohash: "9q8yy"                                  │
│   2. Query drivers in "9q8yy" + neighboring cells                 │
│   3. Filter by exact distance                                      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Location Storage (Redis)

```
# Store driver locations in Redis
GEOADD drivers 122.4194 37.7749 driver_123
GEOADD drivers 122.4195 37.7750 driver_456

# Find drivers within 5km radius
GEORADIUS drivers 122.4194 37.7749 5 km WITHDIST

Result:
1) driver_123, 0.00
2) driver_456, 0.15
```

---

## 5. Matching Algorithm

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Ride Matching Flow                              │
│                                                                     │
│   1. Rider requests ride at location L                             │
│                                                                     │
│   2. Find candidate drivers:                                       │
│      - Query geospatial index (Redis GEORADIUS)                   │
│      - Filter: available, right vehicle type                      │
│      - Get top N candidates by distance                           │
│                                                                     │
│   3. Score candidates:                                             │
│      score = f(distance, rating, acceptance_rate, ETA)            │
│                                                                     │
│   4. Send request to best driver                                   │
│      - Driver has 15 seconds to accept                            │
│      - If declined/timeout → next best driver                     │
│                                                                     │
│   5. Match confirmed:                                              │
│      - Update driver status to "on_trip"                          │
│      - Send confirmation to both parties                          │
│      - Start trip tracking                                         │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 6. ETA Calculation

```
Simple: distance / average_speed
Better: Graph-based routing (Dijkstra, A*)
Best: ML model with historical data

Factors:
- Time of day
- Traffic patterns
- Weather
- Events
- Road types

Pre-compute:
- Partition city into cells
- Pre-calculate ETA between cell pairs
- Adjust with real-time traffic data
```

---

## 7. Surge Pricing

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Surge Pricing                                   │
│                                                                     │
│   For each geohash cell, track:                                    │
│   - Open ride requests (demand)                                    │
│   - Available drivers (supply)                                     │
│                                                                     │
│   surge_multiplier = f(demand / supply)                            │
│                                                                     │
│   Example:                                                         │
│   Cell "9q8yy":                                                    │
│   - 50 ride requests                                               │
│   - 10 available drivers                                           │
│   - Ratio: 5:1                                                     │
│   - Surge: 2.5x                                                    │
│                                                                     │
│   Update every 1-2 minutes based on real-time data                │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 8. Data Model

```sql
-- Drivers
CREATE TABLE drivers (
    id              UUID PRIMARY KEY,
    name            VARCHAR(100),
    phone           VARCHAR(20),
    vehicle_type    VARCHAR(20),
    rating          DECIMAL(2,1),
    status          ENUM('available', 'on_trip', 'offline'),
    current_lat     DECIMAL(10, 8),
    current_lng     DECIMAL(11, 8),
    last_updated    TIMESTAMP
);

-- Trips
CREATE TABLE trips (
    id              UUID PRIMARY KEY,
    rider_id        UUID,
    driver_id       UUID,
    status          ENUM('requested', 'matched', 'started', 'completed', 'cancelled'),
    pickup_lat      DECIMAL(10, 8),
    pickup_lng      DECIMAL(11, 8),
    dropoff_lat     DECIMAL(10, 8),
    dropoff_lng     DECIMAL(11, 8),
    fare            DECIMAL(10, 2),
    surge_multiplier DECIMAL(3, 2),
    created_at      TIMESTAMP,
    started_at      TIMESTAMP,
    completed_at    TIMESTAMP
);
```

---

## 9. Interview Tips

**Key Points:**
- Geospatial indexing (Geohash, QuadTree, R-tree)
- Real-time WebSocket communication
- Matching algorithm optimization
- Surge pricing economics
- Handling scale (city partitioning)

**Questions to Ask:**
- Single city or global?
- Vehicle types (cars, bikes, scooters)?
- Real-time vs batched matching?
- Carpooling/shared rides?
