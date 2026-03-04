# System Design Building Blocks

Essential components used in distributed system architectures.

---

## 1. Load Balancers

Distribute traffic across multiple servers to ensure high availability and reliability.

### Types of Load Balancers

```
┌─────────────────────────────────────────────────────────────┐
│                        Layer 4 (L4)                        │
│                     Transport Layer                         │
│              Works with TCP/UDP connections                │
│              Faster, but limited routing options           │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                        Layer 7 (L7)                        │
│                     Application Layer                       │
│              Works with HTTP/HTTPS content                 │
│              Can route based on URL, headers, cookies      │
└─────────────────────────────────────────────────────────────┘
```

### Load Balancing Algorithms

| Algorithm | Description | Use Case |
|-----------|-------------|----------|
| **Round Robin** | Rotate through servers sequentially | Equal server capacity |
| **Weighted Round Robin** | Assign weights to servers | Different server capacities |
| **Least Connections** | Route to server with fewest connections | Variable request times |
| **Least Response Time** | Route to fastest responding server | Performance-critical |
| **IP Hash** | Hash client IP for server selection | Session persistence |
| **Consistent Hashing** | Minimize redistribution on changes | Caching layers |

### Health Checks

```
┌────────────────┐
│ Load Balancer  │
└───────┬────────┘
        │
        ├──── Health Check ────▶ Server 1 ✓ (healthy)
        │     GET /health
        │
        ├──── Health Check ────▶ Server 2 ✗ (removed)
        │     Timeout
        │
        └──── Health Check ────▶ Server 3 ✓ (healthy)
```

### Real-World Products
- **Hardware:** F5, Citrix NetScaler
- **Software:** HAProxy, NGINX, Envoy
- **Cloud:** AWS ALB/NLB, GCP Load Balancer, Azure Load Balancer

---

## 2. Databases

### SQL vs NoSQL Decision Tree

```
                    Start
                      │
                      ▼
            ┌─────────────────┐
            │ Need ACID       │───Yes──▶ SQL
            │ transactions?   │
            └────────┬────────┘
                     │ No
                     ▼
            ┌─────────────────┐
            │ Complex queries │───Yes──▶ SQL
            │ and JOINs?      │
            └────────┬────────┘
                     │ No
                     ▼
            ┌─────────────────┐
            │ Flexible schema │───Yes──▶ Document DB
            │ needed?         │
            └────────┬────────┘
                     │ No
                     ▼
            ┌─────────────────┐
            │ High write      │───Yes──▶ Wide-Column DB
            │ throughput?     │
            └────────┬────────┘
                     │ No
                     ▼
            ┌─────────────────┐
            │ Graph relations │───Yes──▶ Graph DB
            │ needed?         │
            └────────┬────────┘
                     │ No
                     ▼
            ┌─────────────────┐
            │ Simple key-value│───Yes──▶ Key-Value DB
            │ access pattern? │
            └─────────────────┘
```

### Database Types Comparison

| Type | Examples | Use Cases | Strengths |
|------|----------|-----------|-----------|
| **Relational** | PostgreSQL, MySQL | OLTP, complex queries | ACID, JOINs, mature |
| **Document** | MongoDB, CouchDB | Flexible schema, JSON | Developer friendly |
| **Wide-Column** | Cassandra, HBase | Time-series, high write | Scale, availability |
| **Key-Value** | Redis, DynamoDB | Caching, sessions | Speed, simplicity |
| **Graph** | Neo4j, Neptune | Relationships | Traversal performance |
| **Time-Series** | InfluxDB, TimescaleDB | Metrics, IoT | Time-based queries |
| **Search** | Elasticsearch | Full-text search | Text indexing |

### Database Scaling Strategies

#### Vertical Scaling (Scale Up)
```
┌─────────────────┐         ┌─────────────────┐
│   Small DB      │   ──▶   │   Larger DB     │
│   4 CPU         │         │   32 CPU        │
│   16 GB RAM     │         │   256 GB RAM    │
└─────────────────┘         └─────────────────┘
```
Limited by hardware ceiling.

#### Horizontal Scaling (Scale Out)

**Read Replicas:**
```
┌──────────────┐
│    Primary   │───Writes───▶
│   (Leader)   │
└──────┬───────┘
       │ Replication
       ▼
┌──────────────────────────────────────┐
│  ┌────────┐  ┌────────┐  ┌────────┐ │
│  │Replica1│  │Replica2│  │Replica3│ │
│  │ Read   │  │ Read   │  │ Read   │ │
│  └────────┘  └────────┘  └────────┘ │
└──────────────────────────────────────┘
```

