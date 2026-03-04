# Design a Message Queue (Kafka)

A distributed messaging system for asynchronous communication.

---

## 1. Requirements

### Functional Requirements
- Producers publish messages to topics
- Consumers subscribe to topics and receive messages
- Message ordering (at least within a partition)
- Message persistence (configurable retention)
- Consumer groups for parallel processing

### Non-Functional Requirements
- High throughput (millions of messages/sec)
- Low latency (< 10ms)
- Durability (no message loss)
- High availability
- Horizontal scalability

---

## 2. Core Concepts

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Message Queue Concepts                          │
│                                                                     │
│   Producer: Sends messages                                         │
│   Consumer: Receives messages                                      │
│   Topic: Category/feed name for messages                           │
│   Partition: Ordered, immutable sequence of messages               │
│   Offset: Position of message in partition                         │
│   Consumer Group: Set of consumers that share workload             │
│                                                                     │
│   Topic: "orders"                                                  │
│   ┌─────────────────────────────────────────────────────────────┐  │
│   │ Partition 0: [M0][M1][M2][M3][M4][M5]...                   │  │
│   │              offset→                                        │  │
│   │                                                             │  │
│   │ Partition 1: [M0][M1][M2][M3]...                           │  │
│   │                                                             │  │
│   │ Partition 2: [M0][M1][M2][M3][M4][M5][M6]...               │  │
│   └─────────────────────────────────────────────────────────────┘  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 3. High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│     Producers                                                      │
│     ┌─────┐ ┌─────┐ ┌─────┐                                       │
│     │ P1  │ │ P2  │ │ P3  │                                       │
│     └──┬──┘ └──┬──┘ └──┬──┘                                       │
│        │       │       │                                           │
│        └───────┼───────┘                                           │
│                │                                                    │
│                ▼                                                    │
│   ┌─────────────────────────────────────────────────────────────┐  │
│   │                    Kafka Cluster                             │  │
│   │                                                              │  │
│   │   ┌─────────────┐ ┌─────────────┐ ┌─────────────┐          │  │
│   │   │  Broker 1   │ │  Broker 2   │ │  Broker 3   │          │  │
│   │   │             │ │             │ │             │          │  │
│   │   │ Topic A     │ │ Topic A     │ │ Topic A     │          │  │
│   │   │ Partition 0 │ │ Partition 1 │ │ Partition 2 │          │  │
│   │   │ (Leader)    │ │ (Leader)    │ │ (Leader)    │          │  │
│   │   │             │ │             │ │             │          │  │
│   │   │ Topic A     │ │ Topic A     │ │ Topic A     │          │  │
│   │   │ Partition 1 │ │ Partition 2 │ │ Partition 0 │          │  │
│   │   │ (Replica)   │ │ (Replica)   │ │ (Replica)   │          │  │
│   │   └─────────────┘ └─────────────┘ └─────────────┘          │  │
│   │                                                              │  │
│   └─────────────────────────────────────────────────────────────┘  │
│                │                                                    │
│                ▼                                                    │
│     ┌──────────┴──────────┐                                       │
│     │                     │                                        │
│     ▼                     ▼                                        │
│   Consumer Group A    Consumer Group B                             │
│   ┌─────┐ ┌─────┐    ┌─────┐                                      │
│   │ C1  │ │ C2  │    │ C3  │                                      │
│   └─────┘ └─────┘    └─────┘                                      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 4. Broker Design

