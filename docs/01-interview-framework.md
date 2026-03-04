# System Design Interview Framework

A structured approach for tackling any system design interview question.

---

## Interview Timeline (45 minutes)

```
┌─────────────────────────────────────────────────────────────────────────┐
│  0-5 min    │  5-10 min   │ 10-35 min      │ 35-40 min  │ 40-45 min   │
│  Clarify    │  Estimate   │ Design         │ Deep Dive  │ Wrap Up     │
│  Requirements│  Scale     │ Architecture   │ Trade-offs │ Questions   │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Phase 1: Requirements Clarification (5 minutes)

### Functional Requirements
Ask questions to understand WHAT the system should do:

```
"What are the core features we need to support?"
"Who are the users of this system?"
"What actions can users perform?"
"What are the input/output of the system?"
```

**Example for Twitter:**
- Users can post tweets (max 280 chars)
- Users can follow other users
- Users can view their home timeline
- Users can like and retweet

### Non-Functional Requirements
Ask about quality attributes:

```
"What is the expected scale? (users, requests/sec)"
"What's more important: consistency or availability?"
"What is the acceptable latency?"
"Do we need real-time or near-real-time?"
"What are the durability requirements?"
```

**Common Non-Functional Requirements:**
| Attribute | Question to Ask |
|-----------|-----------------|
| Scale | How many users? DAU? MAU? |
| Availability | What's the target uptime? (99.9%? 99.99%?) |
| Latency | What's acceptable response time? |
| Consistency | Can we tolerate stale data? For how long? |
| Durability | What's the acceptable data loss? |
| Security | Authentication? Authorization? Encryption? |

### Out of Scope
Clarify what you're NOT designing:
```
"Should I focus on X or is Y out of scope?"
"Are we designing the mobile app or just the backend?"
```

---

## Phase 2: Back-of-Envelope Estimation (5 minutes)

### Traffic Estimation
```
Given:
- 500M total users
- 200M DAU (Daily Active Users)
- Each user makes 5 requests/day on average

Calculate:
- Daily requests = 200M × 5 = 1B requests/day
- QPS = 1B / 86400 ≈ 12,000 QPS
- Peak QPS = 12,000 × 3 = 36,000 QPS (assuming 3x peak)
```

### Storage Estimation
```
Given:
- 1M new posts/day
- Each post = 500 bytes (text + metadata)
- Keep data for 5 years

Calculate:
- Daily storage = 1M × 500 bytes = 500 MB/day
- Yearly storage = 500 MB × 365 = 182.5 GB/year
- 5-year storage = 182.5 GB × 5 ≈ 1 TB
- With replication (3x) = 3 TB
```

### Bandwidth Estimation
```
Given:
- 12,000 QPS
- Average response size = 10 KB

Calculate:
- Bandwidth = 12,000 × 10 KB = 120 MB/s
- Daily bandwidth = 120 MB/s × 86400 = 10 TB/day
```

### Memory Estimation (for caching)
```
Rule: Cache 20% of daily traffic (80/20 rule)

Given:
- 1B requests/day
- Average request data = 500 bytes

Calculate:
- Daily data = 1B × 500 bytes = 500 GB
- Cache size = 500 GB × 0.2 = 100 GB
```

---

## Phase 3: High-Level Design (15-20 minutes)

### Step 1: API Design
Define the core APIs first:

```
POST /api/v1/tweets
  Body: { "content": "Hello world", "media_ids": [] }
  Response: { "tweet_id": "123", "created_at": "..." }

GET /api/v1/timeline?user_id=X&page=1&limit=20
  Response: { "tweets": [...], "next_cursor": "..." }

POST /api/v1/follow
  Body: { "followee_id": "456" }
```

### Step 2: Data Model
Define your database schema:

```sql
-- Users table
CREATE TABLE users (
    id          BIGINT PRIMARY KEY,
    username    VARCHAR(50) UNIQUE,
    email       VARCHAR(255) UNIQUE,
    created_at  TIMESTAMP
);

-- Tweets table
CREATE TABLE tweets (
    id          BIGINT PRIMARY KEY,
    user_id     BIGINT REFERENCES users(id),
    content     VARCHAR(280),
    created_at  TIMESTAMP,
    INDEX (user_id, created_at)
);