**Sharding (Partitioning):**
```
┌────────────────────────────────────────────────────┐
│                  Shard Router                       │
└───────────┬──────────────┬──────────────┬──────────┘
            │              │              │
            ▼              ▼              ▼
       ┌─────────┐   ┌─────────┐    ┌─────────┐
       │ Shard 1 │   │ Shard 2 │    │ Shard 3 │
       │ A-H     │   │ I-P     │    │ Q-Z     │
       └─────────┘   └─────────┘    └─────────┘
```

### Sharding Strategies

| Strategy | How it Works | Pros | Cons |
|----------|--------------|------|------|
| **Range-based** | Shard by ranges (A-H, I-P) | Simple, range queries | Hot spots |
| **Hash-based** | Hash(key) % N shards | Even distribution | No range queries |
| **Directory-based** | Lookup table for mapping | Flexible | Lookup overhead |
| **Geographic** | Shard by region | Data locality | Cross-region queries |

---

## 3. Caching

### Cache Placement

```
┌────────┐     ┌─────────┐     ┌────────────┐     ┌──────────┐
│ Client │────▶│   CDN   │────▶│ App Server │────▶│ Database │
│        │     │ (Edge)  │     │  (Local)   │     │ (Remote) │
└────────┘     └─────────┘     └────────────┘     └──────────┘
                  ▲                  ▲                  ▲
                  │                  │                  │
              CDN Cache         Local Cache       Query Cache
            (static assets)    (in-memory)       (DB level)
```

### Caching Strategies

#### Cache-Aside (Lazy Loading)
```
Read:
1. Check cache
2. If miss, read from DB
3. Write to cache
4. Return data

Write:
1. Write to DB
2. Invalidate cache
```

```java
public Data get(String key) {
    Data data = cache.get(key);
    if (data == null) {
        data = db.get(key);
        cache.set(key, data);
    }
    return data;
}
```

#### Write-Through
```
Write:
1. Write to cache
2. Cache writes to DB (synchronous)
```

#### Write-Behind (Write-Back)
```
Write:
1. Write to cache
2. Cache writes to DB (asynchronous, batched)
```

#### Read-Through
```
Read:
1. Request from cache
2. Cache fetches from DB if miss (transparent)
```

### Cache Eviction Policies

| Policy | Description | Use Case |
|--------|-------------|----------|
| **LRU** | Least Recently Used | General purpose |
| **LFU** | Least Frequently Used | Frequency matters |
| **FIFO** | First In, First Out | Simple, predictable |
| **TTL** | Time To Live | Time-sensitive data |
| **Random** | Random eviction | Simple, low overhead |

### Cache Invalidation

The two hardest problems in CS: cache invalidation, naming things, and off-by-one errors.

**Strategies:**
- **TTL-based:** Set expiration time
- **Event-based:** Invalidate on write
- **Version-based:** Include version in key

### Distributed Caching with Consistent Hashing

```
        Node A                    Node B
           ●━━━━━━━━━━━━━━━━━━━━━━━━●
          ╱                          ╲
         ╱                            ╲
        ╱                              ╲
       ╱                                ╲
      ●                                  ●
   Node D                             Node C

Keys are hashed to positions on the ring.
Each node handles keys between it and the previous node.
Adding/removing nodes only affects adjacent keys.
```

### Real-World Products
- **In-memory:** Redis, Memcached
- **CDN:** CloudFlare, Akamai, CloudFront
- **Application:** Caffeine (Java), Guava Cache

---

## 4. Message Queues

### Why Message Queues?

```
Without Queue (Tight Coupling):
┌────────┐         ┌──────────┐
│Producer│────────▶│ Consumer │  (Synchronous, blocking)
└────────┘         └──────────┘

With Queue (Loose Coupling):
┌────────┐     ┌───────┐     ┌──────────┐
│Producer│────▶│ Queue │────▶│ Consumer │  (Async, buffered)
└────────┘     └───────┘     └──────────┘
```

**Benefits:**
- Decoupling
- Async processing
- Load leveling
- Durability
- Scalability

### Messaging Patterns

#### Point-to-Point (Queue)
```
Producers          Queue          Consumers
   P1 ─────┐    ┌────────┐    ┌───── C1
   P2 ─────┼───▶│ M1 M2  │───▶│
   P3 ─────┘    │ M3 M4  │    └───── C2
                └────────┘
Each message consumed by ONE consumer only
```

