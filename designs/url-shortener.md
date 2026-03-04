# Design a URL Shortener (TinyURL)

A classic system design interview question covering hashing, databases, and caching.

---

## 1. Requirements

### Functional Requirements
- Shorten a long URL to a short URL
- Redirect short URL to original long URL
- Optional: Custom short URLs
- Optional: Analytics (click count, location)
- Optional: URL expiration

### Non-Functional Requirements
- High availability (reads are critical)
- Low latency for redirects (< 100ms)
- Short URLs should be unpredictable
- Scale: 100M URLs created/day, 10:1 read/write ratio

### Out of Scope
- User authentication
- Rate limiting (mention briefly)
- Spam/malware detection

---

## 2. Capacity Estimation

### Traffic
```
Writes (URL creation):
- 100M URLs/day
- QPS = 100M / 86400 ≈ 1,200 QPS
- Peak QPS ≈ 3,500 QPS

Reads (redirects):
- 10:1 ratio → 1B redirects/day
- QPS = 1B / 86400 ≈ 12,000 QPS
- Peak QPS ≈ 36,000 QPS
```

### Storage
```
Per URL record:
- Short URL: 7 chars = 7 bytes
- Long URL: avg 200 bytes
- Created at: 8 bytes
- Expiration: 8 bytes
- User ID: 8 bytes
- Total: ~250 bytes

5-year storage:
- 100M × 365 × 5 = 182.5B URLs
- 182.5B × 250 bytes = 45.6 TB
- With replication (3x): ~140 TB
```

### Bandwidth
```
Write: 1,200 QPS × 250 bytes = 300 KB/s
Read: 12,000 QPS × 250 bytes = 3 MB/s

Negligible bandwidth requirements
```

### Cache
```
80/20 rule: Cache 20% of daily URLs
- Daily URLs: 100M × 250 bytes = 25 GB
- Cache size: 25 GB × 0.2 = 5 GB

Easily fits in single Redis instance
```

---

## 3. API Design

### Create Short URL
```
POST /api/v1/urls
Request:
{
    "long_url": "https://example.com/very/long/path",
    "custom_alias": "my-link",      // optional
    "expiration": "2025-12-31"      // optional
}

Response: 201 Created
{
    "short_url": "https://tinyurl.com/abc123",
    "long_url": "https://example.com/very/long/path",
    "created_at": "2024-01-15T10:30:00Z",
    "expires_at": "2025-12-31T00:00:00Z"
}
```

### Redirect
```
GET /{short_code}

Response: 301 Moved Permanently (or 302 Found)
Location: https://example.com/very/long/path
```

### Get URL Info (Optional)
```
GET /api/v1/urls/{short_code}

Response:
{
    "short_url": "https://tinyurl.com/abc123",
    "long_url": "https://example.com/very/long/path",
    "click_count": 1523,
    "created_at": "2024-01-15T10:30:00Z"
}
```

---

## 4. Database Schema

```sql
CREATE TABLE urls (
    id              BIGINT PRIMARY KEY,
    short_code      VARCHAR(10) UNIQUE NOT NULL,
    long_url        VARCHAR(2048) NOT NULL,
    user_id         BIGINT,
    created_at      TIMESTAMP DEFAULT NOW(),
    expires_at      TIMESTAMP,
    click_count     BIGINT DEFAULT 0,

    INDEX idx_short_code (short_code),
    INDEX idx_user_id (user_id),
    INDEX idx_expires_at (expires_at)
);
```

**Database Choice:** NoSQL (DynamoDB, Cassandra) preferred
- Simple key-value access pattern
- High read throughput
- Easy horizontal scaling
- No complex queries needed

---

## 5. High-Level Design