### Message Storage (Log-Based)

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Partition Storage                                │
│                                                                     │
│   Partition = Append-only log (immutable)                          │
│                                                                     │
│   topic-orders/partition-0/                                        │
│   ├── 00000000000000000000.log   (Segment 0: offsets 0-1000)      │
│   ├── 00000000000000000000.index                                   │
│   ├── 00000000000000001001.log   (Segment 1: offsets 1001-2000)   │
│   ├── 00000000000000001001.index                                   │
│   └── 00000000000000002001.log   (Active segment)                 │
│                                                                     │
│   Segment File (.log):                                             │
│   ┌─────────────────────────────────────────────────────────────┐  │
│   │ [Msg1][Msg2][Msg3][Msg4][Msg5]...                           │  │
│   └─────────────────────────────────────────────────────────────┘  │
│                                                                     │
│   Index File (.index):                                             │
│   ┌────────────────────────────────────────┐                       │
│   │ Offset │ Position in log file          │                       │
│   │ 0      │ 0                             │                       │
│   │ 100    │ 15234                         │                       │
│   │ 200    │ 31456                         │                       │
│   └────────────────────────────────────────┘                       │
│                                                                     │
│   Sparse index: Not every offset, sampled                          │
│   Binary search to find approximate position                       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Message Format

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Message Format                                  │
│                                                                     │
│   ┌────────────────────────────────────────────────────────────┐   │
│   │ Offset        │ 8 bytes  │ Sequential ID                  │   │
│   │ Message Size  │ 4 bytes  │ Length of message              │   │
│   │ CRC32         │ 4 bytes  │ Checksum for integrity         │   │
│   │ Magic Byte    │ 1 byte   │ Version                        │   │
│   │ Attributes    │ 1 byte   │ Compression codec, etc.        │   │
│   │ Timestamp     │ 8 bytes  │ Create/append time             │   │
│   │ Key Length    │ 4 bytes  │ -1 if null                     │   │
│   │ Key           │ Variable │ Partition routing              │   │
│   │ Value Length  │ 4 bytes  │                                │   │
│   │ Value         │ Variable │ Message payload                │   │
│   │ Headers       │ Variable │ Key-value metadata             │   │
│   └────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 5. Producer Design

### Partitioning Strategy

```python
class Producer:
    def send(self, topic, key, value):
        # Determine partition
        if key is not None:
            # Hash key to get consistent partition
            partition = hash(key) % num_partitions
        else:
            # Round-robin for null keys
            partition = self.round_robin_counter++ % num_partitions

        # Get partition leader
        leader = self.metadata.get_leader(topic, partition)

        # Send to leader
        self.send_to_broker(leader, topic, partition, key, value)
```

### Batching and Compression

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Producer Batching                                │
│                                                                     │
│   Accumulator (per partition):                                     │
│   ┌──────────────────────────────────────────────────────────┐     │
│   │ Partition 0: [M1][M2][M3]        batch.size or linger.ms │     │
│   │ Partition 1: [M4][M5]                                    │     │
│   │ Partition 2: [M6][M7][M8][M9]                           │     │
│   └──────────────────────────────────────────────────────────┘     │
│                                                                     │
│   Send batch when:                                                 │
│   1. Batch size reached (batch.size = 16KB default)               │
│   2. Time elapsed (linger.ms = 0ms default)                       │
│                                                                     │
│   Compression:                                                     │
│   - Compress entire batch (not individual messages)               │
│   - Options: none, gzip, snappy, lz4, zstd                        │
│   - Trade-off: CPU vs network bandwidth                           │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Acknowledgment Modes

```
acks=0: Fire and forget
  Producer ────▶ Broker
  (No response waited)
  Fastest, but may lose messages

acks=1: Leader acknowledgment
  Producer ────▶ Leader Broker
  Producer ◀──── ACK
  Leader writes to local log before ACK
  May lose if leader fails before replication

acks=all (or -1): All replicas
  Producer ────▶ Leader Broker ────▶ Replicas
  Producer ◀──── ACK ◀──── All in-sync replicas committed
  Slowest, but no data loss
```

---

## 6. Consumer Design