#### Pub/Sub (Topic)
```
Publishers         Topic         Subscribers
   P1 ─────┐    ┌────────┐    ┌───── S1 (gets all)
           ├───▶│ Topic  │───▶├───── S2 (gets all)
   P2 ─────┘    └────────┘    └───── S3 (gets all)

Each message delivered to ALL subscribers
```

### Delivery Guarantees

| Guarantee | Description | Use Case |
|-----------|-------------|----------|
| **At-most-once** | Fire and forget | Metrics, logs |
| **At-least-once** | Retry until ack | Most applications |
| **Exactly-once** | Dedup + idempotency | Financial transactions |

### Message Queue Comparison

| Feature | Kafka | RabbitMQ | SQS | Redis Streams |
|---------|-------|----------|-----|---------------|
| Pattern | Log-based | Traditional | Queue | Log-based |
| Ordering | Per partition | Per queue | FIFO option | Per stream |
| Throughput | Very high | High | High | High |
| Retention | Configurable | Until consumed | 14 days max | Configurable |
| Replay | Yes | No | No | Yes |

### Kafka Architecture

```
                    ┌─────────────────────────────────────────┐
                    │              Kafka Cluster              │
                    │                                         │
Producer ──────────▶│  Topic: orders                         │
                    │  ┌─────────────────────────────────┐   │
                    │  │ Partition 0: [M1][M2][M3]       │◀──┼── Consumer Group A
                    │  │ Partition 1: [M4][M5][M6]       │   │   (Consumer 1, 2)
                    │  │ Partition 2: [M7][M8][M9]       │   │
                    │  └─────────────────────────────────┘   │
                    │                                         │◀── Consumer Group B
                    └─────────────────────────────────────────┘    (Consumer 3)

Key concepts:
- Topics: Categories of messages
- Partitions: Ordered, immutable logs
- Offsets: Position in partition
- Consumer Groups: Share partitions for parallel processing
```

---

## 5. Content Delivery Networks (CDN)

### How CDN Works

```
                         ┌─────────────────┐
                         │   Origin Server │
                         │   (Your Server) │
                         └────────┬────────┘
                                  │
                    ┌─────────────┼─────────────┐
                    │             │             │
              ┌─────▼─────┐ ┌─────▼─────┐ ┌─────▼─────┐
              │ Edge PoP  │ │ Edge PoP  │ │ Edge PoP  │
              │ (US-East) │ │ (EU-West) │ │ (Asia)    │
              └─────┬─────┘ └─────┬─────┘ └─────┬─────┘
                    │             │             │
                 Users         Users         Users
              (New York)     (London)      (Tokyo)
```

### Push vs Pull CDN

| Push CDN | Pull CDN |
|----------|----------|
| Upload content proactively | Content fetched on first request |
| Good for static, known content | Good for dynamic content |
| Higher storage costs | Cache miss latency |
| More control | Less management |

### CDN Use Cases
- Static assets (images, CSS, JS)
- Video streaming
- Software downloads
- API acceleration
- DDoS protection

### Real-World Products
- CloudFlare, Akamai, Fastly
- AWS CloudFront, Azure CDN
- Google Cloud CDN

---

## 6. API Gateway

### Responsibilities

```
┌─────────────────────────────────────────────────────────────┐
│                       API Gateway                           │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │   Auth      │  │   Rate      │  │   Request   │         │
│  │   (JWT,     │  │   Limiting  │  │   Routing   │         │
│  │   OAuth)    │  │             │  │             │         │
│  └─────────────┘  └─────────────┘  └─────────────┘         │
│                                                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │   Request   │  │   Response  │  │   Logging/  │         │
│  │   Transform │  │   Transform │  │   Metrics   │         │
│  └─────────────┘  └─────────────┘  └─────────────┘         │
│                                                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │   Circuit   │  │   Load      │  │   Caching   │         │
│  │   Breaker   │  │   Balancing │  │             │         │
│  └─────────────┘  └─────────────┘  └─────────────┘         │
└─────────────────────────────────────────────────────────────┘
                              │
            ┌─────────────────┼─────────────────┐
            ▼                 ▼                 ▼
     ┌───────────┐     ┌───────────┐     ┌───────────┐
     │ Service A │     │ Service B │     │ Service C │
     └───────────┘     └───────────┘     └───────────┘
```

### Real-World Products
- Kong, Tyk, Apigee
- AWS API Gateway, Azure API Management
- NGINX, Envoy

---

## 7. Service Discovery

### Client-Side Discovery