```
┌─────────────────────────────────────────────────────────────────┐
│                          Clients                                │
└───────────────────────────────┬─────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                         Load Balancer                           │
└───────────────────────────────┬─────────────────────────────────┘
                                │
                ┌───────────────┴───────────────┐
                │                               │
                ▼                               ▼
┌───────────────────────────┐   ┌───────────────────────────────┐
│      URL Service          │   │         Analytics Service      │
│  (Create, Redirect)       │   │    (Async click tracking)     │
└─────────────┬─────────────┘   └───────────────────────────────┘
              │
    ┌─────────┴─────────┐
    │                   │
    ▼                   ▼
┌─────────┐       ┌─────────────────┐
│  Cache  │       │     Database    │
│ (Redis) │       │(DynamoDB/Cassandra)│
└─────────┘       └─────────────────┘
    │                      │
    │                      ▼
    │             ┌─────────────────┐
    │             │  Key Generation │
    │             │    Service      │
    └─────────────┴─────────────────┘
```

---

## 6. Short URL Generation

### Option 1: Hash + Truncate

```
hash = MD5(long_url)           # 128-bit hash
short_code = base62(hash[:43]) # Take first 43 bits → 7 chars

Base62 alphabet: [a-z, A-Z, 0-9]
7 characters = 62^7 = 3.5 trillion combinations
```

**Problem:** Collisions
**Solution:** Check DB, append counter if collision

### Option 2: Counter + Base62

```
counter = getNextId()          # 1, 2, 3, ...
short_code = base62(counter)   # Sequential

Example:
1 → "1"
62 → "10"
1000000 → "4c92"
```

**Problem:** Predictable, sequential
**Solution:** Add randomness or use distributed ID generator

### Option 3: Key Generation Service (KGS) - Recommended

```
┌─────────────────────────────────────────────────────────────────┐
│                    Key Generation Service                       │
│                                                                 │
│   Pre-generate keys and store in database                      │
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  Used Keys DB          │  Available Keys DB             │  │
│   │  [abc123, def456, ...] │  [ghi789, jkl012, mno345, ...] │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                 │
│   URL Service requests batch of keys                           │
│   KGS moves keys from Available → Used                         │
│   Pre-generation handles burst traffic                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Benefits:**
- No collision handling needed
- Fast (pre-computed)
- Unpredictable
- Easy to scale

**Implementation:**
```python
class KeyGenerationService:
    def __init__(self):
        self.key_pool = []  # Local cache
        self.pool_size = 1000

    def get_key(self):
        if len(self.key_pool) == 0:
            self.key_pool = self.fetch_keys_from_db(self.pool_size)
        return self.key_pool.pop()

    def generate_keys(self, count):
        # Pre-generate random 7-char base62 keys
        keys = [generate_random_base62(7) for _ in range(count)]
        # Filter out any that already exist
        keys = [k for k in keys if not self.key_exists(k)]
        self.store_keys(keys)
```

---

## 7. Detailed Component Design

### Write Path (Create Short URL)

```
1. Client sends POST /api/v1/urls with long_url
2. Validate long_url (format, not blocked)
3. Check if long_url already shortened (optional dedup)
4. Get next available key from KGS
5. Create record in database
6. Invalidate cache if exists
7. Return short URL to client

┌────────┐    ┌──────────┐    ┌─────┐    ┌────────┐
│ Client │───▶│   API    │───▶│ KGS │───▶│   DB   │
└────────┘    └──────────┘    └─────┘    └────────┘
                   │                          │
                   │     ┌─────────┐          │
                   └────▶│  Cache  │◀─────────┘
                         └─────────┘
```

### Read Path (Redirect)

```
1. Client requests GET /{short_code}
2. Check cache for short_code
3. If cache hit: redirect immediately
4. If cache miss: query database
5. If found: add to cache, redirect
6. If not found: return 404
7. Async: increment click counter

┌────────┐    ┌──────────┐    ┌─────────┐    ┌────────┐
│ Client │───▶│   API    │───▶│  Cache  │───▶│   DB   │
└────────┘    │          │    └─────────┘    └────────┘
     ▲        │          │          │             │
     │        │  301     │          │ Cache miss  │
     └────────│ Redirect │◀─────────┴─────────────┘
              └──────────┘
```

### Cache Strategy

```
Cache-Aside Pattern:
1. Read: Check cache → if miss, read DB → populate cache
2. Write: Write to DB → invalidate cache

