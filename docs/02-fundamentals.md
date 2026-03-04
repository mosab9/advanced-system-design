# System Design Fundamentals

Core concepts every senior engineer must understand for system design interviews.

---

## 1. CAP Theorem

The CAP theorem states that a distributed system can only guarantee **two out of three** properties:

```
        Consistency
            /\
           /  \
          /    \
         /      \
        /   CA   \
       /          \
      /____________\
     CP            AP
Partition     Partition
Tolerance     Tolerance
     \            /
      \    AP   /
       \      /
        \    /
         \  /
          \/
     Availability
```

### The Three Properties

| Property | Definition | Example |
|----------|------------|---------|
| **Consistency** | All nodes see the same data at the same time | After a write, all reads return that value |
| **Availability** | Every request receives a response | System always responds, even with stale data |
| **Partition Tolerance** | System works despite network failures | System operates even if nodes can't communicate |

### Why Only Two?

During a **network partition**:
- If you want **Consistency**: Must reject some requests (lose Availability)
- If you want **Availability**: Must serve potentially stale data (lose Consistency)
- **Partition Tolerance** is not optional in distributed systems

### Real-World Trade-offs

| System Type | Choice | Examples |
|-------------|--------|----------|
| **CP** | Consistency + Partition Tolerance | MongoDB, HBase, Redis (cluster mode) |
| **AP** | Availability + Partition Tolerance | Cassandra, DynamoDB, CouchDB |
| **CA** | Consistency + Availability | Single-node RDBMS (not distributed) |

### PACELC Extension

CAP only addresses partition scenarios. PACELC adds:
- **P**artition: Choose **A**vailability or **C**onsistency
- **E**lse (no partition): Choose **L**atency or **C**onsistency

```
if (partition) {
    choose(Availability, Consistency);
} else {
    choose(Latency, Consistency);
}
```

---

## 2. ACID Properties (Transactions)

Properties that guarantee reliable database transactions:

### Atomicity
All operations in a transaction succeed or all fail. No partial updates.

```
BEGIN TRANSACTION;
  UPDATE accounts SET balance = balance - 100 WHERE id = 'A';
  UPDATE accounts SET balance = balance + 100 WHERE id = 'B';
COMMIT;
-- Either both happen, or neither happens
```

### Consistency
Database moves from one valid state to another. All constraints are maintained.

```
-- Constraint: balance >= 0
-- If A has $50, transferring $100 violates constraint
-- Transaction is rejected, database stays consistent
```

### Isolation
Concurrent transactions don't interfere with each other.

**Isolation Levels:**
| Level | Dirty Read | Non-Repeatable Read | Phantom Read |
|-------|------------|---------------------|--------------|
| Read Uncommitted | Yes | Yes | Yes |
| Read Committed | No | Yes | Yes |
| Repeatable Read | No | No | Yes |
| Serializable | No | No | No |

### Durability
Once committed, data persists even if system fails.

```
COMMIT;
-- Data is now in Write-Ahead Log (WAL)
-- Safe even if power fails immediately after
```

---

## 3. BASE Properties

Alternative to ACID for distributed systems (eventual consistency):

| Property | Meaning |
|----------|---------|
| **B**asically **A**vailable | System guarantees availability |
| **S**oft State | State may change over time without input |
| **E**ventual Consistency | System will become consistent eventually |

### ACID vs BASE

| ACID | BASE |
|------|------|
| Strong consistency | Eventual consistency |
| Pessimistic | Optimistic |
| Complex transactions | Simple operations |
| Vertical scaling | Horizontal scaling |
| Lower availability | Higher availability |

---

## 4. Consistency Models

### Strong Consistency
All reads see the most recent write. Linearizable.

```
Time ─────────────────────────────────────────▶

Client A:  ──Write(X=5)──
                         │
Client B:        ────────┼──Read(X)──▶ Returns 5 ✓
                         │
                   Synchronization point
```

**Implementations:**
- Synchronous replication
- Consensus protocols (Raft, Paxos)
- Two-phase commit

### Eventual Consistency
Given enough time without updates, all replicas converge.

```
Time ─────────────────────────────────────────▶

Client A:  ──Write(X=5)──
                │
Client B:       │──Read(X)──▶ Returns 3 (stale)
                │
                └──────────────────────┐
                                       │
Client B:                        Read(X)──▶ Returns 5 ✓
                                       │
                              Eventually consistent
```

**Guarantee:** If no new updates, eventually all reads return last written value.

### Causal Consistency
If A causes B, everyone sees A before B.

```
User A: Post("Hello")          → Message ID: 1
User B: Reply(1, "Hi there!")  → Message ID: 2

All users see Message 1 before Message 2 ✓
```

### Read-Your-Writes Consistency
After a write, that client always sees their write.

```
Client A:  Write(X=5) ──────── Read(X) ──▶ Returns 5 ✓
                               │
Client B:                      Read(X) ──▶ Might return old value
```

### Monotonic Reads
Once you read a value, you never see an older value.

```
Client A:  Read(X) → 5 ──────── Read(X) → 5 or higher ✓
                               Never returns 3 (older)
```

---

## 5. Replication Strategies

### Single Leader (Master-Slave)

```
┌────────────────────────────────────┐
│              Leader                │
│  (Handles all writes)              │
└────────────────────────────────────┘
         │           │           │
         ▼           ▼           ▼
    ┌─────────┐ ┌─────────┐ ┌─────────┐
    │Follower │ │Follower │ │Follower │
    │(Replica)│ │(Replica)│ │(Replica)│
    │ Read    │ │ Read    │ │ Read    │
    └─────────┘ └─────────┘ └─────────┘
```

**Pros:** Simple, no write conflicts
**Cons:** Leader is bottleneck and single point of failure