```
┌────────┐     ┌─────────────────┐
│ Client │────▶│Service Registry │
└───┬────┘     │  (Consul/etcd)  │
    │          └─────────────────┘
    │                  │
    │          Get service instances
    │                  │
    │                  ▼
    │          ┌─────────────────┐
    │          │ Instance 1: IP1 │
    │          │ Instance 2: IP2 │
    │          │ Instance 3: IP3 │
    │          └─────────────────┘
    │
    └─────────────────────────────▶ Call IP2 directly
```

### Server-Side Discovery

```
┌────────┐     ┌───────────────┐     ┌─────────────────┐
│ Client │────▶│ Load Balancer │────▶│Service Registry │
└────────┘     └───────┬───────┘     └─────────────────┘
                       │
                       ▼
              Route to healthy instance
                       │
            ┌──────────┼──────────┐
            ▼          ▼          ▼
         ┌─────┐   ┌─────┐   ┌─────┐
         │ S1  │   │ S2  │   │ S3  │
         └─────┘   └─────┘   └─────┘
```

### Real-World Products
- Consul, etcd, ZooKeeper
- Kubernetes DNS/Service
- AWS Cloud Map

---

## 8. Proxies

### Forward Proxy
Client-side proxy. Hides client identity.

```
┌────────┐     ┌───────────────┐     ┌────────┐
│ Client │────▶│ Forward Proxy │────▶│ Server │
└────────┘     └───────────────┘     └────────┘
                    │
              - Anonymity
              - Access control
              - Caching
```

### Reverse Proxy
Server-side proxy. Hides server identity.

```
┌────────┐     ┌───────────────┐     ┌──────────────┐
│ Client │────▶│ Reverse Proxy │────▶│ Server Pool  │
└────────┘     └───────────────┘     └──────────────┘
                    │
              - Load balancing
              - SSL termination
              - Caching
              - Security
```

### Real-World Products
- NGINX, HAProxy, Envoy
- AWS ALB, CloudFlare

---

## 9. Storage Systems

### Block Storage
Raw storage blocks. Like virtual hard drives.

```
┌─────────────────────────────┐
│      Block Storage          │
│  ┌─────┐ ┌─────┐ ┌─────┐   │
│  │Block│ │Block│ │Block│   │
│  │ 1   │ │ 2   │ │ 3   │   │
│  └─────┘ └─────┘ └─────┘   │
└─────────────────────────────┘

Use: Databases, OS boot volumes
Examples: AWS EBS, Azure Disk
```

### File Storage
Hierarchical file systems.

```
┌─────────────────────────────┐
│      File Storage           │
│  /home/                     │
│  ├── user1/                 │
│  │   └── documents/         │
│  └── user2/                 │
│      └── photos/            │
└─────────────────────────────┘

Use: Shared file access
Examples: AWS EFS, NFS
```

### Object Storage
Flat namespace with metadata.

```
┌─────────────────────────────────────────┐
│           Object Storage                │
│  ┌───────────────────────────────────┐  │
│  │ Bucket: images                    │  │
│  │  ├── profile_123.jpg (+ metadata) │  │
│  │  ├── banner_456.png (+ metadata)  │  │
│  │  └── logo.svg (+ metadata)        │  │
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘

Use: Unstructured data, static assets
Examples: S3, GCS, Azure Blob
```

### Comparison

| Aspect | Block | File | Object |
|--------|-------|------|--------|
| Access | Block-level | File path | HTTP/REST |
| Performance | Highest | Medium | Good for large files |
| Scalability | Limited | Medium | Virtually unlimited |
| Cost | $$$$ | $$$ | $ |
| Use Case | Databases | Shared access | Static assets |

---

## 10. Coordination Services

### Use Cases
- Distributed locking
- Leader election
- Configuration management
- Service discovery

### ZooKeeper

```
┌─────────────────────────────────────────┐
│           ZooKeeper Ensemble            │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐   │
│  │ Server1 │ │ Server2 │ │ Server3 │   │
│  │(Leader) │ │(Follower)│ │(Follower)│  │
│  └─────────┘ └─────────┘ └─────────┘   │
└─────────────────────────────────────────┘

Data Model (ZNodes):
/
├── config/
│   └── database_url
├── locks/
│   └── resource_1
└── election/
    └── leader
```

### etcd

Key-value store with strong consistency (Raft).

```
PUT /config/database_url "postgres://..."
GET /config/database_url
WATCH /config/  (subscribe to changes)
```

### Real-World Products
- ZooKeeper, etcd, Consul
- Redis (with RedLock)
