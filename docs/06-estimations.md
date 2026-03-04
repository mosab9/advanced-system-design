# Back-of-Envelope Calculations

Essential numbers and estimation techniques for system design interviews.

---

## 1. Numbers Every Engineer Should Know

### Data Size Units

```
1 Byte      = 8 bits
1 KB        = 1,000 bytes           = 10^3 bytes
1 MB        = 1,000 KB              = 10^6 bytes
1 GB        = 1,000 MB              = 10^9 bytes
1 TB        = 1,000 GB              = 10^12 bytes
1 PB        = 1,000 TB              = 10^15 bytes

Binary (actual):
1 KiB       = 1,024 bytes           = 2^10 bytes
1 MiB       = 1,024 KiB             = 2^20 bytes
1 GiB       = 1,024 MiB             = 2^30 bytes

For estimation, use powers of 10 (simpler math)
```

### Time Units

```
1 second        = 1,000 ms (milliseconds)
1 ms            = 1,000 μs (microseconds)
1 μs            = 1,000 ns (nanoseconds)

Useful conversions:
1 day           = 86,400 seconds      ≈ 100,000 seconds
1 week          = 604,800 seconds     ≈ 600,000 seconds
1 month         = 2.6 million seconds ≈ 2.5 × 10^6 seconds
1 year          = 31.5 million seconds ≈ 3 × 10^7 seconds
```

### Latency Numbers

```
┌─────────────────────────────────────────────────────────────┐
│                    Latency Comparison                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  L1 cache reference                    0.5 ns               │
│  Branch mispredict                     5 ns                 │
│  L2 cache reference                    7 ns                 │
│  Mutex lock/unlock                     25 ns                │
│  Main memory reference                 100 ns               │
│                                                             │
│  Compress 1KB with Snappy              3,000 ns    (3 μs)   │
│  Send 1KB over 1 Gbps network          10,000 ns   (10 μs)  │
│  Read 4KB randomly from SSD            150,000 ns  (150 μs) │
│  Read 1MB sequentially from memory     250,000 ns  (250 μs) │
│  Round trip within same datacenter     500,000 ns  (0.5 ms) │
│  Read 1MB sequentially from SSD        1,000,000 ns (1 ms)  │
│  HDD seek                              10,000,000 ns (10 ms)│
│  Read 1MB sequentially from HDD        20,000,000 ns (20 ms)│
│  Send packet CA → Netherlands → CA     150,000,000 ns       │
│                                        (150 ms)             │
│                                                             │
└─────────────────────────────────────────────────────────────┘

Key Insights:
- Memory is 100x faster than SSD random read
- SSD is 10x faster than HDD
- Network within datacenter: ~0.5ms
- Cross-continental network: ~150ms
```

### Throughput Numbers

```
┌─────────────────────────────────────────────────────────────┐
│                    Throughput Reference                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Network:                                                   │
│  - 1 Gbps = 125 MB/s                                       │
│  - 10 Gbps = 1.25 GB/s                                     │
│                                                             │
│  Storage:                                                   │
│  - HDD sequential: 100-200 MB/s                            │
│  - SSD sequential: 500-3000 MB/s (NVMe)                    │
│                                                             │
│  Database:                                                  │
│  - PostgreSQL: ~10K-50K QPS (depends on query)             │
│  - MySQL: ~10K-50K QPS                                     │
│  - Redis: 100K-500K QPS                                    │
│  - Cassandra: 20K-50K writes/s per node                    │
│                                                             │
│  Web Server:                                                │
│  - Single server can handle: 1K-10K RPS                    │
│  - With async/optimized: 10K-100K RPS                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. Common Data Sizes

### Text Data

```
Character (ASCII)           = 1 byte
Character (UTF-8, avg)      = 1-4 bytes (avg ~2 bytes)
Tweet (280 chars)           ≈ 300 bytes (with overhead)
Email (avg)                 ≈ 50 KB
Web page (avg)              ≈ 2 MB
Book (plain text)           ≈ 1 MB
```

### Media Data

```
Image (thumbnail)           ≈ 10 KB
Image (web quality)         ≈ 200 KB - 500 KB
Image (high resolution)     ≈ 2-5 MB
Photo (raw)                 ≈ 20-50 MB

Audio (1 min, MP3)          ≈ 1 MB
Audio (1 min, high quality) ≈ 10 MB

Video (1 min, 480p)         ≈ 20-40 MB
Video (1 min, 1080p)        ≈ 100-200 MB
Video (1 min, 4K)           ≈ 300-500 MB

Compression ratios (approximate):
- Text: 10:1 (gzip)
- Images: Already compressed (JPEG, PNG)
- Video: 100:1 or more (H.264, H.265)
```

### Database Records

```
User profile (basic)        ≈ 1 KB
User profile (extended)     ≈ 10 KB
Product listing             ≈ 5-10 KB
Order record                ≈ 2-5 KB
Log entry                   ≈ 200-500 bytes
Session data                ≈ 1-5 KB
```

---

## 3. Traffic Estimation

### QPS Calculation

```
Given: 1 billion daily active users (DAU)
       Each user makes 10 requests per day

