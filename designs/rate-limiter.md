# Design a Rate Limiter

Essential component for API protection, commonly asked in system design interviews.

---

## 1. Requirements

### Functional Requirements
- Limit requests per user/IP/API key
- Support different rate limits for different APIs
- Return appropriate error when limit exceeded
- Support various time windows (per second, minute, hour, day)

### Non-Functional Requirements
- Low latency (< 1ms overhead)
- High availability
- Distributed (works across multiple servers)
- Accurate (minimize false positives/negatives)
- Memory efficient

### Out of Scope
- Authentication/Authorization
- Request routing
- Detailed analytics

---

## 2. Rate Limiting Algorithms

### Algorithm 1: Token Bucket

```
┌─────────────────────────────────────────────────────────────────┐
│                        Token Bucket                             │
│                                                                 │
│   Bucket Capacity: 10 tokens                                   │
│   Refill Rate: 1 token/second                                  │
│                                                                 │
│   ┌─────────────────────────────────────────────┐              │
│   │ ● ● ● ● ● ● ○ ○ ○ ○                        │              │
│   │ 6 tokens available                          │              │
│   └─────────────────────────────────────────────┘              │
│                  ▲                                              │
│                  │ Refill 1/sec                                │
│                  │                                              │
│   Request arrives:                                              │
│   - Token available? → Consume token, allow request            │
│   - No token? → Reject (429 Too Many Requests)                 │
│                                                                 │
│   Allows bursts up to bucket capacity                          │
└─────────────────────────────────────────────────────────────────┘
```

**Pros:**
- Allows controlled bursts
- Memory efficient (2 values per user)
- Simple implementation

**Cons:**
- Challenging to tune bucket size and refill rate

```python
class TokenBucket:
    def __init__(self, capacity, refill_rate):
        self.capacity = capacity
        self.refill_rate = refill_rate  # tokens per second
        self.tokens = capacity
        self.last_refill = time.time()

    def allow_request(self, tokens=1):
        self._refill()
        if self.tokens >= tokens:
            self.tokens -= tokens
            return True
        return False

    def _refill(self):
        now = time.time()
        elapsed = now - self.last_refill
        new_tokens = elapsed * self.refill_rate
        self.tokens = min(self.capacity, self.tokens + new_tokens)
        self.last_refill = now
```

### Algorithm 2: Leaky Bucket

```
┌─────────────────────────────────────────────────────────────────┐
│                        Leaky Bucket                             │
│                                                                 │
│   Bucket (Queue) Capacity: 10 requests                         │
│   Outflow Rate: 2 requests/second                              │
│                                                                 │
│            Requests In                                          │
│                 ▼                                               │
│         ┌─────────────┐                                        │
│         │ [R1][R2][R3]│ ◀─── Queue (FIFO)                      │
│         │ [R4][R5][R6]│                                        │
│         └──────┬──────┘                                        │
│                │                                                │
│                ▼ Process at fixed rate (2/sec)                 │
│                                                                 │
│   If bucket full → Reject new requests                         │
│                                                                 │
│   Smooth, consistent output rate                               │
└─────────────────────────────────────────────────────────────────┘
```

**Pros:**
- Smooth output rate
- Good for processing pipelines

**Cons:**
- Bursts fill queue, new requests wait
- More memory (stores queued requests)

### Algorithm 3: Fixed Window Counter

```
┌─────────────────────────────────────────────────────────────────┐
│                    Fixed Window Counter                         │
│                                                                 │
│   Limit: 100 requests per minute                               │
│                                                                 │
│   Time     │ 12:00    │ 12:01    │ 12:02    │                  │
│   Window   │──────────│──────────│──────────│                  │
│   Counter  │    45    │    78    │    23    │                  │
│            │          │          │          │                  │
│                                                                 │
│   New request at 12:01:                                        │
│   - Counter = 78 < 100? → Allow, increment to 79              │
│   - Counter >= 100? → Reject                                   │
│                                                                 │
│   Counter resets at window boundary                            │
└─────────────────────────────────────────────────────────────────┘
```

**Problem: Boundary Burst**
```
Window 1 (12:00-12:01): 100 requests at 12:00:59
Window 2 (12:01-12:02): 100 requests at 12:01:00
Total: 200 requests in 2 seconds!
```

**Pros:**
- Simple, memory efficient
- Easy to implement

**Cons:**
- Burst at window boundaries

### Algorithm 4: Sliding Window Log

