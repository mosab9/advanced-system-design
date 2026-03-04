# Design a Web Crawler

Distributed system for crawling and indexing the web.

---

## 1. Requirements

### Functional
- Crawl web pages starting from seed URLs
- Extract links and continue crawling
- Store page content for indexing
- Respect robots.txt
- Handle dynamic content (JavaScript)

### Non-Functional
- Scalability (billions of pages)
- Politeness (don't overload sites)
- Freshness (recrawl updated pages)
- Robustness (handle failures)

---

## 2. High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│   ┌─────────────┐     ┌─────────────┐     ┌─────────────┐         │
│   │ Seed URLs   │────▶│ URL Frontier│────▶│  Crawler    │         │
│   └─────────────┘     │   (Queue)   │     │  Workers    │         │
│                       └─────────────┘     └──────┬──────┘         │
│                              ▲                   │                 │
│                              │                   ▼                 │
│                       ┌──────┴──────┐     ┌─────────────┐         │
│                       │    Link     │◀────│   HTML      │         │
│                       │  Extractor  │     │   Parser    │         │
│                       └─────────────┘     └──────┬──────┘         │
│                                                  │                 │
│                                                  ▼                 │
│   ┌─────────────┐     ┌─────────────┐     ┌─────────────┐         │
│   │ URL Seen?   │◀────│    Dedup    │◀────│  Content    │         │
│   │  (Bloom)    │     │   Filter    │     │   Store     │         │
│   └─────────────┘     └─────────────┘     └─────────────┘         │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 3. URL Frontier (Priority Queue)

```
┌─────────────────────────────────────────────────────────────────────┐
│                     URL Frontier Design                             │
│                                                                     │
│   Two-level queue:                                                 │
│                                                                     │
│   Front Queues (Prioritization):                                   │
│   ┌─────────────────────────────────────────────────────────────┐  │
│   │ Q1 (High):   [google.com, nytimes.com, ...]                │  │
│   │ Q2 (Medium): [blog.example.com, ...]                        │  │
│   │ Q3 (Low):    [random-site.xyz, ...]                         │  │
│   └─────────────────────────────────────────────────────────────┘  │
│                                                                     │
│   Back Queues (Politeness - one per host):                        │
│   ┌─────────────────────────────────────────────────────────────┐  │
│   │ google.com:     [url1, url2, url3] → crawl at 1 req/sec    │  │
│   │ example.com:    [url4, url5] → crawl at 0.5 req/sec        │  │
│   │ small-site.com: [url6] → crawl at 0.1 req/sec              │  │
│   └─────────────────────────────────────────────────────────────┘  │
│                                                                     │
│   Ensures:                                                         │
│   - High-value pages crawled first                                │
│   - No single host overwhelmed                                    │
│   - Respects crawl-delay from robots.txt                         │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 4. URL Deduplication

```
┌─────────────────────────────────────────────────────────────────────┐
│                     URL Deduplication                               │
│                                                                     │
│   Problem: Don't crawl same URL twice                              │
│   Scale: Billions of URLs → can't store all in memory             │
│                                                                     │
│   Solution: Bloom Filter                                           │
│                                                                     │
│   Properties:                                                      │
│   - Space efficient: 10 bits per URL                              │
│   - Fast: O(k) where k = hash functions                           │
│   - May have false positives (skip some URLs)                     │
│   - No false negatives (never crawl twice)                        │
│                                                                     │
│   check_url(url):                                                  │
│       if bloom_filter.might_contain(url):                         │
│           # Probably seen, skip                                    │
│           return False                                             │
│       else:                                                        │
│           # Definitely new                                         │
│           bloom_filter.add(url)                                    │
│           return True                                              │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 5. Content Deduplication

```
Problem: Same content at different URLs (mirrors, duplicates)

Solution: Content fingerprinting

1. Simhash/MinHash for near-duplicate detection
2. Compare fingerprints, not full content
3. Store only unique content

fingerprint = simhash(page_content)
if similar_fingerprint_exists(fingerprint):
    mark_as_duplicate()
else:
    store_content()
```

---

## 6. Politeness

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Politeness Rules                                │
│                                                                     │
│   1. robots.txt compliance                                         │
│      User-agent: *                                                 │
│      Disallow: /private/                                           │
│      Crawl-delay: 10                                               │
│                                                                     │
│   2. Rate limiting per host                                        │
│      - Default: 1 request per second per host                     │
│      - Adjust based on server response time                       │
│                                                                     │
│   3. Time-of-day awareness                                         │
│      - Prefer off-peak hours for small sites                      │
│                                                                     │
│   4. Exponential backoff on errors                                 │
│      - 5xx errors: wait 1s, 2s, 4s, 8s...                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 7. Distributed Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Distributed Crawling                            │
│                                                                     │
│   Partition by URL hash:                                           │
│   worker_id = hash(url) % num_workers                              │
│                                                                     │
│   Or partition by domain:                                          │
│   worker_id = hash(domain) % num_workers                           │
│   (Better for politeness - one worker per domain)                 │
│                                                                     │
│   ┌──────────┐     ┌──────────┐     ┌──────────┐                  │
│   │ Worker 1 │     │ Worker 2 │     │ Worker N │                  │
│   │*.com     │     │*.org     │     │*.edu     │                  │
│   └──────────┘     └──────────┘     └──────────┘                  │
│        │               │               │                          │
│        └───────────────┼───────────────┘                          │
│                        ▼                                           │
│                 ┌──────────────┐                                   │
│                 │   Central    │                                   │
│                 │   Storage    │                                   │
│                 └──────────────┘                                   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 8. Interview Tips

**Key Points:**
- URL frontier design (prioritization + politeness)
- Bloom filter for URL deduplication
- Robots.txt compliance
- Distributed crawling strategies
- Content fingerprinting for dedup

**Questions to Ask:**
- Scale (how many pages)?
- Freshness requirements?
- JavaScript rendering needed?
- Crawl entire web or specific domains?
