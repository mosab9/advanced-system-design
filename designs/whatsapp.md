# Design WhatsApp (Messaging System)

Real-time messaging platform with presence and delivery tracking.

---

## 1. Requirements

### Functional
- 1:1 messaging
- Group messaging (up to 256 members)
- Message delivery status (sent, delivered, read)
- Online/offline presence
- Media sharing (images, videos)

### Non-Functional
- Real-time delivery (< 100ms)
- Message ordering
- At-least-once delivery
- End-to-end encryption
- Offline message queue

---

## 2. High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Mobile Clients                              │
└───────────────────────────────┬─────────────────────────────────────┘
                                │ WebSocket
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    WebSocket Gateway Cluster                        │
│              (Maintains persistent connections)                     │
└───────────────────────────────┬─────────────────────────────────────┘
                                │
        ┌───────────────────────┼───────────────────────┐
        │                       │                       │
        ▼                       ▼                       ▼
┌───────────────┐      ┌───────────────┐      ┌───────────────┐
│   Message     │      │   Presence    │      │   Group       │
│   Service     │      │   Service     │      │   Service     │
└───────┬───────┘      └───────────────┘      └───────────────┘
        │
        ▼
┌───────────────────────────────────────────────────────────────────┐
│                       Message Queue (Kafka)                       │
└───────────────────────────────────────────────────────────────────┘
        │
        ▼
┌───────────────┐      ┌───────────────┐
│  Message DB   │      │  Push         │
│  (Cassandra)  │      │  Notification │
└───────────────┘      └───────────────┘
```

---

## 3. Message Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Message Delivery Flow                           │
│                                                                     │
│   User A (Online) sends message to User B                          │
│                                                                     │
│   Case 1: User B Online                                            │
│   ┌─────┐      ┌─────────┐      ┌─────────┐      ┌─────┐          │
│   │  A  │─────▶│ Gateway │─────▶│ Gateway │─────▶│  B  │          │
│   └─────┘      │  (A)    │      │  (B)    │      └─────┘          │
│                └─────────┘      └─────────┘                        │
│                     │                ▲                              │
│                     ▼                │                              │
│               ┌──────────┐           │                              │
│               │ Message  │───────────┘                              │
│               │ Service  │                                          │
│               └──────────┘                                          │
│                                                                     │
│   Case 2: User B Offline                                           │
│   ┌─────┐      ┌─────────┐      ┌──────────┐                       │
│   │  A  │─────▶│ Gateway │─────▶│ Message  │                       │
│   └─────┘      │  (A)    │      │ DB       │                       │
│                └─────────┘      └────┬─────┘                       │
│                                      │                              │
│                                      ▼                              │
│                               ┌──────────────┐      ┌─────┐        │
│                               │ Push Service │─────▶│  B  │        │
│                               └──────────────┘      └─────┘        │
│                                                         │          │
│   When B comes online:                                  │          │
│   B connects ──▶ Fetch undelivered messages ──▶ Mark delivered    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 4. Connection Management

```
┌─────────────────────────────────────────────────────────────────────┐
│                  WebSocket Gateway Design                           │
│                                                                     │
│   User-to-Server mapping:                                          │
│   ┌──────────────────────────────────────────────────────────┐     │
│   │  Redis (Connection Registry)                             │     │
│   │  user_123 → gateway_server_5                            │     │
│   │  user_456 → gateway_server_2                            │     │
│   │  user_789 → gateway_server_5                            │     │
│   └──────────────────────────────────────────────────────────┘     │
│                                                                     │
│   Gateway Server:                                                  │
│   - Maintains WebSocket connections (100K-500K per server)        │
│   - Heartbeat for connection health                               │
│   - Publishes user online/offline events                          │
│                                                                     │
│   To send message to user_789:                                     │
│   1. Lookup user_789 → gateway_server_5                           │
│   2. Send to gateway_server_5 via internal pub/sub               │
│   3. gateway_server_5 pushes to WebSocket                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 5. Message Delivery Status

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Delivery Receipt Flow                            │
│                                                                     │
│   Status: ✓ (Sent) → ✓✓ (Delivered) → ✓✓ (Read/Blue)              │
│                                                                     │
│   1. SENT: Message reached server                                  │
│      Server → Sender: "message_id: sent"                          │
│                                                                     │
│   2. DELIVERED: Message reached recipient's device                 │
│      Recipient device → Server → Sender: "message_id: delivered"  │
│                                                                     │
│   3. READ: Recipient opened conversation                           │
│      Recipient device → Server → Sender: "message_id: read"       │
│                                                                     │
│   Batch receipts to reduce traffic:                               │
│   "messages [m1, m2, m3] delivered at timestamp T"                │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 6. Group Messaging

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Group Message Delivery                          │
│                                                                     │
│   Group: 100 members                                               │
│   User A sends message to group                                    │
│                                                                     │
│   Approach 1: Fan-out on Write                                     │
│   - Create 99 message copies (one per recipient)                  │
│   - High write amplification                                       │
│   - Fast reads                                                     │
│                                                                     │
│   Approach 2: Shared Message (WhatsApp approach)                  │
│   - Store message once with group_id                              │
│   - Each user has pointer to last read message                    │
│   - Track delivery/read per user separately                       │
│                                                                     │
│   Message: { id, group_id, sender, content, timestamp }           │
│   Delivery: { message_id, user_id, status, timestamp }            │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 7. Presence (Online/Offline)

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Presence System                                 │
│                                                                     │
│   Challenge: Broadcasting presence to all contacts is expensive    │
│                                                                     │
│   Solution: Subscribe on demand                                    │
│                                                                     │
│   User A opens chat with B:                                        │
│   1. Subscribe to B's presence                                     │
│   2. Server sends B's current status                              │
│   3. When B's status changes, push update to A                    │
│                                                                     │
│   User A closes chat with B:                                       │
│   1. Unsubscribe from B's presence                                │
│                                                                     │
│   Redis for presence:                                              │
│   SETEX presence:user_123 30 "online"  (30 sec TTL)               │
│   Heartbeat every 20 seconds to refresh                           │
│   If TTL expires → user is offline                                 │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 8. Data Model

```sql
-- Messages
CREATE TABLE messages (
    message_id      UUID,
    conversation_id UUID,      -- Can be 1:1 chat or group
    sender_id       UUID,
    content         BLOB,      -- Encrypted
    content_type    VARCHAR(20),
    created_at      TIMESTAMP,
    PRIMARY KEY (conversation_id, created_at, message_id)
) WITH CLUSTERING ORDER BY (created_at DESC);

-- Delivery Status (per user per message)
CREATE TABLE message_status (
    user_id         UUID,
    message_id      UUID,
    status          VARCHAR(20),  -- sent, delivered, read
    updated_at      TIMESTAMP,
    PRIMARY KEY (user_id, message_id)
);

-- Undelivered messages queue
CREATE TABLE undelivered_messages (
    user_id         UUID,
    message_id      UUID,
    conversation_id UUID,
    created_at      TIMESTAMP,
    PRIMARY KEY (user_id, created_at, message_id)
) WITH CLUSTERING ORDER BY (created_at ASC);
```

---

## 9. Interview Tips

**Key Points:**
- WebSocket for real-time bidirectional communication
- Connection registry for user-to-server mapping
- Message acknowledgment and delivery receipts
- Handling offline users (message queue + push)
- Presence subscription model
- End-to-end encryption (Signal Protocol)

**Questions to Ask:**
- Maximum group size?
- Message retention policy?
- Media file size limits?
- Multi-device support?