```
┌─────────────────────────────────────────────────────────────────┐
│                    Sliding Window Log                           │
│                                                                 │
│   Limit: 5 requests per minute                                 │
│   Current time: 12:01:30                                       │
│                                                                 │
│   Request Log (timestamps):                                    │
│   [12:00:45, 12:01:00, 12:01:15, 12:01:20, 12:01:25]          │
│                                                                 │
│   Sliding window: 12:00:30 to 12:01:30                         │
│                                                                 │
│   Requests in window:                                          │
│   - 12:00:45 ✗ (outside window)                               │
│   - 12:01:00 ✓                                                 │
│   - 12:01:15 ✓                                                 │
│   - 12:01:20 ✓                                                 │
│   - 12:01:25 ✓                                                 │
│                                                                 │
│   Count = 4 < 5 → Allow new request                           │
└─────────────────────────────────────────────────────────────────┘
```

**Pros:**
- Accurate, no boundary issues

**Cons:**
- Memory intensive (stores all timestamps)
- O(n) to count requests

### Algorithm 5: Sliding Window Counter (Recommended)

```
┌─────────────────────────────────────────────────────────────────┐
│                  Sliding Window Counter                         │
│                                                                 │
│   Combines fixed window efficiency with sliding accuracy        │
│                                                                 │
│   Limit: 100 requests per minute                               │
│   Current time: 12:01:15 (25% into window)                     │
│                                                                 │
│   Previous window (12:00-12:01): 80 requests                   │
│   Current window (12:01-12:02): 30 requests                    │
│                                                                 │
│   Weighted count = (prev × overlap%) + current                 │
│                  = (80 × 0.75) + 30                            │
│                  = 60 + 30 = 90                                │
│                                                                 │
│   90 < 100 → Allow request                                     │
│                                                                 │
│   ◀───── Prev Window ─────▶◀──── Current Window ────▶         │
│   │                        │■■■■■│                   │         │
│   12:00                  12:01  ▲                  12:02       │
│                                 │                              │
│                           Current time                         │
│                           (12:01:15)                           │
└─────────────────────────────────────────────────────────────────┘
```

**Pros:**
- Memory efficient (2 counters per user)
- Smooths boundary issues
- Fast O(1) operations

**Cons:**
- Approximation (not 100% accurate)

---

## 3. High-Level Design

### Single Server Rate Limiter

```
┌────────────────────────────────────────────────────────────────┐
│                         Server                                  │
│                                                                │
│   ┌────────────────────────────────────────────────────────┐  │
│   │                   Rate Limiter                          │  │
│   │  ┌─────────────────────────────────────────────────┐   │  │
│   │  │          In-Memory Hash Map                     │   │  │
│   │  │  user_1: {tokens: 8, last_refill: 12:01:00}    │   │  │
│   │  │  user_2: {tokens: 2, last_refill: 12:01:05}    │   │  │
│   │  │  user_3: {tokens: 10, last_refill: 12:00:55}   │   │  │
│   │  └─────────────────────────────────────────────────┘   │  │
│   └────────────────────────────────────────────────────────┘  │
│                              │                                 │
│                              ▼                                 │
│   ┌────────────────────────────────────────────────────────┐  │
│   │                  Application Logic                      │  │
│   └────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────┘
```

### Distributed Rate Limiter

```
┌─────────────────────────────────────────────────────────────────┐
│                          Clients                                │
└───────────────────────────────┬─────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                       Load Balancer                             │
└───────────────────────────────┬─────────────────────────────────┘
                                │
         ┌──────────────────────┼──────────────────────┐
         │                      │                      │
         ▼                      ▼                      ▼
┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐
│    Server 1     │   │    Server 2     │   │    Server 3     │
│  ┌───────────┐  │   │  ┌───────────┐  │   │  ┌───────────┐  │
│  │Rate Limit │  │   │  │Rate Limit │  │   │  │Rate Limit │  │
│  │Middleware │  │   │  │Middleware │  │   │  │Middleware │  │
│  └─────┬─────┘  │   │  └─────┬─────┘  │   │  └─────┬─────┘  │
└────────┼────────┘   └────────┼────────┘   └────────┼────────┘
         │                     │                     │
         └─────────────────────┼─────────────────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │    Redis Cluster    │
                    │  (Centralized State)│
                    └─────────────────────┘
```

---

## 4. Detailed Design

### Rate Limiter Middleware

