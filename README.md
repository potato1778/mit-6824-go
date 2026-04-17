# MIT 6.824: Distributed Systems Implementation in Go

![Go](https://img.shields.io/badge/Language-Go-00ADD8?style=for-the-badge&logo=go)
![License](https://img.shields.io/badge/License-MIT-blue?style=for-the-badge)

## Overview

This repository documents my engineering implementation of the MIT 6.824 Distributed Systems labs, completed as part of the Distributed Systems course at the University of Gothenburg/Chalmers. The goal was to build a fault-tolerant, linearizable, and highly available distributed system using **Go**.

The implementation covers three core building blocks of modern cloud infrastructure:
1. **MapReduce** — Distributed data processing framework
2. **Raft** — Consensus algorithm for replicated state machines
3. **KV Service** — A fault-tolerant Key-Value store built on top of Raft

**What I built vs. what I relied on:**
- Raft protocol logic, leader election, log replication, and snapshotting: implemented myself
- RPC layer: Go standard library (`net/rpc`)
- Serialization: Go standard library (`encoding/gob`)
- Concurrency primitives: Go standard library (`sync.Mutex`, `sync.Cond`)
- Test harness: provided by the MIT 6.824 course staff

> ⚠️ **Academic Integrity Note:**
> To adhere to MIT's collaboration policy, the source code is kept in a private repository. This repository serves as a showcase of architecture design, implementation details, and learning notes. I am happy to walk through the code logic and demonstrate the system during an interview.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────┐
│                   Client                        │
│         (Put / Append / Get RPCs)               │
└───────────────────┬─────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────┐
│              KV Service Layer                   │
│   (Duplicate detection, snapshot management)    │
└───────────────────┬─────────────────────────────┘
                    │
                    ▼
┌──────────────────────────────────────────────────────────────────┐
│                        Raft Cluster                              │
│                                                                  │
│   ┌─────────┐      ┌─────────┐      ┌─────────┐                  │
│   │ Peer 0  │◄────►│ Peer 1  │◄────►│ Peer 2  │                  │
│   │(Leader) │      │(Follow.)│      │(Follow.)│                  │
│   └─────────┘      └─────────┘      └─────────┘                  │
│                                                                  │
│   Each peer: Log + Persistent State + State Machine              │
└──────────────────────────────────────────────────────────────────┘
```

**Request flow:** Client → KV Leader → Raft log replication across peers → Applied to KV state machine → Response to client

---

## Project Breakdown

### 1. MapReduce Framework (Lab 1)
Implemented a distributed MapReduce library with a **Coordinator** and multiple **Workers**.

**Key Features:**
- **Dynamic Task Scheduling:** The Coordinator assigns Map/Reduce tasks to idle workers and tracks task status (Idle, In-Progress, Completed).
- **Fault Tolerance:** Handled worker failures using a heartbeat mechanism. If a worker doesn't report back within 10 seconds, the task is re-assigned.
- **Straggler Handling:** Speculatively re-runs slow tasks to avoid long-tail latency.
- **Atomicity:** Used temporary files and atomic renaming (`os.Rename`) to ensure partial writes from crashed workers do not corrupt the final output.

---

### 2. Raft Consensus Algorithm (Lab 2) ⭐ *Core Project*
Implemented the Raft consensus protocol as described in the [extended Raft paper](https://raft.github.io/raft.pdf).

**Architecture Design:**
- **Leader Election:** Randomized election timeouts (300–600ms) to prevent split votes. Used `time.Ticker` for heartbeat generation and election triggers.
- **Log Replication:** `AppendEntries` RPC with an optimized conflict back-off strategy — on conflict, the follower returns the conflict term and index, allowing the leader to skip multiple entries per RPC instead of one at a time.
- **Persistence:** `gob` serialization of `currentTerm`, `votedFor`, and `log` to stable storage to survive crashes.

**Go Concurrency Design:**
- **Locking Granularity:** `sync.Mutex` protects shared state. Locks are always released before blocking RPC calls to prevent deadlocks.
- **Event Loop:** Each Raft peer runs a single main event loop handling incoming RPCs, timer management, and log application — keeping state transitions predictable.

---

### 3. Fault-Tolerant Key-Value Store (Lab 3)
A linearizable KV storage service built on top of the Raft layer.

**Key Features:**
- **Client Request Handling:** Clients send `Put`, `Append`, and `Get` RPCs to the Leader. On Leader failure, clients retry until a new Leader is elected.
- **Duplicate Detection (Idempotency):** Each client carries a unique `ClientId` and monotonically increasing `RequestId`. The state machine tracks the last executed request per client to filter duplicates — handling the case where a leader commits a log entry but crashes before replying.
- **Snapshotting (Log Compaction):** The service snapshots the KV map when the Raft log exceeds a size threshold. Implemented `InstallSnapshot` RPC to bring lagging followers up to date.

---

## Testing & Verification

Tested using the MIT 6.824 test suite, which simulates harsh network conditions:

- **Chaos Testing:** Disconnecting nodes, network partitions, crashing leaders at random.
- **Race Detection:** All tests passed with `go test -race`.
- **Performance:** Passed `TestSpeed` (latency bounds) and `TestUnreliable` (packet loss and reordering).

---

## Key Challenges & Learnings

### 1. Split Vote Liveness
Early on, nodes would endlessly split votes during network partitions, causing liveness failures.

**Fix:** Widened and randomized the election timeout range so one node is statistically guaranteed to time out first. This eliminated the split vote loops without any changes to the core election logic.

### 2. Channels vs. Mutexes — A Design Pivot

I initially modelled inter-goroutine communication using Go channels, which felt idiomatic. However, as the number of goroutines grew (one per peer, one for log application, one for snapshotting), I ran into circular dependencies where goroutines would block waiting on each other's channels, causing deadlocks that were difficult to reason about.

**Refactor:** Switched to shared-memory design using `sync.Mutex` for state protection and `sync.Cond` for signalling state changes (e.g., when `commitIndex` advances). The trade-off: this approach is less "Go-idiomatic" in style but makes state transitions much easier to reason about formally — which matters a lot for a correctness-critical algorithm like Raft. There's likely a performance ceiling compared to a well-designed channel-based model, but for this implementation, correctness was the priority.

### 3. Commit vs. Apply — Linearizability
A subtle but critical correctness requirement: the response to a client must only be sent after a log entry is **applied to the state machine**, not just when Raft marks it as committed. Getting this ordering wrong breaks linearizability even when Raft itself is correct.

---

## What I'd Do Differently

If I were starting over today, I would define the interface boundaries between the Raft core and the KV state machine layer upfront, before writing any implementation code. I underestimated how tightly coupled those two layers would become and ended up doing a significant refactor mid-way through when I hit abstraction issues. Clear interfaces from day one would have made the snapshotting and log compaction work much cleaner to integrate.

---

## References
- [In Search of an Understandable Consensus Algorithm (Extended Version)](https://raft.github.io/raft.pdf)
- [MIT 6.824 Schedule & Materials](https://pdos.csail.mit.edu/6.824/)
- [A Tour of Go](https://tour.golang.org/)

---

## Contact

Feel free to reach out to discuss Distributed Systems, Go, or Infrastructure engineering.

**Ruize Liu**
- Email: ruizeliu.heu@gmail.com
- LinkedIn: https://www.linkedin.com/in/ruize-liu/
