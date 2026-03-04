# Design a Notification System

Multi-channel notification delivery (push, email, SMS).

---

## 1. Requirements

### Functional
- Send notifications via multiple channels (push, email, SMS)
- User preferences (opt-in/opt-out per channel)
- Template management
- Rate limiting
- Scheduling

### Non-Functional
- High throughput (millions/day)
- Reliable delivery
- Low latency for real-time notifications
- Analytics (delivery, open rates)

---

## 2. High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│   ┌─────────────┐                                                  │
│   │   Clients   │─────────────────────────────────────────────────▶│
│   │  (Services) │                                                  │
│   └─────────────┘                                                  │
│          │                                                          │
│          ▼                                                          │
│   ┌─────────────────────────────────────────────────────────────┐  │
│   │                  Notification Service                        │  │
│   │  (Validation, Preferences, Rate Limiting)                   │  │
│   └──────────────────────────┬──────────────────────────────────┘  │
│                              │                                      │
│                              ▼                                      │
│   ┌─────────────────────────────────────────────────────────────┐  │
│   │                     Message Queue (Kafka)                    │  │
│   └──────────────────────────┬──────────────────────────────────┘  │
│                              │                                      │
│         ┌────────────────────┼────────────────────┐                │
│         │                    │                    │                │
│         ▼                    ▼                    ▼                │
│   ┌───────────┐       ┌───────────┐       ┌───────────┐          │
│   │   Push    │       │   Email   │       │    SMS    │          │
│   │  Worker   │       │  Worker   │       │  Worker   │          │
│   └─────┬─────┘       └─────┬─────┘       └─────┬─────┘          │
│         │                   │                   │                 │
│         ▼                   ▼                   ▼                 │
│   ┌───────────┐       ┌───────────┐       ┌───────────┐          │
│   │APNS / FCM │       │SendGrid/  │       │Twilio/    │          │
│   │           │       │Mailgun    │       │Vonage     │          │
│   └───────────┘       └───────────┘       └───────────┘          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 3. Notification Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Notification Flow                               │
│                                                                     │
│   1. Service sends notification request:                           │
│      POST /notifications                                           │
│      {                                                             │
│        "user_id": "123",                                           │
│        "type": "order_shipped",                                    │
│        "data": { "order_id": "456", "tracking": "..." },          │
│        "channels": ["push", "email"]                              │
│      }                                                             │
│                                                                     │
│   2. Notification Service:                                         │
│      a. Validate request                                           │
│      b. Fetch user preferences                                     │
│      c. Check rate limits                                          │
│      d. Render templates                                           │
│      e. Enqueue to channel-specific queues                        │
│                                                                     │
│   3. Workers:                                                      │
│      a. Dequeue message                                            │
│      b. Call external provider (APNS, SendGrid, Twilio)           │
│      c. Handle retries on failure                                  │
│      d. Update delivery status                                     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 4. User Preferences

```sql
CREATE TABLE user_notification_preferences (
    user_id         UUID,
    notification_type VARCHAR(50),  -- order_shipped, promo, etc.
    channel         VARCHAR(20),    -- push, email, sms
    enabled         BOOLEAN,
    PRIMARY KEY (user_id, notification_type, channel)
);

-- Device tokens for push
CREATE TABLE user_devices (
    user_id         UUID,
    device_id       VARCHAR(100),
    platform        VARCHAR(20),    -- ios, android, web
    push_token      VARCHAR(500),
    last_active     TIMESTAMP,
    PRIMARY KEY (user_id, device_id)
);
```

---

## 5. Rate Limiting

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Rate Limiting Strategies                        │
│                                                                     │
│   Per User:                                                        │
│   - Max 10 push notifications per hour                            │
│   - Max 5 emails per day (marketing)                              │
│   - Max 3 SMS per day                                             │
│                                                                     │
│   Per Type:                                                        │
│   - Critical (security): No limit                                 │
│   - Transactional: Higher limit                                   │
│   - Marketing: Lower limit                                        │
│                                                                     │
│   Implementation (Redis):                                          │
│   key = "ratelimit:{user_id}:{channel}:{window}"                  │
│   INCR key                                                         │
│   EXPIRE key {window_seconds}                                      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 6. Reliability

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Reliability Patterns                            │
│                                                                     │
│   1. At-least-once delivery:                                       │
│      - Kafka consumer commits offset after processing             │
│      - Retry on failure with exponential backoff                  │
│                                                                     │
│   2. Idempotency:                                                  │
│      - Unique notification_id                                      │
│      - Dedup before sending                                        │
│                                                                     │
│   3. Fallback channels:                                            │
│      - Push failed? → Try email                                   │
│      - Email bounced? → Try SMS (critical only)                   │
│                                                                     │
│   4. Dead letter queue:                                            │
│      - Failed after N retries → DLQ                               │
│      - Manual review or alternative handling                      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 7. Analytics

```
Track:
- Sent count
- Delivered count
- Failed count
- Opened/clicked (email)
- Dismissed (push)

Store in time-series DB (InfluxDB) or analytics warehouse (BigQuery)
```

---

## 8. Interview Tips

**Key Points:**
- Multi-channel architecture
- User preferences and opt-out
- Rate limiting per user/channel
- Retry and fallback strategies
- Push notification specifics (APNS, FCM)

**Questions to Ask:**
- Which channels?
- Real-time vs batched?
- User preference granularity?
- Global or regional?