```python
class RateLimiterMiddleware:
    def __init__(self, redis_client, rules):
        self.redis = redis_client
        self.rules = rules  # Rate limit rules per endpoint

    def process_request(self, request):
        # Identify the client
        client_id = self._get_client_id(request)

        # Get applicable rule
        rule = self._get_rule(request.endpoint)

        # Check rate limit
        allowed, remaining, reset_time = self._check_limit(
            client_id, rule
        )

        if not allowed:
            return Response(
                status=429,
                headers={
                    'X-RateLimit-Limit': rule.limit,
                    'X-RateLimit-Remaining': 0,
                    'X-RateLimit-Reset': reset_time,
                    'Retry-After': reset_time - time.time()
                },
                body={'error': 'Rate limit exceeded'}
            )

        # Add rate limit headers to response
        request.rate_limit_headers = {
            'X-RateLimit-Limit': rule.limit,
            'X-RateLimit-Remaining': remaining,
            'X-RateLimit-Reset': reset_time
        }

        return None  # Continue to application

    def _get_client_id(self, request):
        # Priority: API key > User ID > IP address
        if request.api_key:
            return f"api:{request.api_key}"
        if request.user_id:
            return f"user:{request.user_id}"
        return f"ip:{request.client_ip}"
```

### Redis Implementation (Sliding Window Counter)

```python
def check_rate_limit(redis, client_id, limit, window_seconds):
    """
    Sliding window counter using Redis
    """
    current_time = int(time.time())
    current_window = current_time // window_seconds
    previous_window = current_window - 1

    current_key = f"rate:{client_id}:{current_window}"
    previous_key = f"rate:{client_id}:{previous_window}"

    # Lua script for atomic operation
    lua_script = """
    local current = tonumber(redis.call('GET', KEYS[1]) or '0')
    local previous = tonumber(redis.call('GET', KEYS[2]) or '0')
    local window_position = tonumber(ARGV[1])
    local limit = tonumber(ARGV[2])
    local window_seconds = tonumber(ARGV[3])

    -- Weighted count
    local weighted_count = (previous * (1 - window_position)) + current

    if weighted_count >= limit then
        return {0, 0}  -- Not allowed
    end

    -- Increment current window
    redis.call('INCR', KEYS[1])
    redis.call('EXPIRE', KEYS[1], window_seconds * 2)

    local remaining = limit - weighted_count - 1
    return {1, remaining}  -- Allowed
    """

    window_position = (current_time % window_seconds) / window_seconds

    result = redis.eval(
        lua_script,
        2,  # Number of keys
        current_key,
        previous_key,
        window_position,
        limit,
        window_seconds
    )

    allowed = result[0] == 1
    remaining = max(0, int(result[1]))
    reset_time = (current_window + 1) * window_seconds

    return allowed, remaining, reset_time
```

### Token Bucket with Redis

```python
def token_bucket_check(redis, client_id, capacity, refill_rate):
    """
    Token bucket implementation using Redis
    """
    key = f"bucket:{client_id}"

    lua_script = """
    local capacity = tonumber(ARGV[1])
    local refill_rate = tonumber(ARGV[2])
    local now = tonumber(ARGV[3])
    local requested = tonumber(ARGV[4])

    local data = redis.call('HMGET', KEYS[1], 'tokens', 'last_refill')
    local tokens = tonumber(data[1]) or capacity
    local last_refill = tonumber(data[2]) or now

    -- Refill tokens
    local elapsed = now - last_refill
    tokens = math.min(capacity, tokens + (elapsed * refill_rate))

    if tokens < requested then
        return {0, tokens}  -- Not allowed
    end

    -- Consume tokens
    tokens = tokens - requested
    redis.call('HMSET', KEYS[1], 'tokens', tokens, 'last_refill', now)
    redis.call('EXPIRE', KEYS[1], capacity / refill_rate * 2)

    return {1, tokens}  -- Allowed
    """

    result = redis.eval(
        lua_script,
        1,
        key,
        capacity,
        refill_rate,
        time.time(),
        1  # tokens requested
    )

    return result[0] == 1, int(result[1])
```

---

## 5. Rate Limiting Rules

### Rule Configuration

```yaml
rate_limits:
  default:
    requests_per_minute: 60
    burst: 10

  endpoints:
    "/api/v1/search":
      requests_per_minute: 30
      burst: 5

    "/api/v1/upload":
      requests_per_minute: 10
      burst: 2

  tiers:
    free:
      requests_per_day: 1000
    premium:
      requests_per_day: 100000
    enterprise:
      requests_per_day: unlimited
```

### Multi-Level Rate Limiting

```
┌─────────────────────────────────────────────────────────────────┐
│                  Multi-Level Rate Limiting                      │
│                                                                 │
│   Level 1: Global (protect infrastructure)                     │
│   ├── 1M requests/minute across all users                      │
│                                                                 │
│   Level 2: Per User (fair usage)                               │
│   ├── 1000 requests/minute per user                            │
│                                                                 │
│   Level 3: Per Endpoint (protect specific resources)           │
│   ├── /search: 30/minute                                       │
│   ├── /upload: 10/minute                                       │
│   └── /api: 100/minute                                         │
│                                                                 │
│   Request must pass ALL levels to proceed                      │
└─────────────────────────────────────────────────────────────────┘
```

