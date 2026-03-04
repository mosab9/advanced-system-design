# Advanced System Design - FAANG Interview Reference

A comprehensive system design guide for Senior Software Engineer interviews at top tech companies.

## Table of Contents

### Core Documentation
| Document | Description |
|----------|-------------|
| [System Design Framework](docs/01-interview-framework.md) | Step-by-step approach for system design interviews |
| [Fundamentals](docs/02-fundamentals.md) | CAP theorem, ACID, BASE, consistency models |
| [Building Blocks](docs/03-building-blocks.md) | Databases, caches, load balancers, queues |
| [Scalability Patterns](docs/04-scalability-patterns.md) | Horizontal scaling, sharding, replication |
| [Design Patterns](docs/05-design-patterns.md) | Common architectural patterns |
| [Back-of-Envelope Calculations](docs/06-estimations.md) | Capacity planning and math |

### System Design Examples
| System | Key Concepts |
|--------|--------------|
| [URL Shortener](designs/url-shortener.md) | Hashing, base62, KGS |
| [Rate Limiter](designs/rate-limiter.md) | Token bucket, sliding window |
| [Distributed Cache](designs/distributed-cache.md) | Consistent hashing, eviction |
| [Message Queue](designs/message-queue.md) | Pub/sub, ordering, delivery |
| [Twitter/X](designs/twitter.md) | Fan-out, timeline, feeds |
| [YouTube](designs/youtube.md) | Video processing, CDN, streaming |
| [Uber](designs/uber.md) | Geospatial, matching, real-time |
| [WhatsApp](designs/whatsapp.md) | Real-time messaging, presence |
| [Instagram](designs/instagram.md) | Photo storage, feeds, stories |
| [Dropbox](designs/dropbox.md) | File sync, chunking, dedup |
| [Web Crawler](designs/web-crawler.md) | Distributed crawling, politeness |
| [Search Engine](designs/search-engine.md) | Indexing, ranking, PageRank |
| [Notification System](designs/notification-system.md) | Push, email, SMS delivery |
| [Payment System](designs/payment-system.md) | Transactions, idempotency |
| [Ticket Booking](designs/ticket-booking.md) | Inventory, concurrency |

## Quick Reference

### The RESHADED Framework
```
R - Requirements (functional & non-functional)
E - Estimation (traffic, storage, bandwidth)
S - Storage schema (data model)
H - High-level design (components)
A - API design (endpoints)
D - Detailed design (deep dives)
E - Evaluation (trade-offs)
D - Distinctive features (what makes it unique)
```

### Key Numbers to Remember
```
1 byte     = 8 bits
1 KB       = 1,000 bytes
1 MB       = 1,000 KB = 10^6 bytes
1 GB       = 1,000 MB = 10^9 bytes
1 TB       = 1,000 GB = 10^12 bytes

Latency Numbers:
L1 cache:           0.5 ns
L2 cache:           7 ns
RAM:                100 ns
SSD random read:    150 μs
HDD seek:           10 ms
Network round trip: 0.5-150 ms
```

### Scale Reference
```
Monthly Active Users (MAU):
- Small:     < 1M
- Medium:    1M - 100M
- Large:     100M - 1B
- Massive:   > 1B

Daily Active Users (DAU) ≈ 20-50% of MAU
QPS = DAU × actions_per_day / 86400
Peak QPS ≈ 2-3× average QPS
```

## How to Use This Guide

1. **Before Interview**: Study fundamentals and building blocks
2. **Practice**: Work through design examples with a timer (45 min)
3. **During Interview**: Use the RESHADED framework
4. **Deep Dive**: Know 2-3 designs deeply for detailed questions

## Contributing

Feel free to add new designs or improve existing documentation.

## License

MIT License - Use freely for interview preparation.