Daily requests = 1B × 10 = 10B requests/day

QPS (Queries Per Second):
QPS = 10B / 86,400 seconds
QPS ≈ 10B / 100K (simplify)
QPS ≈ 100,000 QPS

Peak QPS (assume 2-3x average):
Peak QPS ≈ 300,000 QPS
```

### Quick QPS Formula

```
QPS = (DAU × actions_per_user) / 86,400
    ≈ (DAU × actions_per_user) / 100,000

Peak QPS = QPS × 2 to 3

Example calculations:
┌──────────────┬─────────────┬─────────────┬────────────┐
│     DAU      │ Actions/Day │   Avg QPS   │  Peak QPS  │
├──────────────┼─────────────┼─────────────┼────────────┤
│   1 Million  │      10     │    ~100     │   ~300     │
│  10 Million  │      10     │   ~1,000    │  ~3,000    │
│ 100 Million  │      10     │  ~10,000    │  ~30,000   │
│   1 Billion  │      10     │ ~100,000    │ ~300,000   │
└──────────────┴─────────────┴─────────────┴────────────┘
```

---

## 4. Storage Estimation

### Daily Storage Calculation

```
Example: Twitter-like service
- 500M DAU
- 20% post daily = 100M tweets/day
- Average tweet = 500 bytes

Daily storage = 100M × 500 bytes
             = 50 × 10^9 bytes
             = 50 GB/day

5-year storage:
= 50 GB × 365 × 5
= 50 GB × 1,825
≈ 91 TB (data only)

With replication (3x):
≈ 270 TB

With overhead (indexes, metadata) (2x):
≈ 540 TB ≈ 0.5 PB
```

### Storage Formula

```
Daily Storage = (daily_items) × (item_size)

Total Storage = Daily Storage × days × replication_factor × overhead_factor

Typical factors:
- Replication: 3x for durability
- Overhead: 1.5x to 2x for indexes, metadata, deleted data
```

### Media Storage Example

```
Instagram-like service:
- 100M photos uploaded daily
- Average photo: 500 KB (after compression)

Daily storage = 100M × 500 KB = 50 TB/day

Annual storage = 50 TB × 365 = 18.25 PB/year

With CDN caching:
- Store originals: 18 PB
- CDN serves resized versions
- Hot content (20%): 3.6 PB cached at CDN
```

---

## 5. Bandwidth Estimation

### Bandwidth Calculation

```
Bandwidth = QPS × response_size

Example: Image service
- 100K QPS
- Average image: 200 KB

Egress bandwidth = 100K × 200 KB/s
                = 20,000,000 KB/s
                = 20 GB/s
                = 160 Gbps

Daily egress = 20 GB/s × 86,400
            ≈ 1.7 PB/day
```

### Read vs Write Ratio

```
Most systems are read-heavy:
- Social media: 100:1 to 1000:1 (read:write)
- E-commerce: 10:1 to 100:1
- Messaging: 1:1 to 10:1

Example: Twitter
- Reads: 100K QPS
- Writes: 1K QPS (100:1 ratio)

Plan for:
- Write bandwidth: uploads, posts
- Read bandwidth: views, downloads (usually 10-100x writes)
```

---

## 6. Memory/Cache Estimation

### Cache Size (80/20 Rule)

```
Principle: 80% of traffic hits 20% of data

Example:
- 1B daily requests
- Each request accesses 1 KB of data
- Total data accessed: 1 TB

Cache 20% of hot data:
Cache size = 1 TB × 0.2 = 200 GB

With typical overhead (25%):
Cache size ≈ 250 GB

Can fit in single Redis server (up to 300GB)
or distributed cache with 3 nodes (100GB each)
```

### Cache Hit Rate Impact

```
Cache hit rate matters a lot:

Scenario: 100K QPS
- Cache latency: 1ms
- DB latency: 100ms

At 99% hit rate:
- 99K cache hits: 99K × 1ms = 99 seconds of latency
- 1K cache misses: 1K × 100ms = 100 seconds of latency
- Avg latency: ~2ms

At 95% hit rate:
- 95K cache hits: 95K × 1ms = 95 seconds
- 5K cache misses: 5K × 100ms = 500 seconds
- Avg latency: ~6ms (3x worse!)

At 90% hit rate:
- Avg latency: ~11ms (5.5x worse!)

Conclusion: Even small improvements in cache hit rate matter significantly
```

---

## 7. Server Estimation

### Servers Needed

```
Rule of thumb (varies widely):
- 1 web server: 1K-10K QPS (simple requests)
- 1 app server: 1K-5K QPS (business logic)
- 1 DB server: 10K-50K QPS (varies by query)

Example: 100K QPS service
- Web tier: 100K / 5K = 20 servers
- App tier: 100K / 2K = 50 servers
- DB tier: Sharded across 5-10 servers

