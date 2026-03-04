# Design YouTube (Video Streaming Platform)

A large-scale video streaming service design.

---

## 1. Requirements

### Functional Requirements
- Upload videos
- Watch/stream videos
- Search videos
- Like/comment/subscribe
- View recommendations

### Non-Functional Requirements
- High availability
- Low latency for video playback
- Global scale (1B+ users)
- Support multiple resolutions

---

## 2. Capacity Estimation

```
Users: 1B DAU
Videos watched: 5 videos/user/day = 5B views/day
Video uploads: 500K/day
Average video: 5 min, 50MB (compressed)

Storage:
- 500K × 50MB = 25TB/day (raw)
- Multiple resolutions (5x) = 125TB/day
- Annual: ~45PB

Bandwidth (read):
- 5B views × 50MB = 250PB/day
- With CDN caching: ~90% served from edge
```

---

## 3. High-Level Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                           Clients                                    │
└───────────────────────────────┬──────────────────────────────────────┘
                                │
            ┌───────────────────┴───────────────────┐
            │                                       │
            ▼                                       ▼
┌───────────────────────┐              ┌───────────────────────────────┐
│   CDN (Edge Servers)  │              │         API Gateway          │
│   Video Streaming     │              │    (Upload, Metadata)        │
└───────────────────────┘              └───────────────────────────────┘
            │                                       │
            │                          ┌────────────┴────────────┐
            │                          │                         │
            │                          ▼                         ▼
            │              ┌─────────────────────┐   ┌───────────────────┐
            │              │   Video Upload      │   │  Video Metadata   │
            │              │   Service           │   │     Service       │
            │              └──────────┬──────────┘   └───────────────────┘
            │                         │
            │                         ▼
            │              ┌─────────────────────┐
            │              │   Message Queue     │
            │              │     (Kafka)         │
            │              └──────────┬──────────┘
            │                         │
            │                         ▼
            │              ┌─────────────────────┐
            │              │ Video Processing    │
            │              │ Pipeline (Encoding) │
            │              └──────────┬──────────┘
            │                         │
            │                         ▼
            └─────────────▶┌─────────────────────┐
                          │   Object Storage    │
                          │   (S3/GCS)          │
                          └─────────────────────┘
```

---

## 4. Video Upload & Processing Pipeline

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Video Processing Pipeline                       │
│                                                                     │
│   1. Upload                                                        │
│      Client ──▶ Upload Service ──▶ Raw Storage (S3)               │
│                      │                                              │
│                      ▼                                              │
│   2. Queue                                                         │
│      Upload Service ──▶ Kafka (video-processing topic)            │
│                                                                     │
│   3. Processing Workers                                            │
│      ┌──────────────────────────────────────────────────────────┐  │
│      │ Worker Pool:                                              │  │
│      │ - Transcoding (H.264, VP9, AV1)                          │  │
│      │ - Multiple resolutions (240p, 360p, 480p, 720p, 1080p,   │  │
│      │                         1440p, 4K)                        │  │
│      │ - Generate thumbnails                                     │  │
│      │ - Extract audio tracks                                    │  │
│      │ - Generate subtitles (speech-to-text)                    │  │
│      └──────────────────────────────────────────────────────────┘  │
│                      │                                              │
│                      ▼                                              │
│   4. Storage                                                       │
│      Processed videos ──▶ Object Storage ──▶ CDN                  │
│                                                                     │
│   5. Update Metadata                                               │
│      Processing complete ──▶ Metadata Service ──▶ Search Index    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 5. Video Streaming (Adaptive Bitrate)

```
┌─────────────────────────────────────────────────────────────────────┐
│                  Adaptive Bitrate Streaming                         │
│                                                                     │
│   Video split into segments (~2-10 seconds each)                   │
│   Multiple quality levels available                                │
│                                                                     │
│   Manifest File (HLS .m3u8 or DASH .mpd):                         │
│   ┌─────────────────────────────────────────────────────────────┐  │
│   │ #EXTM3U                                                     │  │
│   │ #EXT-X-STREAM-INF:BANDWIDTH=800000                         │  │
│   │ video_480p.m3u8                                             │  │
│   │ #EXT-X-STREAM-INF:BANDWIDTH=1400000                        │  │
│   │ video_720p.m3u8                                             │  │
│   │ #EXT-X-STREAM-INF:BANDWIDTH=2800000                        │  │
│   │ video_1080p.m3u8                                            │  │
│   └─────────────────────────────────────────────────────────────┘  │
│                                                                     │
│   Client behavior:                                                 │
│   1. Download manifest                                             │
│   2. Choose quality based on bandwidth                             │
│   3. Download segments sequentially                                │
│   4. Continuously measure bandwidth                                │
│   5. Switch quality dynamically (up/down)                         │
│                                                                     │
│   Poor connection: 480p ──▶ 360p ──▶ 240p                         │
│   Good connection: 480p ──▶ 720p ──▶ 1080p                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 6. CDN Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                     CDN Hierarchy                                   │
│                                                                     │
│                    ┌─────────────────┐                             │
│                    │  Origin Server  │                             │
│                    │  (Object Store) │                             │
│                    └────────┬────────┘                             │
│                             │                                       │
│              ┌──────────────┼──────────────┐                       │
│              ▼              ▼              ▼                       │
│        ┌──────────┐   ┌──────────┐   ┌──────────┐                 │
│        │  Shield  │   │  Shield  │   │  Shield  │                 │
│        │ (US)     │   │ (EU)     │   │ (Asia)   │                 │
│        └────┬─────┘   └────┬─────┘   └────┬─────┘                 │
│             │              │              │                        │
│        ┌────┴────┐    ┌────┴────┐    ┌────┴────┐                  │
│        ▼         ▼    ▼         ▼    ▼         ▼                  │
│   ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐         │
│   │Edge PoP│ │Edge PoP│ │Edge PoP│ │Edge PoP│ │Edge PoP│         │
│   │New York│ │LA     │ │London  │ │Paris   │ │Tokyo   │         │
│   └────────┘ └────────┘ └────────┘ └────────┘ └────────┘         │
│       │                                                           │
│       ▼                                                           │
│   End Users (served from nearest edge)                           │
│                                                                     │
│   Cache strategy:                                                  │
│   - Popular videos: Cached at edge                                │
│   - Less popular: Cached at shield                                │
│   - Rare/old: Fetched from origin                                 │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 7. Database Design