### Consumer Groups

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Consumer Groups                                  │
│                                                                     │
│   Topic: orders (3 partitions)                                     │
│                                                                     │
│   Consumer Group A (3 consumers):                                  │
│   ┌─────────────────────────────────────────────────────────────┐  │
│   │ Consumer 1 ◀─── Partition 0                                 │  │
│   │ Consumer 2 ◀─── Partition 1                                 │  │
│   │ Consumer 3 ◀─── Partition 2                                 │  │
│   └─────────────────────────────────────────────────────────────┘  │
│   Each partition assigned to exactly one consumer in group         │
│                                                                     │
│   Consumer Group B (2 consumers):                                  │
│   ┌─────────────────────────────────────────────────────────────┐  │
│   │ Consumer 1 ◀─── Partition 0, Partition 1                    │  │
│   │ Consumer 2 ◀─── Partition 2                                 │  │
│   └─────────────────────────────────────────────────────────────┘  │
│   One consumer handles multiple partitions                         │
│                                                                     │
│   Consumer Group C (4 consumers):                                  │
│   ┌─────────────────────────────────────────────────────────────┐  │
│   │ Consumer 1 ◀─── Partition 0                                 │  │
│   │ Consumer 2 ◀─── Partition 1                                 │  │
│   │ Consumer 3 ◀─── Partition 2                                 │  │
│   │ Consumer 4 ◀─── (idle)                                      │  │
│   └─────────────────────────────────────────────────────────────┘  │
│   Extra consumers sit idle (max = num partitions)                  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Offset Management

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Offset Tracking                                  │
│                                                                     │
│   Partition: [M0][M1][M2][M3][M4][M5][M6][M7][M8][M9]              │
│                              ▲                ▲                     │
│                              │                │                     │
│                      Committed Offset    Current Position           │
│                          (offset 4)         (offset 7)              │
│                                                                     │
│   Committed Offset: Last processed and acknowledged                │
│   Current Position: Where consumer is reading                      │
│                                                                     │
│   Offset storage: __consumer_offsets topic (Kafka internal)        │
│                                                                     │
│   Commit strategies:                                               │
│   1. Auto-commit (enable.auto.commit=true)                        │
│      - Periodic commit (auto.commit.interval.ms)                  │
│      - Risk: at-least-once (may reprocess on failure)            │
│                                                                     │
│   2. Manual commit                                                 │
│      - commitSync(): Block until confirmed                        │
│      - commitAsync(): Non-blocking                                │
│      - More control, can achieve exactly-once                     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 7. Replication

### Leader-Follower Replication

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Partition Replication                            │
│                                                                     │
│   Topic: orders, Partition 0, Replication Factor: 3                │
│                                                                     │
│   ┌────────────┐      ┌────────────┐      ┌────────────┐          │
│   │  Broker 1  │      │  Broker 2  │      │  Broker 3  │          │
│   │            │      │            │      │            │          │
│   │  Partition │      │  Partition │      │  Partition │          │
│   │  0 LEADER  │─────▶│  0 Replica │      │  0 Replica │          │
│   │            │      │            │      │            │          │
│   └────────────┘      └────────────┘      └────────────┘          │
│         │                   ▲                   ▲                  │
│         │                   │                   │                  │
│         └───────────────────┴───────────────────┘                  │
│                    Replication                                      │
│                                                                     │
│   Writes: Go to Leader only                                        │
│   Reads: From Leader (or any replica if configured)               │
│   Replicas: Pull from leader                                       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### In-Sync Replicas (ISR)

