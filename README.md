# Distributed Systems Implementation (MIT 6.824)

This repository documents my implementation of the MIT 6.824 Distributed Systems labs using **Go**.

> ⚠️ **Note:** To adhere to the course collaboration policy, the source code is kept in a **private repository**. I am happy to discuss the implementation details, architecture design, and race condition handling during an interview.

## 🚀 Key Features Implemented

### 1. Raft Consensus Module (Lab 2)
- **Leader Election:** Randomized election timeouts to minimize split votes.
- **Log Replication:** Consistency checks and conflict resolution.
- **Persistence:** Saving state (currentTerm, votedFor, log) to stable storage to survive crashes.
- **Optimized Concurrency:** Heavy use of `select`, `time.Ticker`, and `sync.Cond` to handle asynchronous events.

### 2. Distributed Key-Value Store (Lab 3)
- Built on top of the Raft library.
- **Linearizable Consistency:** Ensures strict consistency for Get/Put/Append operations.
- **Snapshotting:** Log compaction to reduce storage usage and speed up recovery.
- **Duplicate Detection:** Idempotency mechanism to handle unreliable networks.