-- Follows table
CREATE TABLE follows (
    follower_id BIGINT,
    followee_id BIGINT,
    created_at  TIMESTAMP,
    PRIMARY KEY (follower_id, followee_id)
);
```

### Step 3: Component Architecture
Draw the main components:

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Clients    │────▶│    CDN       │────▶│ Load Balancer│
│ (Web/Mobile) │     │              │     │              │
└──────────────┘     └──────────────┘     └──────────────┘
                                                 │
                                                 ▼
                     ┌───────────────────────────────────────┐
                     │            API Gateway                │
                     │  (Auth, Rate Limiting, Routing)       │
                     └───────────────────────────────────────┘
                                         │
            ┌────────────────────────────┼────────────────────────────┐
            ▼                            ▼                            ▼
    ┌──────────────┐            ┌──────────────┐            ┌──────────────┐
    │ User Service │            │ Tweet Service│            │ Feed Service │
    └──────────────┘            └──────────────┘            └──────────────┘
            │                            │                            │
            ▼                            ▼                            ▼
    ┌──────────────┐            ┌──────────────┐            ┌──────────────┐
    │   User DB    │            │  Tweet DB    │            │   Cache      │
    │  (Postgres)  │            │ (Cassandra)  │            │   (Redis)    │
    └──────────────┘            └──────────────┘            └──────────────┘
```

### Step 4: Data Flow
Explain how data flows through the system:

**Write Path (Posting a Tweet):**
```
1. Client sends POST /tweets to API Gateway
2. API Gateway authenticates and routes to Tweet Service
3. Tweet Service validates and stores in Tweet DB
4. Tweet Service publishes event to Message Queue
5. Feed Service consumes event and updates follower timelines
6. Returns success to client
```

**Read Path (Fetching Timeline):**
```
1. Client sends GET /timeline to API Gateway
2. API Gateway routes to Feed Service
3. Feed Service checks Redis cache
4. If cache miss, fetch from DB and populate cache
5. Return timeline to client
```

---

## Phase 4: Detailed Design & Deep Dives (10 minutes)

Focus on 2-3 critical components based on the problem:

### Common Deep Dive Topics

**1. Database Design**
- SQL vs NoSQL decision
- Sharding strategy
- Indexing strategy
- Replication

**2. Caching Strategy**
- What to cache?
- Cache eviction policy (LRU, LFU)
- Cache invalidation
- Cache-aside vs Write-through

**3. Scalability**
- Horizontal vs Vertical scaling
- Database sharding
- Consistent hashing
- Read replicas

**4. Reliability**
- Failure scenarios
- Redundancy
- Health checks
- Circuit breakers

**5. Data Consistency**
- Strong vs Eventual consistency
- Conflict resolution
- Distributed transactions

---

## Phase 5: Trade-offs & Evaluation (5 minutes)

### Discuss Trade-offs Made
```
"We chose eventual consistency over strong consistency because..."
"We selected NoSQL over SQL because..."
"We used push model for small followings but pull for celebrities because..."
```

### Identify Bottlenecks
```
"The main bottleneck is the database writes during peak hours"
"Solution: We can add a write-ahead buffer or queue"
```

### Future Improvements
```
"If we had more time, we could add:"
- ML-based content ranking
- Real-time notifications
- Analytics pipeline
```

---

## Interview Communication Tips

### DO:
- Think out loud
- Ask clarifying questions
- Make reasonable assumptions (state them clearly)
- Discuss trade-offs
- Use numbers and math
- Draw diagrams
- Start simple, then iterate

### DON'T:
- Jump straight into details
- Stay silent while thinking
- Overcomplicate the design
- Ignore the interviewer's hints
- Get stuck on one component
- Forget about scale and reliability

### Useful Phrases:
```
"Let me make sure I understand the requirements..."
"I'm going to make an assumption that... Does that sound reasonable?"
"Let me do a quick calculation..."
"There's a trade-off here between X and Y..."
"One potential bottleneck is... Here's how we could address it..."
"An alternative approach would be..."
```

---

## Quick Reference: Common Questions

| Problem | Key Focus Areas |
|---------|-----------------|
| URL Shortener | Hashing, collision handling, caching |
| Twitter | Fan-out, timeline, read-heavy |
| Instagram | CDN, storage, feed ranking |
| YouTube | Video processing, streaming, CDN |
| WhatsApp | Real-time, WebSocket, presence |
| Uber | Geospatial, matching, real-time |
| Dropbox | File sync, chunking, deduplication |
| Web Crawler | Distributed, politeness, dedup |
| Search Engine | Indexing, ranking, typeahead |
| Rate Limiter | Token bucket, distributed counters |

---

## Checklist Before Interview Ends

- [ ] Covered functional requirements
- [ ] Covered non-functional requirements
- [ ] Did capacity estimation
- [ ] Designed APIs
- [ ] Designed data model
- [ ] Drew high-level architecture
- [ ] Explained data flow
- [ ] Deep-dived into 2-3 components
- [ ] Discussed trade-offs
- [ ] Identified bottlenecks and solutions