---

## 6. Handling Distributed Systems

### Approach 1: Sticky Sessions

```
┌─────────────────────────────────────────────────────────────────┐
│                     Load Balancer                               │
│              (Route by client IP hash)                          │
└───────────────────────────────────────────────────────────────┬─┘
                                                                │
    User A always routes to Server 1 ──────────────────────────▶│
    User B always routes to Server 2 ──────────────────────────▶│
                                                                │
         ┌──────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐
│    Server 1     │   │    Server 2     │   │    Server 3     │
│ (Local counters)│   │ (Local counters)│   │ (Local counters)│
└─────────────────┘   └─────────────────┘   └─────────────────┘

Pros: Simple, no shared state
Cons: Uneven distribution, server failure issues
```

### Approach 2: Centralized Store (Recommended)

```
┌─────────────────────────────────────────────────────────────────┐
│                      All Servers                                │
└───────────────────────────────────┬─────────────────────────────┘
                                    │
                                    ▼
                         ┌─────────────────────┐
                         │    Redis Cluster    │
                         │                     │
                         │  ┌───────────────┐  │
                         │  │ user_1: 45    │  │
                         │  │ user_2: 78    │  │
                         │  │ user_3: 12    │  │
                         │  └───────────────┘  │
                         │                     │
                         └─────────────────────┘

Pros: Accurate, consistent
Cons: Redis latency, SPOF (mitigate with cluster)
```

### Approach 3: Local + Sync

```
┌─────────────────────────────────────────────────────────────────┐
│                        Server 1                                 │
│   ┌───────────────────────────────────────────────────────┐    │
│   │  Local Cache (fast path)                              │    │
│   │  user_1: {count: 45, synced_at: 12:01:00}            │    │
│   └───────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
                                │
                    Periodic sync (every 1s)
                                │
                                ▼
                    ┌─────────────────────┐
                    │   Redis (source     │
                    │   of truth)         │
                    └─────────────────────┘

Pros: Fast local checks, eventually consistent
Cons: Slight inaccuracy during sync window
```

---

## 7. Response Headers

```
HTTP/1.1 200 OK
X-RateLimit-Limit: 100          # Max requests allowed
X-RateLimit-Remaining: 45       # Requests remaining
X-RateLimit-Reset: 1609459200   # Unix timestamp when limit resets

HTTP/1.1 429 Too Many Requests
Retry-After: 30                 # Seconds to wait
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1609459200
```

---

## 8. Race Conditions

### Problem: Read-Modify-Write

```
Server 1: GET count → 99
Server 2: GET count → 99
Server 1: SET count = 100 (allow request)
Server 2: SET count = 100 (allow request - WRONG!)

Both requests allowed when only one should be!
```

### Solution: Atomic Operations

```python
# Use Redis INCR (atomic)
count = redis.incr(key)
if count > limit:
    redis.decr(key)  # Rollback
    return False
return True

# Or use Lua scripts for complex logic
```

---

## 9. Algorithm Comparison

| Algorithm | Memory | Accuracy | Burst Handling | Complexity |
|-----------|--------|----------|----------------|------------|
| Token Bucket | Low | High | Allows controlled bursts | Low |
| Leaky Bucket | Medium | High | Smooths traffic | Medium |
| Fixed Window | Low | Medium | Boundary issues | Low |
| Sliding Log | High | Perfect | Good | High |
| Sliding Counter | Low | High | Good | Medium |

**Recommendation:**
- **Token Bucket** for API rate limiting (allows bursts)
- **Sliding Window Counter** for strict limits

---

## 10. Edge Cases

### Clock Synchronization
```
Problem: Servers have different times
Solution: Use Redis server time, not local time
         EVAL script with redis.call('TIME')
```

### Client Behind NAT
```
Problem: Many users share same IP
Solution: Prefer API key or user ID over IP
         Higher limits for shared IPs
```

### Distributed Client
```
Problem: Same user from multiple IPs
Solution: Rate limit by user ID, not IP
```

---

## 11. Interview Tips

**Key Points to Cover:**
1. Clarify requirements (per-user? per-IP? per-endpoint?)
2. Discuss algorithm trade-offs
3. Explain distributed challenges
4. Show Redis implementation
5. Mention response headers (HTTP 429)
6. Discuss race conditions and solutions

**Questions to Ask:**
- What's the expected QPS?
- Is it per user, IP, or API key?
- Do we need different limits per endpoint?
- What's the acceptable accuracy?
- What should happen when limit is exceeded?