Add redundancy (N+2 or 1.5x):
- Web: 30 servers
- App: 75 servers
- DB: 15 servers
```

### Connection Limits

```
TCP connections per server:
- Theoretical max: 65,535 ports
- Practical limit: 10K-50K concurrent connections

If each user holds 1 connection:
- 1M concurrent users needs 20-100 servers just for connections

Long-polling/WebSocket considerations:
- Users hold connections longer
- Need more servers for same user count
```

---

## 8. Worked Examples

### Example 1: URL Shortener

```
Requirements:
- 100M URLs shortened per day
- 10:1 read to write ratio
- Keep URLs for 5 years

Write QPS:
= 100M / 86,400
≈ 1,150 QPS
Peak: ~3,500 QPS

Read QPS:
= 1,150 × 10
= 11,500 QPS
Peak: ~35,000 QPS

Storage:
- URL entry: ~500 bytes (short URL + original + metadata)
- Daily: 100M × 500 bytes = 50 GB
- 5 years: 50 GB × 365 × 5 = 91 TB
- With replication (3x): 273 TB

Cache:
- Assume 20% of URLs are hot
- Daily data: 50 GB
- Cache: 50 GB × 0.2 = 10 GB
```

### Example 2: Twitter Timeline

```
Requirements:
- 500M DAU
- Each user: 5 home timeline requests/day
- Average user follows 200 accounts
- 100M tweets/day

Timeline reads:
= 500M × 5 / 86,400
≈ 29,000 QPS
Peak: ~90,000 QPS

Tweet writes:
= 100M / 86,400
≈ 1,150 QPS
Peak: ~3,500 QPS

Fan-out consideration:
- Average followers: 200
- Celebrity followers: 10M+

Fan-out on write (pre-compute timelines):
- 100M tweets × 200 avg followers
= 20B timeline insertions/day
= 230K insertions/second (!)

Solution: Hybrid approach
- Regular users: fan-out on write
- Celebrities: fan-out on read
```

### Example 3: Video Streaming (YouTube)

```
Requirements:
- 1B DAU
- Each user watches 5 videos (avg 5 min each)
- Video bitrate: 5 Mbps (1080p)
- 500K video uploads/day

Video watch bandwidth:
- Hours watched: 1B × 5 × 5 min = 25B minutes = 417M hours/day
- Concurrent viewers (peak): ~50M
- Bandwidth: 50M × 5 Mbps = 250 Tbps (!)

This is why CDN is critical.

Video storage (uploads):
- 500K uploads/day
- Avg video: 10 min @ 5 Mbps = 375 MB
- Multiple resolutions (4x): 1.5 GB per video
- Daily storage: 500K × 1.5 GB = 750 TB/day
- Annual: 274 PB/year
```

---

## 9. Quick Reference Tables

### Powers of 2

```
2^10 = 1,024           ≈ 1 thousand (1 KB)
2^20 = 1,048,576       ≈ 1 million (1 MB)
2^30 = 1,073,741,824   ≈ 1 billion (1 GB)
2^40 = ...             ≈ 1 trillion (1 TB)
```

### Useful Multipliers

```
Seconds in a day: 86,400    ≈ 10^5 (use 100K)
Seconds in a year: 31.5M    ≈ 3 × 10^7
Days in a year: 365         ≈ 400
```

### Scale Reference

```
┌──────────────────┬─────────────┬────────────────┐
│     Company      │     DAU     │   Requests/day │
├──────────────────┼─────────────┼────────────────┤
│   Small startup  │    10K      │      1M        │
│   Growing app    │   100K      │     10M        │
│   Popular app    │     1M      │    100M        │
│   Major platform │    10M      │      1B        │
│   Top 10 site    │   100M      │     10B        │
│   Facebook/Google│     1B+     │    100B+       │
└──────────────────┴─────────────┴────────────────┘
```

---

## 10. Estimation Tips

### Round Numbers Liberally

```
Good: 86,400 seconds/day → 100,000
Good: 31.5M seconds/year → 30M
Good: 365 days/year → 400

Precision doesn't matter. Order of magnitude matters.
```

### State Your Assumptions

```
"I'll assume each user makes 10 requests per day..."
"Let's say the average image is 200KB..."
"Assuming a 100:1 read to write ratio..."

Interviewers want to see your reasoning, not perfect numbers.
```

### Sanity Check Results

```
If you calculate 1 PB storage for a to-do app: Wrong!
If you calculate 10 servers for YouTube: Wrong!

Compare to known systems:
- Google: ~15 exabytes stored
- Netflix: ~10 PB bandwidth/day
- Twitter: ~500M tweets/day
```

### Show Your Work

```
Don't: "We need about 100 servers"
Do: "100M users × 10 requests = 1B/day
     1B / 100K seconds ≈ 10K QPS
     At 500 QPS per server: 10K/500 = 20 servers
     With 3x redundancy: ~60-70 servers"
```
