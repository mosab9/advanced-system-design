# Design Instagram (Photo Sharing)

Photo-centric social media platform with feeds and stories.

---

## 1. Requirements

### Functional
- Upload photos/videos
- Follow users
- View feed (photos from followed users)
- Like, comment on posts
- Stories (24-hour ephemeral content)
- Direct messages

### Non-Functional
- High availability
- Low latency feed loading
- Scale: 1B+ users, 500M DAU

---

## 2. High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Clients                                     │
└───────────────────────────────┬─────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         CDN (Media)                                 │
└───────────────────────────────┬─────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        Load Balancer                                │
└───────────────────────────────┬─────────────────────────────────────┘
                                │
        ┌───────────────────────┼───────────────────────┐
        │                       │                       │
        ▼                       ▼                       ▼
┌───────────────┐      ┌───────────────┐      ┌───────────────┐
│   Post        │      │   Feed        │      │   User        │
│   Service     │      │   Service     │      │   Service     │
└───────┬───────┘      └───────┬───────┘      └───────────────┘
        │                      │
        ▼                      ▼
┌───────────────┐      ┌───────────────┐
│  Media        │      │  Feed Cache   │
│  Storage (S3) │      │  (Redis)      │
└───────────────┘      └───────────────┘
```

---

## 3. Photo Upload Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Upload Flow                                     │
│                                                                     │
│   1. Client uploads image to Upload Service                        │
│   2. Upload Service:                                               │
│      - Stores original in Object Storage (S3)                     │
│      - Generates unique photo ID                                   │
│      - Publishes to processing queue                              │
│                                                                     │
│   3. Image Processing Workers:                                     │
│      - Generate thumbnails (150x150, 320x320, 640x640)            │
│      - Apply filters (if requested)                               │
│      - Generate multiple sizes for responsive display             │
│      - Store processed images in S3                               │
│                                                                     │
│   4. Post Service:                                                 │
│      - Creates post metadata                                       │
│      - Publishes PostCreated event                                │
│                                                                     │
│   5. Feed Service (async):                                         │
│      - Fan-out to followers' feeds                                │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 4. Feed Generation (Hybrid Fan-out)

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Feed Generation                                 │
│                                                                     │
│   Regular users (< 10K followers): Fan-out on Write               │
│   - When post created, push to all followers' feeds               │
│   - Pre-computed feeds in Redis                                   │
│                                                                     │
│   Celebrities (> 10K followers): Fan-out on Read                  │
│   - Don't pre-compute for millions of followers                   │
│   - Merge celebrity posts when user requests feed                 │
│                                                                     │
│   User's Feed = Merge(                                            │
│       Redis cached feed,                       (regular followees)│
│       Recent posts from celebrity followees    (fetched on read)  │
│   )                                                                │
│                                                                     │
│   Then apply ranking algorithm:                                    │
│   score = f(recency, engagement, user_affinity, content_type)     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 5. Stories System

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Stories Architecture                            │
│                                                                     │
│   Stories are ephemeral (24 hour TTL)                              │
│                                                                     │
│   Storage:                                                         │
│   - Redis with TTL for fast access                                │
│   - Key: stories:{user_id}                                        │
│   - Value: List of story objects                                  │
│   - TTL: 24 hours                                                 │
│                                                                     │
│   When user opens app:                                             │
│   1. Get list of followed users                                   │
│   2. Check which have active stories (Redis MGET)                 │
│   3. Return story tray (user avatars with stories)               │
│                                                                     │
│   Story views:                                                     │
│   - Track who viewed (for creator to see)                        │
│   - Write to Cassandra (high write volume)                       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 6. Data Model

```sql
-- Posts
CREATE TABLE posts (
    id              UUID PRIMARY KEY,
    user_id         UUID,
    caption         TEXT,
    location        VARCHAR(200),
    created_at      TIMESTAMP,
    likes_count     INT,
    comments_count  INT
);

-- Post Media (multiple images per post)
CREATE TABLE post_media (
    post_id         UUID,
    media_order     INT,
    media_type      VARCHAR(20),
    url_original    VARCHAR(500),
    url_thumbnail   VARCHAR(500),
    PRIMARY KEY (post_id, media_order)
);

-- Feed (pre-computed)
CREATE TABLE user_feed (
    user_id         UUID,
    post_id         UUID,
    post_user_id    UUID,
    created_at      TIMESTAMP,
    PRIMARY KEY (user_id, created_at, post_id)
) WITH CLUSTERING ORDER BY (created_at DESC);
```

---

## 7. Key Design Decisions

| Component | Choice | Rationale |
|-----------|--------|-----------|
| Photo Storage | S3 + CDN | Scalable, cheap, global distribution |
| Feed Cache | Redis | Fast read, easy TTL |
| Feed Generation | Hybrid fan-out | Balance write/read costs |
| Database | Cassandra | High write throughput |
| Search | Elasticsearch | Hashtag, user search |

---

## 8. Interview Tips

**Key Points:**
- Image processing pipeline
- Hybrid fan-out for feed (celebrity problem)
- CDN for global photo delivery
- Stories with TTL-based expiration
- Feed ranking algorithm

**Questions to Ask:**
- Feed chronological or ranked?
- Video support?
- Multi-photo posts (carousel)?
- International scale?