### Multi-Leader

```
┌─────────────┐           ┌─────────────┐
│   Leader 1  │◄─────────▶│   Leader 2  │
│ (DC: US)    │  Replicate│ (DC: EU)    │
└─────────────┘           └─────────────┘
      │                         │
      ▼                         ▼
 ┌─────────┐               ┌─────────┐
 │Followers│               │Followers│
 └─────────┘               └─────────┘
```

**Pros:** Better latency, no single point of failure
**Cons:** Write conflicts need resolution

### Leaderless (Peer-to-Peer)

```
    ┌─────────┐
    │ Node 1  │
    └────┬────┘
         │
    ┌────┴────┐
    │         │
┌───▼───┐ ┌───▼───┐
│Node 2 │◄┼▶│Node 3│
└───────┘ │ └───────┘
          │
      ┌───▼───┐
      │Node 4 │
      └───────┘
```

**Uses quorum:** W + R > N for consistency
- N = total nodes
- W = write quorum
- R = read quorum

**Example:** N=3, W=2, R=2
- Write succeeds when 2 nodes acknowledge
- Read queries 2 nodes, uses latest version

---

## 6. Consensus Algorithms

### Raft

Leader-based consensus for distributed systems.

**Key Concepts:**
1. **Leader Election:** One node is leader at a time
2. **Log Replication:** Leader replicates log to followers
3. **Safety:** If a log entry is committed, it appears in all future leaders' logs

```
Term 1:   [Leader: A] ─────────────────────────────────▶
                                │
                         A fails │
                                │
Term 2:   ──────────────────────┼─[Leader: B]─────────▶
                                │
                   Election     │
```

**Used by:** etcd, Consul, CockroachDB

### Paxos

Original consensus algorithm. More complex than Raft.

**Phases:**
1. **Prepare:** Proposer sends prepare request
2. **Promise:** Acceptors promise not to accept older proposals
3. **Accept:** Proposer sends accept request
4. **Accepted:** Acceptors acknowledge

---

## 7. Distributed Transactions

### Two-Phase Commit (2PC)

```
                    Coordinator
                        │
        ┌───────────────┼───────────────┐
        │               │               │
        ▼               ▼               ▼
    ┌───────┐       ┌───────┐       ┌───────┐
    │Node A │       │Node B │       │Node C │
    └───────┘       └───────┘       └───────┘

Phase 1 (Prepare):
Coordinator → All: "Prepare to commit"
All → Coordinator: "Ready" or "Abort"

Phase 2 (Commit):
If all ready:
    Coordinator → All: "Commit"
Else:
    Coordinator → All: "Abort"
```

**Problems:** Blocking if coordinator fails after prepare

### Saga Pattern

Alternative to 2PC using compensating transactions:

```
T1 → T2 → T3 → T4
        ↓ (if T3 fails)
       C2 ← C1
   (compensate)
```

**Example: Order Saga**
```
1. Create Order      → Compensate: Cancel Order
2. Reserve Inventory → Compensate: Release Inventory
3. Charge Payment    → Compensate: Refund Payment
4. Ship Order        → Compensate: Cancel Shipment
```

**Types:**
- **Choreography:** Events trigger next step
- **Orchestration:** Central coordinator manages steps

---

## 8. Time in Distributed Systems

### The Problem
No global clock. Each node has its own clock with drift.

### Vector Clocks
Track causality, not wall-clock time.

```
Initial:      A[0,0,0]  B[0,0,0]  C[0,0,0]

A sends to B: A[1,0,0] ────────▶ B[1,1,0]

B sends to C:                    B[1,2,0] ───▶ C[1,2,1]

Concurrent:   A[2,0,0]           C[1,2,2]
              (A doesn't know about B's update to C)
```

### Lamport Timestamps
Simpler than vector clocks, but only provides partial ordering.

```
Rules:
1. Before event: increment local counter
2. On send: attach timestamp
3. On receive: max(local, received) + 1
```

### Hybrid Logical Clocks (HLC)
Combines physical time with logical time.

```
HLC = (physical_time, logical_counter)

Benefits:
- Bounded drift from real time
- Captures causality
- Used by CockroachDB, MongoDB
```

---

## 9. Failure Modes

### Fail-Stop
Node stops and never recovers.

### Fail-Recover
Node stops but may recover later with state intact.

### Byzantine Failures
Node may behave arbitrarily (including maliciously).

```
Types of failures (increasing difficulty):
1. Crash failures (easiest to handle)
2. Omission failures (messages lost)
3. Timing failures (responses too slow)
4. Byzantine failures (arbitrary behavior)
```

### Handling Failures

**Detection:**
- Heartbeats
- Timeout-based detection
- Phi Accrual failure detector

**Recovery:**
- Replication
- Checkpointing
- Write-Ahead Logging (WAL)

---

## 10. Key Theorems & Laws

### Amdahl's Law
Speedup limited by sequential portion:

```
Speedup = 1 / (S + P/N)

Where:
- S = serial fraction
- P = parallel fraction (1 - S)
- N = number of processors
```

### Universal Scalability Law
Accounts for contention and coherence:

```
C(N) = N / (1 + α(N-1) + βN(N-1))

Where:
- α = contention parameter
- β = coherence parameter
- N = number of nodes
```

### Little's Law
Relates throughput, latency, and concurrency:

```
L = λW

Where:
- L = average number in system (concurrency)
- λ = arrival rate (throughput)
- W = average time in system (latency)
```

### The Eight Fallacies of Distributed Computing
1. The network is reliable
2. Latency is zero
3. Bandwidth is infinite
4. The network is secure
5. Topology doesn't change
6. There is one administrator
7. Transport cost is zero
8. The network is homogeneous
