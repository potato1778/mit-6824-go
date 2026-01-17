# 🚀 MIT 6.824: Distributed Systems Implementation in Go

![Go](https://img.shields.io/badge/Language-Go-00ADD8?style=for-the-badge&logo=go)
![Build](https://img.shields.io/badge/Build-Passing-success?style=for-the-badge)
![License](https://img.shields.io/badge/License-MIT-blue?style=for-the-badge)

## 📖 Overview

This repository documents my journey and engineering implementation of the **MIT 6.824 Distributed Systems** course labs. The primary goal was to build a fault-tolerant, linearizable, and highly available distributed system from scratch using **Go**.

The implementation covers the core building blocks of modern cloud infrastructure:
1.  **MapReduce:** Distributed data processing.
2.  **Raft:** Consensus algorithm for replicated state machines.
3.  **KV Service:** A fault-tolerant Key-Value store built on top of Raft.

> ⚠️ **Academic Integrity Note:**
> To adhere to MIT's collaboration policy, **the source code is kept in a private repository**. This repository serves as a showcase of the architecture design, implementation details, and learning notes. I am happy to walk through the code logic and demonstrate the system during an interview.

---

## 🛠️ Project Breakdown

### 1. MapReduce Framework (Lab 1)
Implemented a distributed MapReduce library consisting of a **Coordinator (Master)** and multiple **Workers**.

#### Key Features:
* **Dynamic Task Scheduling:** The Coordinator assigns Map/Reduce tasks to idle workers and tracks task status (Idle, In-Progress, Completed).
* **Fault Tolerance:** Handled worker failures using a **heartbeat mechanism**. If a worker doesn't report back within 10 seconds, the task is re-assigned to another worker.
* **Straggler Handling:** Implemented logic to handle "slow" workers by speculatively re-running slow tasks.
* **Atomicity:** Used temporary files and atomic renaming (`os.Rename`) to ensure that partial writes from crashed workers do not corrupt the final output.

---

### 2. Raft Consensus Algorithm (Lab 2) ⭐ *Core Project*
Implemented the Raft consensus protocol as described in the [extended Raft paper](https://raft.github.io/raft.pdf). This serves as the foundation for the fault-tolerant KV store.

#### 🏗️ Architecture Design
* **Leader Election:**
    * Implemented randomized election timeouts (e.g., 300-600ms) to prevent split votes.
    * Used `time.Ticker` for heartbeat generation and election triggers.
* **Log Replication:**
    * Designed the `AppendEntries` RPC to handle log consistency checks.
    * Implemented an optimized "back-off" strategy: when a conflict occurs, the follower returns the conflict term and index, allowing the leader to skip multiple conflicting entries in one RPC (instead of one by one).
* **Persistence:**
    * Used `gob` to serialize state (`currentTerm`, `votedFor`, `log`) to stable storage to survive server crashes.

#### 🧩 Technical Highlights (Go Concurrency)
* **Locking Granularity:** Used `sync.Mutex` extensively to protect shared state. Learned to avoid holding locks during blocking I/O (RPC calls) to prevent deadlocks.
* **Event Loop:** Each Raft peer runs a main event loop that handles incoming RPCs, applies committed entries to the state machine, and manages timers.

---

### 3. Fault-Tolerant Key-Value Store (Lab 3)
Built a linearizable KV storage service on top of the Raft layer.

#### Key Features:
* **Client Request Handling:** Clients send `Put`, `Append`, and `Get` RPCs to the Leader. If the Leader crashes, the client retries until a new Leader is found.
* **Duplicate Detection (Idempotency):**
    * **Challenge:** If a leader commits a log but crashes before replying, the client will retry. This could lead to applying the same operation twice.
    * **Solution:** Each client has a unique `ClientId` and a monotonically increasing `RequestId`. The state machine tracks the last executed request for each client to filter duplicates.
* **Snapshotting (Log Compaction):**
    * To prevent the Raft log from growing infinitely, the service creates snapshots of the KV map when the log size exceeds a threshold.
    * Implemented `InstallSnapshot` RPC to send snapshots to lagging followers who have fallen too far behind the leader's log.

---

## 🧪 Testing & Verification

The system was rigorously tested using the course's provided test suite, which simulates harsh network conditions.

* **Chaos Testing:** Tests involve disconnecting nodes, partitioning the network, and crashing leaders randomly.
* **Race Detection:** All tests passed with the Go race detector enabled (`go test -race`).
* **Performance:**
    * Passed `TestSpeed` requirements (committing operations within latency limits).
    * Passed `TestUnreliable` (network packet loss/reordering).

*(Optional: Insert a screenshot of your terminal showing "PASS" results here)*
---

## 💡 Key Challenges & Learnings

### 1. The "Split Vote" Deadlock
Initially, my Raft implementation struggled with liveness during network partitions. Nodes would endlessly split votes.
* **Fix:** I refined the randomized election timeout range. By spreading out the timeouts, one node is statistically guaranteed to time out first and win the election.

### 2. Deadlocks with Channels vs. Mutexes
I initially tried to manage state using Go channels exclusively, but it led to complex circular dependencies.
* **Refactor:** I switched to a shared-memory design using `sync.Mutex` for state protection and `sync.Cond` for signaling state changes (like `commitIndex` updates), which simplified the logic significantly.

### 3. Linearizability
Understanding that "committing a log" is not the same as "applying to the state machine" was crucial. The response to the client must only be sent after the entry is **applied** to the KV map, not just when Raft commits it.

---

## 📚 References
* [In Search of an Understandable Consensus Algorithm (Extended Version)](https://raft.github.io/raft.pdf)
* [MIT 6.824 Schedule & Materials](https://pdos.csail.mit.edu/6.824/)
* [A Tour of Go](https://tour.golang.org/)

---

### 📫 Contact
Feel free to reach out if you want to discuss Distributed Systems, Go, or Infrastructure engineering!

**Ruize Liu**
* Email: [Your Email]
* LinkedIn: [Your LinkedIn Link]