TTL: 24 hours (frequently accessed URLs stay cached)

Eviction: LRU (Least Recently Used)
```

---

## 8. Scaling Considerations

### Database Sharding

```
Shard by short_code (hash-based):
shard_id = hash(short_code) % num_shards

┌─────────────────────────────────────────────────────────────────┐
│                         Shard Router                            │
└──────────────────────────────┬──────────────────────────────────┘
                               │
         ┌─────────────────────┼─────────────────────┐
         │                     │                     │
         ▼                     ▼                     ▼
    ┌─────────┐           ┌─────────┐          ┌─────────┐
    │ Shard 0 │           │ Shard 1 │          │ Shard 2 │
    │ a-h     │           │ i-p     │          │ q-z     │
    └─────────┘           └─────────┘          └─────────┘
```

### Cache Cluster

```
Consistent hashing for cache distribution:
- Add/remove nodes without full redistribution
- Redis Cluster or Memcached with consistent hashing

cache_node = consistent_hash(short_code)
```

### KGS High Availability

```
┌─────────────────────────────────────────────────────────────────┐
│                    KGS Cluster                                  │
│                                                                 │
│   ┌────────────┐  ┌────────────┐  ┌────────────┐              │
│   │   KGS-1    │  │   KGS-2    │  │   KGS-3    │              │
│   │ Keys: 1-33%│  │Keys: 34-66%│  │Keys: 67-100%│             │
│   └────────────┘  └────────────┘  └────────────┘              │
│                                                                 │
│   Each KGS owns a range of keys                                │
│   If one fails, others continue serving                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 9. Additional Features

### URL Expiration

```sql
-- Cleanup job (runs periodically)
DELETE FROM urls WHERE expires_at < NOW();

-- Or: Lazy expiration
-- Check expiration on read, return 410 Gone if expired
```

### Analytics

```
┌──────────────┐     ┌─────────────────┐     ┌───────────────┐
│ Click Event  │────▶│  Message Queue  │────▶│ Analytics     │
│              │     │  (Kafka)        │     │ Processor     │
└──────────────┘     └─────────────────┘     └───────┬───────┘
                                                     │
                                                     ▼
                                             ┌───────────────┐
                                             │ Analytics DB  │
                                             │ (ClickHouse)  │
                                             └───────────────┘

Track:
- Timestamp
- Short code
- User agent
- Referrer
- IP (for geolocation)
```

### Custom Aliases

```python
def create_short_url(long_url, custom_alias=None):
    if custom_alias:
        # Check if alias is available
        if exists(custom_alias):
            raise AliasAlreadyExists()
        short_code = custom_alias
    else:
        short_code = kgs.get_key()

    # Store and return
    ...
```

---

## 10. Trade-offs & Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Database | NoSQL (Cassandra/DynamoDB) | Simple KV access, high throughput |
| Key Generation | Pre-generated (KGS) | No collisions, fast, unpredictable |
| Short URL length | 7 characters | 3.5T combinations, manageable length |
| Redirect code | 301 (permanent) | SEO friendly, reduces server load |
| Cache | Redis with LRU | Fast, simple, reliable |

### 301 vs 302 Redirect
- **301 Permanent:** Browser caches, reduces server load, better SEO
- **302 Temporary:** Every request hits server, better for analytics

**Recommendation:** Use 301 for regular URLs, 302 if real-time analytics needed

---

## 11. Potential Bottlenecks

| Bottleneck | Solution |
|------------|----------|
| KGS single point of failure | Replicate KGS, pre-fetch keys |
| Database write throughput | Sharding, async writes |
| Hot URLs (viral) | CDN caching, edge redirects |
| Key exhaustion | Monitor usage, 7 chars = 3.5T combinations |

---

## 12. Interview Tips

**Questions to ask:**
- What's the expected scale?
- Do we need analytics?
- Should URLs expire?
- Do we need custom aliases?

**Key talking points:**
- Base62 encoding and why 7 characters
- KGS for collision-free key generation
- Caching strategy for read-heavy workload
- 301 vs 302 trade-offs
- Sharding strategy for database