```
ISR: Set of replicas that are "caught up" with leader

┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│   Leader: [M0][M1][M2][M3][M4][M5][M6]                             │
│                                       ▲ Latest                     │
│                                                                     │
│   Replica 1: [M0][M1][M2][M3][M4][M5][M6]  ✓ In sync              │
│   Replica 2: [M0][M1][M2][M3][M4][M5]      ✓ In sync (within lag)  │
│   Replica 3: [M0][M1][M2]                  ✗ Out of sync          │
│                                                                     │
│   ISR = {Leader, Replica 1, Replica 2}                             │
│                                                                     │
│   Replica removed from ISR if:                                     │
│   - Falls behind by replica.lag.time.max.ms (10 sec default)      │
│                                                                     │
│   Message committed when all ISR replicas have it                  │
│   High watermark: Last committed offset                            │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 8. Exactly-Once Semantics

### Idempotent Producer

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Idempotent Producer                              │
│                                                                     │
│   Problem: Network failure → Retry → Duplicate messages            │
│                                                                     │
│   Solution: Producer ID + Sequence Number                          │
│                                                                     │
│   Producer sends:                                                  │
│   { producer_id: 1234, sequence: 5, message: "order-123" }         │
│                                                                     │
│   Broker tracks:                                                   │
│   producer_id: 1234 → last_sequence: 5                             │
│                                                                     │
│   If retry sends same sequence:                                    │
│   - Broker detects duplicate                                       │
│   - Returns success without writing again                          │
│                                                                     │
│   enable.idempotence=true                                          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Transactions

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Kafka Transactions                               │
│                                                                     │
│   Atomic writes across multiple partitions/topics                  │
│                                                                     │
│   producer.beginTransaction()                                      │
│   producer.send(topic1, message1)                                  │
│   producer.send(topic2, message2)                                  │
│   producer.sendOffsetsToTransaction(offsets, consumerGroup)       │
│   producer.commitTransaction()   // All or nothing                │
│                                                                     │
│   Use case: Read-process-write pattern                             │
│                                                                     │
│   Read from topic A                                                │
│   ──▶ Process                                                      │
│   ──▶ Write to topic B                                             │
│   ──▶ Commit offset for A                                          │
│                                                                     │
│   All three happen atomically                                      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 9. ZooKeeper vs KRaft

### Traditional (with ZooKeeper)

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│   ┌─────────────────────────────────────────────────────────────┐  │
│   │                    ZooKeeper Cluster                        │  │
│   │   Stores: Broker list, topic config, partition assignment  │  │
│   │           Controller election, ACLs                        │  │
│   └─────────────────────────────────────────────────────────────┘  │
│                              │                                      │
│                              ▼                                      │
│   ┌─────────────┐ ┌─────────────┐ ┌─────────────┐                 │
│   │  Broker 1   │ │  Broker 2   │ │  Broker 3   │                 │
│   │ (Controller)│ │             │ │             │                 │
│   └─────────────┘ └─────────────┘ └─────────────┘                 │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### KRaft Mode (ZooKeeper-less)

```
┌─────────────────────────────────────────────────────────────────────┐
│                    KRaft Mode                                       │
│                                                                     │
│   Metadata stored in Kafka itself (__cluster_metadata topic)       │
│   Uses Raft consensus for controller election                      │
│                                                                     │
│   ┌─────────────┐ ┌─────────────┐ ┌─────────────┐                 │
│   │  Broker 1   │ │  Broker 2   │ │  Broker 3   │                 │
│   │ Controller  │ │ Controller  │ │ Controller  │                 │
│   │  (Active)   │ │  (Standby)  │ │  (Standby)  │                 │
│   └─────────────┘ └─────────────┘ └─────────────┘                 │
│                                                                     │
│   Benefits:                                                        │
│   - Simpler operations (no ZK cluster)                            │
│   - Faster failover                                                │
│   - Better scalability                                             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 10. Performance Optimizations

### Zero-Copy

```
Traditional:
Disk → Kernel Buffer → User Buffer → Socket Buffer → NIC

Zero-copy (sendfile):
Disk → Kernel Buffer ──────────────────────────▶ NIC

Eliminates copying to user space
Significant throughput improvement
```

### Page Cache

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Page Cache Usage                                 │
│                                                                     │
│   Kafka relies on OS page cache instead of JVM heap               │
│                                                                     │
│   Write: Append to page cache → Async flush to disk               │
│   Read: Page cache hit (recent data) → No disk I/O                │
│                                                                     │
│   Benefits:                                                        │
│   - No GC pressure                                                 │
│   - Survives broker restart                                        │
│   - OS manages cache efficiently                                   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 11. Delivery Guarantees Summary

| Guarantee | Producer Config | Consumer Config |
|-----------|-----------------|-----------------|
| At-most-once | acks=0 | Auto-commit before process |
| At-least-once | acks=all, retries>0 | Commit after process |
| Exactly-once | Idempotent + transactions | Transactional consumer |

---

## 12. Interview Tips

**Key Topics:**
1. Log-based storage model
2. Partitioning and ordering guarantees
3. Consumer groups and rebalancing
4. Replication and ISR
5. Exactly-once semantics
6. ZooKeeper role (or KRaft)

**Questions to Ask:**
- What's the message throughput requirement?
- Ordering requirements (global vs per-key)?
- Retention period?
- Delivery guarantee needed?
- Message size?