### Video Metadata (SQL/NoSQL)

```sql
-- Videos table
CREATE TABLE videos (
    id              UUID PRIMARY KEY,
    user_id         UUID,
    title           VARCHAR(500),
    description     TEXT,
    duration_sec    INT,
    status          ENUM('processing', 'ready', 'failed'),
    view_count      BIGINT,
    like_count      BIGINT,
    upload_time     TIMESTAMP,
    INDEX (user_id, upload_time DESC)
);

-- Video files (resolutions)
CREATE TABLE video_files (
    video_id        UUID,
    resolution      VARCHAR(10),  -- '1080p', '720p', etc.
    codec           VARCHAR(20),
    file_path       VARCHAR(500),
    file_size       BIGINT,
    PRIMARY KEY (video_id, resolution)
);
```

### View Counts (Real-time)

```
Problem: Millions of concurrent views → can't update DB per view

Solution: Redis + Batch aggregation

┌─────────────────────────────────────────────────────────────────────┐
│   View Event ──▶ Kafka ──▶ View Counter Service                    │
│                                      │                              │
│                           ┌──────────┴──────────┐                  │
│                           ▼                     ▼                  │
│                    ┌──────────┐          ┌──────────┐              │
│                    │  Redis   │          │ Analytics│              │
│                    │(real-time)│         │ (batch)  │              │
│                    └──────────┘          └──────────┘              │
│                           │                                         │
│                 Every 5 min, flush to DB                           │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 8. Key Design Decisions

| Component | Choice | Rationale |
|-----------|--------|-----------|
| Video storage | Object storage (S3) | Cheap, scalable, durable |
| Streaming | HLS/DASH | Adaptive bitrate, wide support |
| Processing | FFmpeg workers | Industry standard, flexible |
| Metadata | PostgreSQL + cache | ACID for critical data |
| View counts | Redis + batch | Handle high write volume |
| Search | Elasticsearch | Full-text search, faceting |

---

## 9. Interview Tips

**Key Discussion Points:**
- Video encoding pipeline (parallel processing)
- Adaptive bitrate streaming (HLS/DASH)
- CDN architecture and caching strategy
- Handling viral videos (hot content)
- Storage cost optimization (cold storage for old videos)

**Questions to Ask:**
- What's the target audience (global/regional)?
- Do we need live streaming?
- What about copyright/content moderation?
- Mobile vs desktop breakdown?
