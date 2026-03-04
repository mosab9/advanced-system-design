# Design Twitter / X

A comprehensive social media platform design covering feeds, timelines, and real-time updates.

---

## 1. Requirements

### Functional Requirements
- Post tweets (text, images, videos)
- Follow/unfollow users
- View home timeline (tweets from followed users)
- View user timeline (user's own tweets)
- Like, retweet, reply to tweets
- Search tweets and users

### Non-Functional Requirements
- High availability
- Low latency for timeline reads (< 200ms)
- Eventual consistency acceptable for timeline
- Scale: 500M DAU, 200M tweets/day

### Out of Scope
- Direct messages
- Notifications
- Trending topics
- Ads system

---

## 2. Capacity Estimation

### Traffic
```
Users:
- 500M DAU
- Average user:
  - Views timeline: 10 times/day
  - Posts: 0.4 tweets/day (200M total)
  - Follows: 200 users on average

Read QPS (timeline):
= 500M × 10 / 86400
≈ 58,000 QPS
Peak: ~175,000 QPS

Write QPS (tweets):
= 200M / 86400
≈ 2,300 QPS
Peak: ~7,000 QPS

Read:Write ratio ≈ 25:1 (read-heavy)
```

### Storage
```
Tweet:
- ID: 8 bytes
- User ID: 8 bytes
- Content: 280 chars × 2 bytes = 560 bytes
- Metadata: ~200 bytes
- Total: ~800 bytes

Daily tweet storage:
= 200M × 800 bytes = 160 GB/day

5-year storage:
= 160 GB × 365 × 5 = 292 TB

Media storage (assuming 20% have media):
= 40M × 500KB = 20 TB/day
= 20 TB × 365 × 5 = 36.5 PB
```

### Bandwidth
```
Read bandwidth:
= 175,000 QPS × 1MB (timeline with media)
= 175 GB/s (needs CDN!)

Write bandwidth:
= 7,000 QPS × 1MB
= 7 GB/s
```

---

## 3. API Design

### Tweet APIs
```
POST /api/v1/tweets
{
    "content": "Hello, world!",
    "media_ids": ["abc123"],
    "reply_to": null
}
Response: { "tweet_id": "123456789" }

GET /api/v1/tweets/{tweet_id}
Response: {
    "id": "123456789",
    "user": { "id": "1", "username": "john" },
    "content": "Hello, world!",
    "created_at": "2024-01-15T10:30:00Z",
    "likes_count": 42,
    "retweets_count": 5,
    "replies_count": 3
}

DELETE /api/v1/tweets/{tweet_id}
```

### Timeline APIs
```
GET /api/v1/timeline/home?cursor=abc&limit=20
Response: {
    "tweets": [...],
    "next_cursor": "def456"
}

GET /api/v1/users/{user_id}/tweets?cursor=abc&limit=20
Response: {
    "tweets": [...],
    "next_cursor": "xyz789"
}
```

### Social APIs
```
POST /api/v1/users/{user_id}/follow
DELETE /api/v1/users/{user_id}/follow

POST /api/v1/tweets/{tweet_id}/like
DELETE /api/v1/tweets/{tweet_id}/like

POST /api/v1/tweets/{tweet_id}/retweet
```

---

## 4. Data Model

```sql
-- Users
CREATE TABLE users (
    id              BIGINT PRIMARY KEY,
    username        VARCHAR(50) UNIQUE,
    display_name    VARCHAR(100),
    bio             VARCHAR(280),
    followers_count BIGINT DEFAULT 0,
    following_count BIGINT DEFAULT 0,
    created_at      TIMESTAMP
);

-- Tweets
CREATE TABLE tweets (
    id              BIGINT PRIMARY KEY,
    user_id         BIGINT,
    content         VARCHAR(560),
    reply_to_id     BIGINT,
    created_at      TIMESTAMP,
    likes_count     BIGINT DEFAULT 0,
    retweets_count  BIGINT DEFAULT 0,

    INDEX (user_id, created_at DESC)
);

-- Follows
CREATE TABLE follows (
    follower_id     BIGINT,
    followee_id     BIGINT,
    created_at      TIMESTAMP,
    PRIMARY KEY (follower_id, followee_id),
    INDEX (followee_id, follower_id)
);

-- Home Timeline (pre-computed)
CREATE TABLE home_timeline (
    user_id         BIGINT,
    tweet_id        BIGINT,
    created_at      TIMESTAMP,
    PRIMARY KEY (user_id, tweet_id),
    INDEX (user_id, created_at DESC)
);
```

---

## 5. High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                            Clients                                  │
└─────────────────────────────────┬───────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         CDN (Media)                                 │
└─────────────────────────────────┬───────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        Load Balancer                                │
└─────────────────────────────────┬───────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         API Gateway                                 │
│              (Auth, Rate Limiting, Routing)                         │
└─────────────────────────────────┬───────────────────────────────────┘
                                  │
        ┌─────────────────────────┼─────────────────────────┐
        │                         │                         │
        ▼                         ▼                         ▼
┌───────────────┐       ┌───────────────┐        ┌───────────────┐
│ Tweet Service │       │Timeline Service│       │ User Service  │
│               │       │               │        │               │
└───────┬───────┘       └───────┬───────┘        └───────┬───────┘
        │                       │                        │
        │     ┌─────────────────┴─────────────────┐     │
        │     │                                   │     │
        ▼     ▼                                   ▼     ▼
┌───────────────┐    ┌───────────────┐    ┌───────────────┐
│  Tweet DB     │    │Timeline Cache │    │   User DB     │
│ (Cassandra)   │    │   (Redis)     │    │ (PostgreSQL)  │
└───────────────┘    └───────────────┘    └───────────────┘
        │
        ▼
┌───────────────────────────────────────────────────────────────────┐
│                       Message Queue (Kafka)                       │
└───────────────────────────────────────────────────────────────────┘
        │
        ▼
┌───────────────┐              ┌───────────────┐
│ Fan-out Service│              │Search Service │
│ (Timeline Gen) │              │(Elasticsearch)│
└───────────────┘              └───────────────┘
```

---

## 6. Timeline Generation - The Core Problem

### The Fan-out Problem

When a user posts a tweet, how do we show it to their followers?

```
User A posts a tweet
User A has 1,000,000 followers

How do we update 1,000,000 timelines?
```

### Approach 1: Fan-out on Write (Push Model)

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Fan-out on Write                                │
│                                                                     │
│   When User A posts:                                               │
│   1. Store tweet in Tweet DB                                       │
│   2. Get list of followers (1M users)                              │
│   3. For each follower: Insert tweet into their timeline cache     │
│                                                                     │
│   User A posts ──▶ [Tweet 123]                                     │
│                         │                                           │
│                         ▼                                           │
│   Followers' timelines updated:                                    │
│   Follower 1: [..., Tweet 123]                                     │
│   Follower 2: [..., Tweet 123]                                     │
│   ...                                                               │
│   Follower 1M: [..., Tweet 123]                                    │
│                                                                     │
│   Timeline read: Just fetch from cache (FAST!)                     │
│                                                                     │
│   Pros: Fast reads                                                 │
│   Cons: Slow writes for popular users, high storage               │
└─────────────────────────────────────────────────────────────────────┘
```

### Approach 2: Fan-out on Read (Pull Model)

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Fan-out on Read                                 │
│                                                                     │
│   When User A posts:                                               │
│   1. Store tweet in Tweet DB                                       │
│   2. Done!                                                          │
│                                                                     │
│   When follower reads timeline:                                    │
│   1. Get list of users they follow                                 │
│   2. Fetch recent tweets from each                                 │
│   3. Merge and sort                                                │
│                                                                     │
│   User B requests timeline ──▶                                     │
│   Get following list: [A, C, D, E...]                              │
│   Fetch tweets from: A's tweets, C's tweets, D's tweets...        │
│   Merge and sort by time                                           │
│   Return top 20                                                    │
│                                                                     │
│   Pros: Fast writes                                                │
│   Cons: Slow reads (especially if following many users)           │
└─────────────────────────────────────────────────────────────────────┘
```

### Approach 3: Hybrid (Twitter's Actual Approach)

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Hybrid Approach                                 │
│                                                                     │
│   Regular users (< 10K followers): Fan-out on Write               │
│   Celebrities (> 10K followers): Fan-out on Read                  │
│                                                                     │
│   Timeline generation:                                             │
│   1. Fetch pre-computed timeline (regular followees)              │
│   2. Fetch celebrity tweets (on-demand)                           │
│   3. Merge and return                                              │
│                                                                     │
│   User B's timeline:                                               │
│   ┌─────────────────────────────────────────────────────────────┐  │
│   │ Pre-computed (regular users): [T1, T2, T5, T8]              │  │
│   │ Celebrity tweets (on-read):   [T3 (Obama), T6 (Taylor)]     │  │
│   │ Merged result:                [T1, T2, T3, T5, T6, T8]      │  │
│   └─────────────────────────────────────────────────────────────┘  │
│                                                                     │
│   This balances write amplification and read latency               │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 7. Detailed Component Design

### Tweet Service

```python
class TweetService:
    def create_tweet(self, user_id, content, media_ids=None):
        # 1. Validate content
        if len(content) > 280:
            raise ValidationError("Tweet too long")

        # 2. Create tweet
        tweet_id = self.id_generator.next_id()
        tweet = Tweet(
            id=tweet_id,
            user_id=user_id,
            content=content,
            created_at=datetime.now()
        )

        # 3. Store in database
        self.tweet_db.save(tweet)

        # 4. Handle media
        if media_ids:
            self.media_service.attach(tweet_id, media_ids)

        # 5. Publish event for fan-out
        self.event_bus.publish(TweetCreatedEvent(
            tweet_id=tweet_id,
            user_id=user_id,
            created_at=tweet.created_at
        ))

        return tweet
```

### Fan-out Service

```python
class FanoutService:
    def handle_tweet_created(self, event):
        user_id = event.user_id
        tweet_id = event.tweet_id

        # Check if celebrity
        follower_count = self.user_service.get_follower_count(user_id)

        if follower_count > CELEBRITY_THRESHOLD:  # e.g., 10,000
            # Don't fan out, will be fetched on-read
            return

        # Fan-out to followers
        followers = self.follow_service.get_followers(user_id)

        # Batch process
        for batch in chunks(followers, 1000):
            self._update_timelines(batch, tweet_id, event.created_at)

    def _update_timelines(self, user_ids, tweet_id, created_at):
        # Use Redis pipeline for efficiency
        pipeline = self.redis.pipeline()
        for user_id in user_ids:
            key = f"timeline:{user_id}"
            pipeline.zadd(key, {tweet_id: created_at.timestamp()})
            pipeline.zremrangebyrank(key, 0, -801)  # Keep last 800
        pipeline.execute()
```

### Timeline Service

```python
class TimelineService:
    def get_home_timeline(self, user_id, cursor=None, limit=20):
        # 1. Get pre-computed timeline from cache
        cache_key = f"timeline:{user_id}"
        cached_tweets = self.redis.zrevrange(
            cache_key,
            start=0 if not cursor else cursor,
            end=limit - 1
        )

        # 2. Get celebrity tweets
        celebrities = self.follow_service.get_celebrity_followees(user_id)
        celebrity_tweets = self._fetch_celebrity_tweets(
            celebrities,
            limit=limit
        )

        # 3. Merge
        all_tweet_ids = self._merge_timelines(
            cached_tweets,
            celebrity_tweets
        )

        # 4. Hydrate tweets (fetch full tweet data)
        tweets = self.tweet_service.get_tweets(all_tweet_ids[:limit])

        # 5. Create cursor for pagination
        next_cursor = all_tweet_ids[limit] if len(all_tweet_ids) > limit else None

        return TimelineResponse(tweets=tweets, next_cursor=next_cursor)
```

---

## 8. Database Choices

### Tweet Storage: Cassandra

```
Why Cassandra:
- High write throughput (200M tweets/day)
- Linear scalability
- No single point of failure
- Tunable consistency

Schema:
CREATE TABLE tweets (
    tweet_id    BIGINT,
    user_id     BIGINT,
    content     TEXT,
    created_at  TIMESTAMP,
    PRIMARY KEY (tweet_id)
);

CREATE TABLE user_tweets (
    user_id     BIGINT,
    created_at  TIMESTAMP,
    tweet_id    BIGINT,
    PRIMARY KEY (user_id, created_at)
) WITH CLUSTERING ORDER BY (created_at DESC);
```

### Timeline Cache: Redis

```
Why Redis:
- Sub-millisecond latency
- Sorted sets for chronological ordering
- Built-in expiration

Structure:
Key: timeline:{user_id}
Value: Sorted Set of tweet_ids (score = timestamp)

Operations:
ZADD timeline:123 1609459200 "tweet_456"  # Add tweet
ZREVRANGE timeline:123 0 19               # Get latest 20
ZREMRANGEBYRANK timeline:123 0 -801       # Trim to 800
```

### User Data: PostgreSQL

```
Why PostgreSQL:
- Strong consistency for user data
- ACID transactions
- Rich querying

Sharding:
- Shard by user_id
- Follower/following in separate table
```

---

## 9. Data Sharding

### Tweet Sharding (by tweet_id)

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Tweet Shards                                 │
│                                                                     │
│   tweet_id contains timestamp (Snowflake ID)                       │
│   shard = hash(tweet_id) % num_shards                              │
│                                                                     │
│   ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐        │
│   │ Shard 0 │    │ Shard 1 │    │ Shard 2 │    │ Shard 3 │        │
│   │         │    │         │    │         │    │         │        │
│   └─────────┘    └─────────┘    └─────────┘    └─────────┘        │
│                                                                     │
│   Advantage: Even distribution, easy to scale                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Follow Graph Sharding

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Follow Relationship                             │
│                                                                     │
│   Store twice for efficient queries both ways:                     │
│                                                                     │
│   followers:{user_id} → [list of follower IDs]                     │
│   following:{user_id} → [list of following IDs]                    │
│                                                                     │
│   Shard by user_id for both                                        │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 10. ID Generation (Snowflake)

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Twitter Snowflake ID                             │
│                                                                     │
│   64-bit ID structure:                                             │
│                                                                     │
│   ┌─────────┬───────────┬──────────┬────────────┐                 │
│   │ Sign    │ Timestamp │ Worker ID│ Sequence   │                 │
│   │ (1 bit) │ (41 bits) │ (10 bits)│ (12 bits)  │                 │
│   └─────────┴───────────┴──────────┴────────────┘                 │
│                                                                     │
│   Timestamp: Milliseconds since custom epoch                       │
│   Worker ID: Machine identifier (1024 machines)                    │
│   Sequence: Counter per millisecond (4096 IDs/ms/machine)         │
│                                                                     │
│   Benefits:                                                        │
│   - Roughly sorted by time                                         │
│   - No coordination needed                                         │
│   - 4M IDs/second per machine                                     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 11. Search

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Search Architecture                             │
│                                                                     │
│   Tweet Created ──▶ Kafka ──▶ Search Indexer ──▶ Elasticsearch     │
│                                                                     │
│   Elasticsearch Index:                                             │
│   {                                                                │
│     "tweet_id": "123456789",                                       │
│     "user_id": "1",                                                │
│     "username": "john",                                            │
│     "content": "Hello world",                                      │
│     "hashtags": ["greeting"],                                      │
│     "created_at": "2024-01-15T10:30:00Z"                          │
│   }                                                                │
│                                                                     │
│   Search query:                                                    │
│   GET /search?q=hello&from=john&since=2024-01-01                  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 12. Caching Strategy

```
┌─────────────────────────────────────────────────────────────────────┐
│                      Cache Layers                                   │
│                                                                     │
│   Layer 1: CDN (media, static assets)                              │
│   Layer 2: Timeline cache (Redis)                                  │
│   Layer 3: Tweet cache (Redis/Memcached)                           │
│   Layer 4: User cache (Redis/Memcached)                            │
│                                                                     │
│   Cache Population:                                                │
│   - Timeline: Write-through on tweet creation                      │
│   - Tweet: Cache-aside with TTL                                    │
│   - User: Cache-aside with longer TTL                              │
│                                                                     │
│   Hot Tweet Handling:                                              │
│   - Viral tweets: Replicate across cache nodes                     │
│   - TTL based on engagement rate                                   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 13. Reliability

### Handling Failures

```
Fan-out Service failure:
- Use message queue (Kafka) for durability
- At-least-once delivery
- Idempotent timeline updates

Timeline Cache failure:
- Fallback to database
- Lazy repopulation on read
- Warm up cache before rotation

Celebrity following edge case:
- If celebrity lookup fails, return partial timeline
- Show banner "Some tweets may be missing"
```

### Rate Limiting

```
Per-user limits:
- Tweet creation: 100 tweets/hour
- Timeline reads: 1000 requests/hour
- Follows: 400 follows/day
```

---

## 14. Trade-offs Summary

| Decision | Trade-off |
|----------|-----------|
| Hybrid fan-out | Complexity vs performance |
| Cassandra for tweets | Consistency vs availability |
| Redis for timelines | Cost vs latency |
| Snowflake IDs | Complexity vs coordination-free |
| 800 tweet timeline limit | Memory vs completeness |

---

## 15. Interview Tips

**Key Discussion Points:**
1. Fan-out strategies (push vs pull vs hybrid)
2. Celebrity problem
3. Data sharding strategy
4. Timeline cache management
5. Consistency vs availability trade-offs

**Questions to Clarify:**
- What's the read/write ratio?
- Do we need real-time updates?
- How important is consistency?
- What's the celebrity threshold?

**Numbers to Know:**
- Twitter's 500M tweets/day
- 200-300ms timeline latency target
- 800 tweets per cached timeline
- 10K+ followers = celebrity
