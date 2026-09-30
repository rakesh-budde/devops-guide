# Section 4: etcd

etcd is the only stateful component in the Kubernetes control plane. Everything else — the apiserver, scheduler, controller-manager, kubelet — is stateless and reconstructs its view from etcd on restart. If etcd is unavailable, Kubernetes cannot accept writes or schedule new work. If etcd data is corrupted or lost, cluster state is lost. Understanding etcd means understanding Raft consensus, WAL semantics, snapshot mechanics, compaction, and the specific read/write paths Kubernetes relies on — because etcd failures manifest as some of the most severe and hardest-to-diagnose Kubernetes outages.

## Subtopic Index

- [Architecture](#architecture)
- [Raft Algorithm](#raft-algorithm)
- [Leader Election](#leader-election)
- [Log Replication](#log-replication)
- [WAL — Write-Ahead Log](#wal--write-ahead-log)
- [Snapshots](#snapshots)
- [Quorum and Cluster Sizing](#quorum-and-cluster-sizing)
- [Consistency Models](#consistency-models)
- [Read Path](#read-path)
- [Write Path](#write-path)
- [Compaction](#compaction)
- [Defragmentation](#defragmentation)
- [Backup and Restore](#backup-and-restore)
- [Failure Handling](#failure-handling)

---

## 🗺️ Visual Overview

**Mind map — the whole section at a glance** (skim first, revisit last):

```mermaid
mindmap
  root((etcd and Raft))
    Raft Consensus
      Terms as logical clock
      One leader per term
      AppendEntries RPC
      Pre vote phase
    Leader Election
      Randomized timeout
      RequestVote
      Majority wins
      Step down on higher term
    Log Replication
      Append to WAL
      fdatasync durability
      Commit on quorum
      Apply to bbolt
    MVCC and Revisions
      New revision per write
      Compound key plus revision
      Tombstone on delete
      Watch since revision
    Watch
      Stream events
      Resume from revision
      CompactRevision error
    Compaction and Defrag
      Drop old revisions
      Frees bbolt pages
      Defrag reclaims disk
    Quorum
      Majority is N over 2 plus 1
      Odd nodes only
      Tolerate minority loss
    Backup and Restore
      snapshot save
      snapshot restore
      New cluster id
    Performance Tuning
      NVMe SSD
      WAL fsync p99
      Heartbeat and election timeout
```

**Raft state machine — how a node becomes leader** (the single highest-value diagram):

```mermaid
stateDiagram-v2
    [*] --> Follower
    Follower --> Candidate: election timeout<br/>no heartbeat
    Candidate --> Candidate: split vote<br/>bump term, retry
    Candidate --> Leader: wins majority quorum
    Candidate --> Follower: sees higher term
    Leader --> Follower: sees higher term<br/>or partitioned away
    Leader --> [*]

    classDef start fill:#e3f2fd,stroke:#1565c0,color:#0d47a1,stroke-width:2px;
    classDef proc fill:#fff9c4,stroke:#f9a825,color:#000,stroke-width:2px;
    classDef good fill:#c8e6c9,stroke:#2e7d32,color:#1b5e20,stroke-width:2px;
    classDef ctrl fill:#e1bee7,stroke:#6a1b9a,color:#4a148c,stroke-width:2px;
    class Follower proc
    class Candidate start
    class Leader ctrl
```

**The write path — client → leader → quorum → commit** (memorize this flow):

```mermaid
flowchart LR
    C["📥 Client<br/>kube-apiserver<br/>Put or Txn"] --> L["👑 Leader<br/>append entry<br/>WAL fsync"]
    L --> F1["📗 follower 1<br/>append WAL<br/>fsync + ack"]
    L --> F2["📗 follower 2<br/>append WAL<br/>fsync + ack"]
    F1 --> Q{"🗳️ Majority<br/>acked?"}
    F2 --> Q
    Q -->|"yes: quorum reached"| CM["✅ Commit<br/>apply to bbolt<br/>bump revision"]
    Q -->|"no: still waiting"| W["⏳ Retry / block"]
    CM --> R["📤 Respond<br/>to client"]

    classDef start fill:#e3f2fd,stroke:#1565c0,color:#0d47a1,stroke-width:2px;
    classDef proc fill:#fff9c4,stroke:#f9a825,color:#000,stroke-width:2px;
    classDef good fill:#c8e6c9,stroke:#2e7d32,color:#1b5e20,stroke-width:2px;
    classDef bad fill:#ffcdd2,stroke:#c62828,color:#b71c1c,stroke-width:2px;
    classDef ctrl fill:#e1bee7,stroke:#6a1b9a,color:#4a148c,stroke-width:2px;
    classDef store fill:#ffe0b2,stroke:#e65100,color:#000,stroke-width:2px;
    class C start
    class L ctrl
    class F1,F2 store
    class Q proc
    class CM,R good
    class W bad
```

**Quorum math — why clusters are odd-numbered:**

```mermaid
flowchart LR
    N["🔢 N members"] --> Q["🗳️ Quorum = floor N over 2 plus 1"]
    Q --> T["🛡️ Tolerates N minus quorum failures"]
    Q --> O["⚠️ Use ODD N only<br/>4 tolerates same as 3"]

    classDef start fill:#e3f2fd,stroke:#1565c0,color:#0d47a1,stroke-width:2px;
    classDef proc fill:#fff9c4,stroke:#f9a825,color:#000,stroke-width:2px;
    classDef good fill:#c8e6c9,stroke:#2e7d32,color:#1b5e20,stroke-width:2px;
    classDef bad fill:#ffcdd2,stroke:#c62828,color:#b71c1c,stroke-width:2px;
    class N start
    class Q proc
    class T good
    class O bad
```

> 🧠 **Memory hooks (mnemonics):**
> - **Quorum** = ⌊N/2⌋ + 1 → **3 nodes need 2, 5 nodes need 3.**
> - **"Odd nodes only"** — a 4th node tolerates the same **1** failure as 3 nodes, just slower.
> - **Raft states:** *"Follow the Candidate to Leadership"* → **F**ollower → **C**andidate → **L**eader.
> - **"WAL first, bbolt later"** — durability lives in the fsync'd log, not the B-tree.
> - **"Higher term wins"** — any node that sees a bigger term number steps down instantly (no split-brain).
> - **Restore truth:** *"snapshot = state, not WAL"* — everything after the last snapshot is lost on restore.

---

## Architecture

> 🎯 **Interview weight: High** — the foundation for every etcd question; know the ports, storage engine, and "sole client" facts cold.

**In one line:** etcd is a **distributed key-value store** built on **Raft**, and in Kubernetes the **kube-apiserver is its only client**, mapping every API object to a key under `/registry/...`.

**What etcd is:**

- A **distributed key-value store** on the **Raft** consensus protocol.
- Exposes a **gRPC API** (plus an HTTP/JSON gateway) for read, write, **watch**, and transaction operations.
- The **kube-apiserver is the sole etcd client** — it maps every object to keys under `/registry/<group>/<resource>/<namespace>/<name>`.

**How a member is built:**

- Runs as an **odd number of members** (3 or 5) so a **majority quorum** can always form.
- Each member holds a **full copy** of the data across three artifacts:

| Artifact | What it is | Role |
|---|---|---|
| `bbolt` | Embedded B-tree on disk | The state machine (current + historical values) |
| **WAL** | Write-ahead log | Durability — fsync'd **before** bbolt |
| **Snapshots** | Periodic serialized dumps | Bound WAL size, bootstrap laggards |

- **Ports:** peer/Raft traffic on **2380**, client traffic on **2379**.

```mermaid
flowchart TB
    Client["📥 kube-apiserver<br/>sole client<br/>gRPC :2379"] --> L
    subgraph cluster["etcd cluster · Raft peers :2380"]
      L["👑 member 1 LEADER<br/>WAL + bbolt"]
      F1["📗 member 2 follower<br/>WAL + bbolt"]
      F2["📗 member 3 follower<br/>WAL + bbolt"]
      L <-->|"Raft :2380"| F1
      L <-->|"Raft :2380"| F2
      F1 <-->|"Raft :2380"| F2
    end

    classDef start fill:#e3f2fd,stroke:#1565c0,color:#0d47a1,stroke-width:2px;
    classDef proc fill:#fff9c4,stroke:#f9a825,color:#000,stroke-width:2px;
    classDef good fill:#c8e6c9,stroke:#2e7d32,color:#1b5e20,stroke-width:2px;
    classDef bad fill:#ffcdd2,stroke:#c62828,color:#b71c1c,stroke-width:2px;
    classDef ctrl fill:#e1bee7,stroke:#6a1b9a,color:#4a148c,stroke-width:2px;
    classDef store fill:#ffe0b2,stroke:#e65100,color:#000,stroke-width:2px;
    class Client start
    class L ctrl
    class F1,F2 store
```

> ⚠️ **etcd is for coordination, not bulk data.** It's tuned for **many small writes** (typical Kubernetes write < 1 KB), moderate reads, and **strong consistency**. It is **not** an application data store, cache, or high-throughput database.

> 🔍 **Why big objects hurt:** etcd keeps **all revisions** in memory and on disk. Giant ConfigMaps or CRDs with embedded blobs bloat every member and drag down the whole control plane.

### Key commands
```bash
# Set etcd client environment variables (self-managed cluster)
export ETCDCTL_API=3
export ETCDCTL_ENDPOINTS=https://127.0.0.1:2379
export ETCDCTL_CACERT=/etc/kubernetes/pki/etcd/ca.crt
export ETCDCTL_CERT=/etc/kubernetes/pki/etcd/server.crt
export ETCDCTL_KEY=/etc/kubernetes/pki/etcd/server.key

# Cluster health
etcdctl endpoint health --cluster --write-out=table
etcdctl endpoint status --cluster --write-out=table

# List Kubernetes API objects stored in etcd
etcdctl get /registry --prefix --keys-only | head -30
etcdctl get /registry/pods/default --prefix --keys-only

# Read a specific object (binary encoded, decode with auger)
etcdctl get /registry/pods/default/my-pod
```

---

## Raft Algorithm

> 🎯 **Interview weight: High** — Raft is *the* etcd deep-dive topic; expect to explain terms, quorum, and the commit path.

**In one line:** Raft is a **consensus algorithm** (a more understandable Paxos) that makes every member agree on the **same ordered sequence** of key-value ops, tolerating any **minority** failure.

**Why Raft exists:** it delivers linearizable reads/writes across a cluster where any minority may fail — with rules simple enough to reason about.

**Terms — Raft's logical clock:**

- Time is divided into **terms**, each uniquely numbered.
- A term **begins with an election**; there is **at most one leader per term**.
- A term with no winner (split vote / simultaneous timeout) → the **next** term holds another election.

> 🧠 **The single most important Raft rule:** *a node that sees a **higher term number** immediately steps down.* This is how a stale, partitioned leader learns it's obsolete — no split-brain possible.

**Everything linearizable goes through the leader.** For each such request the leader:

1. Receives the client request.
2. **Appends** it to its log.
3. Sends `AppendEntries` RPCs to all followers.
4. Waits for **majority acknowledgment**.
5. **Commits** the entry and applies it to the state machine (bbolt B-tree).
6. Responds to the client.

💡 The apply step is the observable side effect: the key-value pair lands in bbolt **and** watchers on that key get notified.

### Key commands
```bash
# Check which member is the current leader
etcdctl endpoint status --cluster --write-out=table | grep true

# Check member list (IDs, peer URLs, client URLs)
etcdctl member list --write-out=table

# Inject artificial leader change (for testing — use with care)
etcdctl move-leader <target-member-id>
```

---

## Leader Election

> 🎯 **Interview weight: High** — the classic "walk me through what happens when the leader dies" question.

**In one line:** A follower whose **election timeout** fires becomes a **candidate**, votes for itself, and wins if it collects a **majority** — with randomized timeouts and a **pre-vote** phase keeping the process stable.

**Follower → Candidate → Leader:**

```mermaid
stateDiagram-v2
    [*] --> Follower
    Follower --> Candidate: election timeout fires<br/>no heartbeat from leader
    Candidate --> Candidate: split vote<br/>new term, retry
    Candidate --> Leader: majority votes<br/>floor n over 2 plus 1
    Candidate --> Follower: discovers higher term
    Leader --> Follower: sees higher term<br/>or rejoins after partition

    classDef start fill:#e3f2fd,stroke:#1565c0,color:#0d47a1,stroke-width:2px;
    classDef proc fill:#fff9c4,stroke:#f9a825,color:#000,stroke-width:2px;
    classDef ctrl fill:#e1bee7,stroke:#6a1b9a,color:#4a148c,stroke-width:2px;
    class Follower proc
    class Candidate start
    class Leader ctrl
```

**As a candidate, the node:**

1. **Increments** its current term.
2. **Votes for itself.**
3. Sends `RequestVote` RPCs to all members, including its **last log index and term**.

**A voter grants its vote only if both hold:**

- It has **not already voted** this term, **and**
- The candidate's log is **at least as up-to-date** (compare last log term, then last log index).

➡️ Collect a **majority** (⌊n/2⌋ + 1, self included) → become leader → immediately send `AppendEntries` heartbeats to assert leadership and suppress rivals.

> ⚠️ **Split vote:** if nobody reaches majority, the next term starts a fresh election. **Randomized timeouts (150–300 ms)** stagger candidates so split votes are rare.

> 🔍 **Pre-vote phase (etcd uses it):** before bumping the term and sending `RequestVote`, a candidate first sends **`PreVote`** probes that *don't* increment the term. No majority agreement → no disruption. This stops a **partitioned member from endlessly inflating the term** and forcing re-elections when it rejoins.

### Key commands
```bash
# Monitor leader changes (watch metrics)
watch -n2 "etcdctl endpoint status --cluster --write-out=table"

# Check current term (indicates election history)
etcdctl endpoint status --write-out=json | python3 -m json.tool | grep raftTerm

# Force-resign current leader (useful for maintenance of leader node)
etcdctl move-leader $(etcdctl endpoint status --cluster --write-out=json | \
  python3 -c "import json,sys; d=json.load(sys.stdin); \
  print([m['Status']['header']['member_id'] for m in d if not m['Status']['leader'] == 0][0])")
```

---

## Log Replication

> 🎯 **Interview weight: High** — the mechanism behind durability, consistency, and why etcd needs fast disks.

**In one line:** The leader appends each write to an **immutable, ordered log**, replicates it via `AppendEntries`, and **commits only after a majority persists it** — with a consistency check that keeps logs from ever diverging.

**The log itself:**

- A sequence of entries, each with a **term**, an **index**, and a **command** (the key-value op).
- Once **committed**, an entry is **permanent** — never removed from a correct replica. This immutability *is* Raft's safety property.

**`AppendEntries` does double duty:** empty = **heartbeat**; with data = **replication**. On a client write the leader:

1. Appends the entry to its log and calls **`fdatasync`** to persist it to the WAL.
2. Sends `AppendEntries` to all followers **concurrently** with a consistency check (`previousLogIndex`, `previousLogTerm`).
3. Waits for a **majority** ack (self included).
4. Marks the entry **committed**; updates `commitIndex`.
5. **Applies** to the state machine (bbolt) and responds to the client.
6. On the next `AppendEntries`/heartbeat, tells followers the new `commitIndex`; followers apply committed entries **asynchronously**.

> 🔍 **The consistency check prevents divergence:** a follower appends an entry *only if* its log matches the leader's up to the previous entry. Any inconsistent suffix (from a prior leader's **uncommitted** entries) is **overwritten** — safe, because those entries were never acked to a client.

> ⚠️ **Write amplification — why disks matter.** Every write pays for:
>
> | # | Cost | On critical path? |
> |---|---|---|
> | 1 | Leader WAL `fdatasync` | ✅ **Yes** — dominant latency |
> | 2 | Network round-trip to followers | ✅ Yes |
> | 3 | Follower WAL `fdatasync` | ✅ Yes |
> | 4 | Follower acknowledgment | ✅ Yes |
> | 5 | Leader state-machine apply | ⬜ After commit |
>
> **Slow disk = slow writes, directly.** The leader `fdatasync` gates every acknowledgment.

### Key commands
```bash
# Check log index and commit index across all members
etcdctl endpoint status --cluster --write-out=json | \
  python3 -c "import json,sys; [print(m['Endpoint'], 'raftIndex:', m['Status']['raftIndex']) for m in json.load(sys.stdin)]"

# Check if any follower is significantly behind the leader
# Large difference between leader raftIndex and follower raftIndex indicates lag
```

---

## WAL — Write-Ahead Log

> 🎯 **Interview weight: High** — the durability core, and the source of etcd's #1 operational metric.

**In one line:** The WAL is etcd's **durability mechanism** — every change is `fdatasync`'d to an append-only log **before** touching bbolt, so a crash can be replayed back to a consistent state.

**How it works:**

- Before **any** bbolt modification, the change is written to the WAL and **`fdatasync`'d** to disk.
- Crash *after* WAL write but *before* bbolt update? → **replay the WAL** on restart to reconstruct state.
- WAL files live in `--wal-dir` (defaults to the data dir), are **sequentially named** and **append-only**.
- Each record is either a **Raft log entry** (the op) or a **state record** (current term + vote), carrying: `type`, `data` (Raft entry bytes), and a **CRC32 checksum**.

**Startup replay:** etcd loads the last snapshot, then **replays WAL entries after it** in order to rebuild the state machine. 🔍 This is why startup time grows with WAL size when no recent snapshot exists.

**`fdatasync` latency dominates write latency:**

| Storage | Typical `fdatasync` |
|---|---|
| HDD | **10–20 ms** ❌ |
| Consumer SSD | 1–5 ms |
| Datacenter NVMe | **< 1 ms** ✅ |

> ⚠️ **The 1s election timeout must dwarf fsync latency** — otherwise a disk hiccup delays heartbeats and triggers **spurious leader elections**.

> 🧠 **Memorize this metric:** `etcd_disk_wal_fsync_duration_seconds` is the **single most important etcd signal**. **p99 > 10 ms = disk pressure and impending instability.**

### Key commands
```bash
# Check WAL fsync latency (the single most important etcd metric)
etcdctl endpoint status --write-out=table   # also shows db size
# Via Prometheus:
# etcd_disk_wal_fsync_duration_seconds_bucket
# etcd_disk_backend_commit_duration_seconds_bucket

# Find etcd's data directory
systemctl cat etcd | grep data-dir
ls -lh /var/lib/etcd/member/wal/

# Check WAL file sizes
du -sh /var/lib/etcd/member/wal/
```

---

## Snapshots

> 🎯 **Interview weight: Medium** — know the two purposes and how they relate to WAL size and backups.

**In one line:** A snapshot is a **full serialized dump of bbolt** at a Raft index — it **caps WAL growth** and lets a far-behind follower catch up without replaying thousands of entries.

**Two jobs a snapshot does:**

1. **Bootstrap a laggard** — a follower too far behind to catch up via log replay alone.
2. **Bound WAL size** — entries before the snapshot index can be discarded.

**Automatic snapshots:**

- Triggered when applied entries since the last snapshot exceed **`--snapshot-count`** (default **100,000**).
- Contain **all key-value pairs at the current revision** + metadata (term, index).
- Written to `<data-dir>/member/snap/`; afterward, older WAL entries are **purged**.

**Catching up a follower:** the leader ships the **snapshot** instead of replaying thousands of entries. The follower applies it directly to bbolt (replacing its state), then resumes normal log replication from the snapshot index.

> 💡 **For backups**, `etcdctl snapshot save` creates an **on-demand** snapshot — the primary disaster-recovery mechanism (see [Backup and Restore](#backup-and-restore)).

### Key commands
```bash
# Create an on-demand snapshot (backup)
etcdctl snapshot save /backup/etcd-snapshot-$(date +%Y%m%d-%H%M%S).db

# Verify snapshot integrity
etcdctl snapshot status /backup/etcd-snapshot.db --write-out=table
# Shows: hash, revision, total keys, total size

# Check automatic snapshot files on disk
ls -lh /var/lib/etcd/member/snap/

# Snapshot size roughly correlates with cluster state size
du -sh /var/lib/etcd/
```

---

## Quorum and Cluster Sizing

> 🎯 **Interview weight: High** — quorum math and "why odd numbers?" are near-guaranteed questions.

**In one line:** Raft needs a **majority (⌊n/2⌋ + 1)** to commit, so clusters are **always odd** — a 4th member adds latency without adding fault tolerance.

**The math to know cold:**

| Cluster size | Write quorum | Tolerated failures |
|---|---|---|
| 1 | 1 | 0 |
| **3** | **2** | **1** |
| **5** | **3** | **2** |
| 7 | 4 | 3 |

> 🧠 **Why odd only:** a **4-node** cluster needs **3** to agree — the *same* 1-failure tolerance as 3 nodes, but the leader now waits for **2 of 3** followers instead of 1. More cost, no benefit.

**Why not go big (7+)?**

- ⬆️ **Write latency** rises (more acks to wait for).
- ⏳ **Leader election** takes longer.
- 🔧 **Operational complexity** grows.
- ✅ **5 members already tolerate 2 failures**, covering most datacenter scenarios.
- 📐 For huge deployments, the answer is **multiple smaller etcd clusters** (API partitioning), not one giant cluster.

> ⚠️ **etcd is latency-sensitive between members.** Defaults (heartbeat **100 ms**, election timeout **1 s**) assume **< 10 ms RTT**. Cross-datacenter etcd (**> 50 ms RTT**) needs `--heartbeat-interval` and `--election-timeout` tuned **up** — and that larger election timeout becomes the **minimum downtime** during a leader failure. A real multi-region trade-off.

### Key commands
```bash
# Add a new member (before starting the new etcd process)
etcdctl member add etcd4 --peer-urls=https://etcd4:2380

# Remove a failed member
etcdctl member remove <member-id>

# Check member health and round-trip times
etcdctl endpoint health --cluster --write-out=table --dial-timeout=5s
```

---

## Consistency Models

> 🎯 **Interview weight: Medium** — know the two levels, which one Kubernetes defaults to, and *why*.

**In one line:** etcd offers **linearizable** (default, always fresh) and **serializable** (fast, possibly stale) reads — Kubernetes defaults to **linearizable** for correctness.

| | **Linearizable** (default) | **Serializable** |
|---|---|---|
| Guarantee | Sees the **most recent committed write** | May be **one or more entries behind** |
| Quorum check | ✅ Leader confirms via **read-index heartbeat** to a quorum | ❌ None |
| Speed | Slower (round-trip) | **Faster** (local) |
| Served by | Confirmed current leader | Any member, incl. followers |
| Kubernetes use | **Default** | Rare, controlled cases only |

> 🔍 **Why the quorum check matters:** it stops a **stale leader** (partitioned away, not yet deposed) from serving out-of-date reads.

**Why Kubernetes insists on linearizable:** when the apiserver reads to check "does this pod already exist?", a stale read could **create a duplicate pod or miss a deletion**. Correctness depends on it.

> 💡 **The watch cache absorbs most read load.** The apiserver's in-memory watch cache (Section 3) serves the vast majority of reads. Direct linearizable etcd reads happen only for edge cases — `GET` with `resourceVersion=""` (opting out of the cache) or when the cache has expired.

### Key commands
```bash
# Linearizable read (default)
etcdctl get /registry/pods/default/my-pod

# Serializable read (faster, potentially stale)
etcdctl get /registry/pods/default/my-pod --consistency=s

# Check linearizable read latency
etcdctl endpoint status --write-out=table  # includes round-trip time to leader
```

---

## Read Path

> 🎯 **Interview weight: Medium** — ties together MVCC, revisions, and how watches are possible.

**In one line:** A linearizable read goes to the **leader**, which confirms leadership via a read-index heartbeat, then reads the **current revision** of the key from the MVCC bbolt B-tree.

**Linearizable read sequence:**

1. Client sends `Range` RPC to **any** member.
2. **Leader** → confirms leadership via **read-index heartbeat** → reads bbolt → serializes → returns.
3. **Follower** → forwards to the leader (or errors, telling the client to retry via the leader).

**MVCC — the key mental model:**

- bbolt stores **all key-value pairs at all revisions** (Multi-Version Concurrency Control).
- Each write creates a **new revision**, never an overwrite; the **`(key, revision)`** pair uniquely identifies a value.
- The "current" value = the **highest-revision, non-deleted** write for that key.

> 🧠 **MVCC is what makes watch work:** a watcher asking for changes **since revision N** is served by scanning all entries with **revision > N** for the watched prefix. The apiserver's watch cache implements this one level up, directly from Raft log entries.

> 💡 **Range scans are cheap** because bbolt keeps keys **sorted by byte representation**. Listing all pods in a namespace is just a range scan over `/registry/pods/<namespace>/`.

### Key commands
```bash
# Range scan (list all pods in a namespace)
etcdctl get /registry/pods/default --prefix --keys-only

# Get a specific key with its revision
etcdctl get /registry/pods/default/my-pod -w json | python3 -m json.tool

# Watch for changes on a key (useful for debugging)
etcdctl watch /registry/pods/default --prefix

# Check etcd revision (monotonically increasing, unique per write)
etcdctl endpoint status --write-out=json | python3 -m json.tool | grep revision
```

---

## Write Path

> 🎯 **Interview weight: High** — combines Raft, `Txn` optimistic concurrency, protobuf encoding, and the two-fsync cost.

**In one line:** A write travels **client → leader (WAL fsync) → followers (WAL fsync + ack) → commit to bbolt → respond**, and the apiserver wraps it in a `Txn` for **compare-and-swap** concurrency.

```mermaid
flowchart LR
    C["📥 Client<br/>Put or Txn"] --> L["👑 Leader<br/>append log<br/>WAL fsync"]
    L --> F["📗 Followers<br/>append WAL<br/>fsync + ack"]
    F --> Q{"🗳️ Majority ack?"}
    Q -->|"yes"| CM["✅ Commit<br/>apply bbolt<br/>bump revision"]
    Q -->|"no"| X["❌ Fail / retry"]
    CM --> R["📤 Respond to client"]
    CM --> H["🔁 Followers commit<br/>on next heartbeat"]

    classDef start fill:#e3f2fd,stroke:#1565c0,color:#0d47a1,stroke-width:2px;
    classDef proc fill:#fff9c4,stroke:#f9a825,color:#000,stroke-width:2px;
    classDef good fill:#c8e6c9,stroke:#2e7d32,color:#1b5e20,stroke-width:2px;
    classDef bad fill:#ffcdd2,stroke:#c62828,color:#b71c1c,stroke-width:2px;
    classDef ctrl fill:#e1bee7,stroke:#6a1b9a,color:#4a148c,stroke-width:2px;
    classDef store fill:#ffe0b2,stroke:#e65100,color:#000,stroke-width:2px;
    class C start
    class L ctrl
    class F,H store
    class Q proc
    class CM,R good
    class X bad
```

**`Txn` = optimistic concurrency (the apiserver's workhorse):**

```
Txn({ If:   [key.modRevision == expectedRevision],
      Then: [Put(key, newValue)],
      Else: [] })
```

- ModRevision matches (no one else changed the key) → **write succeeds**.
- Mismatch → **transaction fails** (CAS failure) → apiserver returns **409 Conflict** to the client.

**Object encoding:** Kubernetes objects are stored as **protobuf** (since 1.6) — a magic byte prefix + the encoded object. Not human-readable in etcd; decode with **`auger`** or **`etcdhelper`**.

> ⚠️ **Two fsyncs per write.** The **bbolt commit** (`fdatasync` of the data file) is a *second* disk op, separate from the WAL fsync. etcd **batches** bbolt commits (`--backend-batch-interval`, default **100 ms**) to amortize cost — but the **WAL fsync stays per-entry on the critical path.**

### Key commands
```bash
# Check bbolt database size and fragmentation
etcdctl endpoint status --write-out=table  # shows "DB SIZE"
# If DB size >> actual object count * avg object size, fragmentation is high

# Watch write rate
# etcd_mvcc_put_total, etcd_mvcc_delete_total in Prometheus

# Decode a protobuf-encoded Kubernetes object from etcd (requires auger)
etcdctl get /registry/pods/default/my-pod | auger decode

# Check current revision (write counter)
etcdctl get "" --from-key --rev=0 --limit=1 --keys-only  # get latest revision
```

---

## Compaction

> 🎯 **Interview weight: High** — the root of the infamous "informer relist storm" and runaway DB growth.

**In one line:** Compaction **drops MVCC revisions below a threshold** so etcd stops growing forever — but clients watching from an older revision get a **`CompactRevision` error**.

**Why it's needed:** MVCC keeps **every** historical revision of every key. Without compaction, memory and disk grow **indefinitely**. Compaction discards revisions below a point, keeping only history from there forward.

**How Kubernetes drives it:**

- The apiserver auto-triggers via **`--etcd-compaction-interval`** (default **5 min**), compacting to *latest revision minus configured history*.
- After compaction, any watch/read at a revision **older than the compaction point** gets a **`CompactRevision` error** → apiserver surfaces it as **410 Gone**.

> ⚠️ **The relist storm:** that 410 forces every affected informer to do a **full LIST**. If compaction jumps past all watches at once, **all controllers relist simultaneously** (the Section 3 production incident).

> 🔍 **Compaction does NOT free disk.** It marks old B-tree pages **free**; the bbolt file (`member/snap/db`) stays the same size. **Defragmentation** (next) is what actually reclaims disk.

💡 **Cost:** on a large cluster (2M+ keys, 10M+ revisions), a compaction scan can take **seconds** and raise write latency — tracked by `etcd_debugging_mvcc_db_compaction_pause_duration_milliseconds`.

### Key commands
```bash
# Compact manually to the current revision (emergency disk recovery)
REV=$(etcdctl endpoint status --write-out=json | \
  python3 -c "import json,sys; print(json.load(sys.stdin)[0]['Status']['header']['revision'])")
etcdctl compact $REV

# Check compaction progress in metrics
# etcd_mvcc_db_compaction_keys_total
# etcd_mvcc_db_compaction_pause_duration_milliseconds

# View current revision and compaction point
etcdctl endpoint status --cluster --write-out=json | \
  python3 -m json.tool | grep -E 'revision|compactRevision'
```

---

## Defragmentation

> 🎯 **Interview weight: Medium** — know it reclaims disk, locks the member, and must be done **one at a time**.

**In one line:** Defrag **rewrites the bbolt file sequentially** to return freed pages to the OS — shrinking the file — while holding an **exclusive lock** that blocks that member.

**Why compaction alone isn't enough:** freed pages are marked available but **not returned to the OS**; the on-disk file stays at its **peak size**. Defrag physically compacts it.

**What `etcdctl defrag` does (online):**

1. Create a new empty bbolt database.
2. Walk the existing DB, copying **all live key-value pairs** to the new one.
3. **Atomically replace** the old database with the new one.

> ⚠️ **Defrag locks the member.** During defrag, reads **and** writes on that member are blocked (seconds to minutes on a large DB). **Defrag one member at a time — never all at once**, or you take the whole cluster down.

💡 **Operational cadence:** schedule during **low-traffic windows** — many clusters defrag **weekly** or when the file crosses a threshold (e.g., `etcd_mvcc_db_total_size_in_bytes > 4Gi`).

### Key commands
```bash
# Defragment a single member (causes brief lock)
etcdctl defrag --endpoints=https://etcd1:2379   # defrag only this member

# Defrag all members one at a time (automated, with health checks between)
for EP in $(etcdctl endpoint status --cluster --write-out=json | \
  python3 -c "import json,sys; [print(m['Endpoint']) for m in json.load(sys.stdin)]"); do
  echo "Defragging $EP"
  etcdctl defrag --endpoints=$EP
  sleep 5
  etcdctl endpoint health --endpoints=$EP
done

# Check database size before and after defrag
etcdctl endpoint status --cluster --write-out=table  # before
# ... defrag ...
etcdctl endpoint status --cluster --write-out=table  # after
```

---

## Backup and Restore

> 🎯 **Interview weight: High** — disaster recovery is a favorite scenario; know the commands and what a restore does *not* recover.

**In one line:** Backup = **`etcdctl snapshot save`**; restore = **`etcdctl snapshot restore`** into a fresh data dir — but the snapshot holds **state, not WAL**, so everything after it is lost.

etcd backup means taking a **snapshot of the database**, used to recover from catastrophe: corrupted data, unrecoverable quorum loss, or accidental mass deletion.

**Backup procedure:**
```bash
ETCDCTL_API=3 etcdctl snapshot save \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  /backup/etcd-$(date +%Y%m%d-%H%M%S).db

# Verify
etcdctl snapshot status /backup/etcd-*.db --write-out=table
```

**Restore procedure** (after a complete etcd loss):
```bash
# Stop apiserver and etcd on all control-plane nodes
# On each node, restore from snapshot:
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-backup.db \
  --name=etcd1 \
  --initial-cluster="etcd1=https://etcd1:2380,etcd2=https://etcd2:2380,etcd3=https://etcd3:2380" \
  --initial-cluster-token=etcd-cluster-1 \
  --initial-advertise-peer-urls=https://etcd1:2380 \
  --data-dir=/var/lib/etcd-restore

# Move restored data into place and restart etcd
mv /var/lib/etcd /var/lib/etcd-old
mv /var/lib/etcd-restore /var/lib/etcd
# Restart etcd (static pod: move manifest, wait, move back)
```

> ⚠️ **A snapshot = state, NOT the WAL.** It holds all key-value data **at the snapshot revision** only. Restoring means **all progress after the snapshot is lost** — which is why production takes automated snapshots **every 5–30 minutes**.

> 🔍 **etcd holds API objects only.** Tools like **Velero** also capture **PersistentVolume** data — etcd never contains the application data inside volumes.

### Key commands
```bash
# Automated backup as a CronJob (conceptual — runs in a privileged pod on control plane)
# Store backups with rotation: keep last 24h hourly, last 7 days daily
# Use rclone/aws s3 cp to push to remote storage

# List available backups and check integrity
for SNAP in /backup/*.db; do
  echo -n "$SNAP: "
  etcdctl snapshot status $SNAP --write-out=table 2>/dev/null | grep -E 'Hash|Total|Size'
done
```

---

## Failure Handling

> 🎯 **Interview weight: High** — the payoff section; scenario questions here separate juniors from seniors.

**In one line:** Failures scale from **single-member loss** (no downtime) → **quorum loss** (read-only) → **total data loss** (restore from backup) — and Raft makes **split-brain impossible**.

**The failure spectrum:**

| Scenario (3-node cluster) | Quorum? | Effect | Recovery |
|---|---|---|---|
| **Single member down** | ✅ 2/3 | Writes + reads continue; reduced redundancy | Restart → catch up via replication or snapshot |
| **Leader down** | ✅ | Writes rejected **~1–2s** during election, then resume | Automatic re-election; committed writes preserved |
| **Two members down** | ❌ 1/3 | **Read-only** — writes fail; no scheduling/updates | Restart failed members, or `--force-new-cluster` from survivor |
| **Complete data loss** | ❌ | State gone; kubelet stops containers apiserver "forgets" | Restore from snapshot backup, rebuild |

**Key details:**

- 🟢 **Single member:** the failed member misses entries while down and catches up on restart (log replication if within the snapshot window, else a full snapshot). No control-plane impact beyond lost redundancy.
- 🟣 **Leader failure:** followers notice the missing heartbeat within the election timeout (default **1s**); the ~1–2s write pause is usually invisible thanks to apiserver retries. **All committed writes survive.**
- 🔴 **Two-member failure:** the survivor goes **read-only** — serves reads (possibly stale via serializable) but **cannot commit**. Kubernetes can't schedule pods, create resources, or update status.
- ⚠️ **Complete data loss:** everything after the last snapshot is gone. Post-backup pods vanish from etcd; the kubelet gets "pod no longer exists" and stops those containers.

> 🧠 **Split-brain is impossible in etcd — Raft guarantees it.** The minority partition **lacks quorum** and can never commit. Even a stale minority leader makes no progress; on heal, it sees a **higher term** and steps down instantly.

```mermaid
flowchart TD
    Start["💥 Member fails"] --> Q{"🗳️ Quorum remaining?"}
    Q -->|"Yes: n-1 still >= majority"| Continue["✅ Cluster continues<br/>reduced redundancy"]
    Q -->|"No: quorum lost"| ReadOnly["🔴 Writes fail<br/>reads may work"]
    ReadOnly --> Recover{"Failed members<br/>recoverable?"}
    Recover -->|"Yes: data intact, restart"| Rejoin["♻️ Members rejoin<br/>catch up via replication"]
    Recover -->|"No: data lost"| Restore["🗄️ Restore from<br/>snapshot backup"]

    classDef start fill:#e3f2fd,stroke:#1565c0,color:#0d47a1,stroke-width:2px;
    classDef proc fill:#fff9c4,stroke:#f9a825,color:#000,stroke-width:2px;
    classDef good fill:#c8e6c9,stroke:#2e7d32,color:#1b5e20,stroke-width:2px;
    classDef bad fill:#ffcdd2,stroke:#c62828,color:#b71c1c,stroke-width:2px;
    classDef ctrl fill:#e1bee7,stroke:#6a1b9a,color:#4a148c,stroke-width:2px;
    classDef store fill:#ffe0b2,stroke:#e65100,color:#000,stroke-width:2px;
    class Start start
    class Q,Recover proc
    class Continue,Rejoin good
    class ReadOnly bad
    class Restore store
```

### Key commands
```bash
# Check quorum status after a failure
etcdctl endpoint health --cluster --dial-timeout=5s

# Force a single member to become a new cluster (LAST RESORT after quorum loss)
# On the surviving member, after stopping all other etcd processes:
etcd --force-new-cluster --data-dir=/var/lib/etcd

# Check if apiserver is rejecting writes due to etcd unavailability
kubectl get nodes 2>&1    # will timeout or show error
journalctl -u kube-apiserver | grep -i 'etcd\|storage\|error' | tail -20

# Recovery: after restoring etcd, restart apiserver to reconnect
# On kubeadm: move and restore apiserver static pod manifest
```

---

## Interview Questions

### Conceptual / Internals (8 questions)

**1. Explain the Raft leader election algorithm. What happens in a 3-node cluster when the leader fails?**

When the leader fails, its followers stop receiving heartbeats. After a randomized election timeout (150–300ms), one follower transitions to candidate, increments its term, votes for itself, and sends `RequestVote` to the other member. If the remaining member grants the vote (hasn't voted in this term and the candidate's log is at least as up-to-date), the candidate wins with a 2-of-3 majority and becomes leader. It immediately sends heartbeats to suppress any other candidate. The new leader has all committed entries from the previous leader — Raft guarantees that no committed entry is ever lost because a majority had to acknowledge it. Uncommitted entries from the old leader (if any) may be overwritten by the new leader. From etcd's perspective, client writes fail for up to ~1 second during the election window, then resume with the new leader.

**2. Why does etcd require low-latency SSDs, and what is the specific failure mode when disk is slow?**

Every Raft log entry requires an `fdatasync` syscall to persist the WAL to disk before the leader can acknowledge the write. If `fdatasync` takes 20ms (typical HDD), the leader cannot ack the write in under 20ms. Under load with many concurrent writes, the disk becomes the bottleneck and write latency grows. More critically: the Raft heartbeat and election timeout depend on timing. If `fdatasync` is slow enough that the leader cannot send heartbeats on schedule, followers trigger spurious elections, causing disruption even with no actual failures. The `etcd_disk_wal_fsync_duration_seconds` p99 metric is the single most important etcd operational signal. Production recommendation: dedicated NVMe SSD for etcd data and WAL directories, separate from the OS disk.

**3. What is the difference between linearizable and serializable reads in etcd, and which does Kubernetes use by default?**

A linearizable read guarantees seeing the most recently committed write. Before serving the read, the leader confirms it is still the current leader by getting acknowledgment from a majority — preventing a partitioned stale leader from serving reads. A serializable read is served from the local state machine without a quorum check; a follower can serve it and may return data that is one or more commits behind the leader. Kubernetes (via the kube-apiserver) uses linearizable reads by default to ensure correctness — the apiserver must not return stale data when checking for resource conflicts or serving user queries. The watch cache overlays this: most Kubernetes reads are served from the apiserver's in-memory watch cache, not directly from etcd. Direct etcd reads happen primarily when the cache is cold or a client explicitly requests a current/uncached read.

**4. Explain etcd MVCC and how it enables the watch mechanism.**

MVCC (Multi-Version Concurrency Control) means etcd never overwrites a key's value — it appends a new revision. Every write creates a new `(key, revision)` pair. Reading a key returns the value at the latest revision for that key. This means etcd holds the complete history of all key mutations in the bbolt B-tree. The watch mechanism leverages this: a watcher registers for changes to a key or prefix since a given revision (`watchRevision`). etcd can serve watch events by scanning all entries with revision > watchRevision for matching keys. This is why the apiserver's watch cache can efficiently reconstruct recent history from etcd's data, and why etcd compaction (removing old revisions) can cause watchers to receive `CompactRevision` errors if they are too far behind.

**5. Why does compaction sometimes cause all Kubernetes controllers to relist simultaneously, and what are the cascading effects?**

Compaction removes revisions below a threshold. If any apiserver watch cache or informer watch is holding a revision older than the compaction point, the etcd watch returns a `CompactRevision` error. The apiserver translates this into a 410 Gone for its watchers (informers in the controller-manager, scheduler, kubelet). Every informer receiving 410 must perform a full LIST (relist) to establish fresh state. If the compaction point advances past all current watches simultaneously (common if compaction hasn't run in a long time and then runs aggressively), all informers in all controllers relist at once. Each relist is a range scan of all objects for that resource type. Thousands of concurrent LIST calls to the apiserver overflow the APF `workload-low` bucket, causing 429s. The controllers back off and retry, eventually converging, but reconciliation is delayed by several minutes.

**6. Explain the snapshot and WAL restore process on etcd startup.**

On startup, etcd checks for the latest snapshot in `<data-dir>/member/snap/`. It loads the snapshot into the bbolt state machine directly (no entry-by-entry replay needed). The snapshot includes a `term`, `index`, and a complete dump of all key-value pairs at that index. etcd then opens the WAL and scans for entries after the snapshot index. These entries are replayed in order: each entry is applied to the bbolt state machine until the WAL is exhausted. If the WAL has uncommitted entries (entries that were in the leader's log but not yet committed before a crash), they are truncated. The final state is a consistent committed view of the cluster. The startup time is proportional to: (1) the snapshot size (read from disk) + (2) the number of WAL entries after the last snapshot.

**7. How does the `etcdctl snapshot restore` command work and why must all members restore from the same snapshot file?**

`etcdctl snapshot restore` takes a snapshot file and creates a new etcd data directory with the snapshot data and a new WAL. The key point: it creates a **new cluster** with a new cluster ID and initial peer URLs. If different members restore from different snapshot files, they will have divergent histories — there is no mechanism to merge two different etcd state machines. All members must restore from the same snapshot to have the same base state. After restore, each member starts fresh from the snapshot revision, and new Raft log entries are appended as writes resume. The `--initial-cluster`, `--name`, and `--initial-advertise-peer-urls` flags must match the cluster's new topology exactly, or members won't be able to peer.

**8. What is the significance of the bbolt defragmentation operation, and what is the risk of running it during business hours?**

bbolt maintains an in-memory freelist of pages that have been released by MVCC compaction or key deletions. These pages are available for reuse but are not returned to the OS; the database file on disk stays at its peak historical size. Defragmentation rewrites the entire database file sequentially, eliminating fragmentation and shrinking the file. During defragmentation, bbolt holds an exclusive write lock — no reads or writes can be served from that member for the duration. On a 2 GB database, defrag may take 10–30 seconds. If run during business hours on a 3-member cluster, one member is offline for 30 seconds. Other members absorb the load. If run simultaneously on all members (a common mistake), the entire cluster is unavailable. The risk is compounded during high write periods: the post-defrag sync of the new database file to disk can spike I/O, competing with normal write traffic.

---

### Scenario / Troubleshooting (6 questions)

**9. etcd reports `mvcc: required revision has been compacted`. What caused this and how do you fix it?**

A client (the apiserver watch, or an informer) requested a watch or read at a revision that etcd has compacted away. This means the client was too far behind — it hasn't read from etcd for long enough that compaction has passed its revision. In the apiserver, this manifests as 410 Gone responses to watchers. The informer handles this by relisting. The fix for most cases is to do nothing — the system self-heals within minutes as informers relist. For persistent issues: (1) check if compaction is too aggressive; consider increasing `--etcd-compaction-interval` on the apiserver or the etcd `--auto-compaction-retention` value; (2) check if a specific informer is stuck and not relisting — look for controllers with old cache generations; (3) ensure watch cache memory is sufficient to hold recent history.

**10. A Kubernetes cluster loses 2 of 3 etcd members simultaneously. Walk through the exact failure progression and recovery steps.**

Immediately after the 2-member loss: the surviving member cannot achieve write quorum. etcd returns errors for all writes. The apiserver attempts to write (e.g., kubelet status patch) and gets errors; it retries with backoff. `kubectl apply`, `kubectl scale`, etc. fail. Reads from the apiserver watch cache continue (the cache is in-memory), so `kubectl get pods` may still work. Running pods continue running. After ~40 seconds without kubelet lease renewal, nodes start showing NotReady (the node lifecycle controller cannot write the NotReady taint because etcd is down — so even this is delayed until etcd comes back). Recovery: if both failed members have intact data, restart both etcd processes with their existing data directories — they will join the surviving member, replay log entries they missed, and the cluster will resume. If one member has intact data but one is corrupted, remove the corrupted member and add a new member. If both are corrupted: restore all three members from the last snapshot backup.

**11. `etcd_disk_wal_fsync_duration_seconds` p99 is 150ms. What are you doing?**

150ms WAL fsync is extremely high and indicates severe disk I/O problems. Immediate investigation: `iostat -xd 1 sda` (or the etcd disk device) to check `%util`, `await`, and `w_await`. If disk utilization is near 100%, something is contending for I/O: (1) check for other processes writing to the same disk (log rotation, system journal, application containers on the same node), (2) check for I/O intensive workloads on the same VM (cloud disk throttling), (3) check for a noisy neighbor on the hypervisor. If the disk is an HDD: migrate to SSD immediately. If NVMe is already in use: check for a firmware issue, RAID configuration, or hardware failure. Interim mitigation: set etcd's `--wal-dir` to a dedicated, faster disk. Reduce the `--election-timeout` concern — 150ms fsync means election timeout should be at minimum 1500ms (10x fsync). Alert: `etcd_disk_wal_fsync_duration_seconds_bucket` p99 > 25ms should trigger a warning.

**12. A developer accidentally runs `etcdctl del /registry --prefix` and wipes the entire Kubernetes cluster state. Describe the recovery.**

The entire etcd key space under `/registry/` is gone. Running pods are still alive (data plane continues), but etcd has no record of them. When the apiserver reconnects to etcd, watches return empty results. Controllers see no objects and begin deleting their notion of desired state. The kubelet, receiving a watch event that its pods no longer exist (from the apiserver, based on empty etcd), begins terminating containers. Depending on timing, this cascades into complete cluster loss. Recovery requires: (1) immediately stop all controllers to prevent reconciliation from acting on empty state (stop controller-manager); (2) restore etcd from the last snapshot backup using `etcdctl snapshot restore`; (3) restart all control-plane components; (4) assess what state was lost between the backup and the deletion. For any objects not in the backup, manually recreate them from your GitOps repository. This scenario underscores why etcd access must require strong authentication and RBAC, and ideally the etcd endpoints should not be accessible outside the control-plane network.

**13. etcd's database size is 8 GB but you only have ~10,000 Kubernetes objects. What happened and how do you fix it?**

High DB size relative to object count has two main causes: (1) **Revision accumulation**: high write rate (many controllers updating status frequently, HPA adjustments, event objects) generates millions of revisions without compaction. Run `etcdctl endpoint status` to check the current revision; if it's in the hundreds of millions, compaction is not keeping up. Fix: run `etcdctl compact $(etcdctl endpoint status --write-out=json | python3 -c "...")` to compact to the current revision, then defragment. (2) **Large objects**: something is storing large data in Kubernetes objects — giant ConfigMaps (>1MB each), CRD objects with embedded binary data, or event objects that haven't been purged. Check with `etcdctl get /registry --prefix -w json | python3 -m json.tool | grep -E '"key"|"value"' | head -50` and identify large entries by value size. Fix the application storing large objects and prune the oversized objects.

**14. You need to upgrade etcd from 3.4 to 3.5 with zero downtime. Describe the procedure.**

etcd supports rolling upgrades for minor/patch versions. Procedure: (1) Verify all 3 members are healthy: `etcdctl endpoint health --cluster`. (2) On member 1 (not the leader — check `etcdctl endpoint status`), stop etcd, update the binary, and restart with the same data directory. etcd 3.5 can join a 3.4 cluster in mixed-version mode. (3) Verify member 1 rejoined: `etcdctl endpoint health`. (4) Repeat for member 2. (5) Move the leader to a non-upgrading member: `etcdctl move-leader <member-2-or-3-id>`. (6) Upgrade the original leader last. (7) After all members run 3.5, verify cluster health. Always check the etcd release notes for migration requirements between specific versions. For major versions (3.x → 4.x), additional migration steps may apply. Take a snapshot backup before starting.

---

### FAANG-Level Deep Dive (6 questions)

**15. How does bbolt implement MVCC, and what is stored at the byte level for each key revision in etcd?**

bbolt is an append-friendly B-tree that does not overwrite existing pages — it allocates new pages for modifications (copy-on-write B-tree pages). etcd's MVCC layer builds on top of bbolt by using a compound key format: `(key-bytes)(8-byte-big-endian-revision-number)`. Every `Put` creates a new compound key with the new revision. A `Get` of the logical key `foo` finds the compound key `foo<max-revision>` — the B-tree scan lands at the lexicographically largest revision for that key. A `Range` watch from revision N scans all compound keys with revision number > N. The current value of a key is the compound key with the highest revision number. Deletion is represented as a tombstone entry (empty value with a `tombstone` flag). Compaction deletes compound keys below the compaction revision by B-tree range deletion, freeing their bbolt pages to the freelist.

**16. Explain the Raft log entry structure used by etcd and how Raft guarantees that no committed entry is ever lost after a leader change.**

An etcd Raft log entry has: `term` (the term when the entry was created by the leader), `index` (monotonically increasing position in the log), `type` (normal entry vs config change), and `data` (the serialized etcd operation — Put/Delete/Txn). The safety guarantee (Election Safety + Log Matching): a leader can only be elected if it has all committed entries. A committed entry required a majority quorum to acknowledge. That majority shares at least one member with any future election's majority. The `RequestVote` logic checks that the candidate's last log entry has a term >= the voter's and an index >= the voter's. Therefore, any future leader will have all entries that were acknowledged by a majority — those entries are committed. Uncommitted entries (in the leader's log but not yet on a majority) may be rolled back when a new leader takes over, but this is safe because they were never acknowledged to the client.

**17. How does etcd's watch resumption work after a network partition, and what revision-level mechanism ensures no watch events are missed?**

When a client's etcd connection is interrupted (network partition or timeout), the etcd gRPC stream closes. The client (inside the apiserver's watch loop) records the last received revision. On reconnect, it sends a new `Watch` RPC requesting events starting from `lastReceivedRevision + 1`. etcd checks whether its watch cache (or compacted history) contains events since that revision. If yes, it replays the missed events in order and then begins streaming new events. If the revision is below the compaction point, the server returns a `CompactRevision` error, and the client must relist. The revision counter itself is the sequence that guarantees no events are skipped: since etcd uses monotonically increasing revision numbers for every write, scanning from `lastRevision + 1` is a mathematically complete coverage of all intermediate writes, assuming they have not been compacted.

**18. What happens inside etcd when the cluster loses quorum but one member is still running — what operations succeed and fail, and at what layer?**

The surviving member's Raft state machine enters a state where it can receive `AppendEntries` from itself (as a follower that cannot find a leader) or from stale pre-failure messages, but it cannot commit new entries without a quorum. `Put` operations sent to the surviving member are rejected with `etcdserver: leader changed or not ready`. The member continues serving `Range` (read) requests, but only in serializable mode — linearizable reads fail because the member cannot confirm it is still the leader (there is no leader). The member's bbolt state remains readable. `Watch` streams continue to be served from the in-memory watch cache for as long as the member is running. No new watch events are emitted because no new entries are committed. From the apiserver's perspective, write RPCs begin failing and the apiserver starts returning 5xx errors to clients for write operations, while serving reads from its own watch cache (which stops updating).

**19. Describe how etcd's lease mechanism works and how Kubernetes uses leases both for node heartbeats and controller leader election.**

etcd leases are server-side TTL objects with unique IDs. A lease is created with a TTL; all keys attached to the lease expire together when the TTL elapses unless renewed. Kubernetes uses etcd leases internally via the `coordination.k8s.io/v1 Lease` Kubernetes API object (stored in etcd like any other object). Node heartbeats: the kubelet updates `spec.renewTime` in a `Lease` object in the `kube-node-lease` namespace every 10 seconds. The node lifecycle controller checks if `renewTime + leaseDurationSeconds` has elapsed without renewal; if so, it marks the node `NotReady` and applies `not-ready:NoExecute` taint. Controller leader election: the controller-manager and scheduler compete to hold a `Lease` object in `kube-system`. The holder updates `renewTime` on every reconcile cycle. If a holder crashes, its `renewTime` stops updating. After `leaseDurationSeconds` (default 15s), a standby acquires the Lease by performing a CAS write on the `holderIdentity` and `acquireTime` fields — using etcd's optimistic concurrency to ensure only one acquirer wins.

**20. How would you design an etcd cluster for a Kubernetes control plane that must tolerate a full availability zone failure with no downtime and minimal write latency increase?**

A 5-member cluster spanning 3 AZs (2+2+1 distribution) tolerates any single AZ failure while maintaining quorum (3 of 5 members remain). Write latency: the leader waits for 3 acknowledgments (majority of 5). If 2 of the 3 AZs contain 2 members each, 1 cross-AZ round trip is always needed for quorum — acceptable at <2ms intra-region latency but significant cross-region (50+ ms). Alternative: 3-member cluster with 1 member per AZ. Tolerates 1 AZ failure with write quorum from 2 remaining. Write latency: 1 cross-AZ round trip to any 1 follower. Simpler and faster than 5-member for single-AZ-failure tolerance. For the leader placement: don't pin the leader to a specific AZ; let Raft elect naturally. Use `--initial-cluster-state=existing` for maintenance. Disk: dedicated NVMe per member, separate from OS. Separate WAL disk if available (different I/O path reduces contention). Place etcd on dedicated nodes with CPU/memory isolation from workloads. Set `--quota-backend-bytes=8589934592` (8 GiB) and monitor `etcd_mvcc_db_total_size_in_bytes` with an alert at 80%.

---

## Hands-On Labs

### Lab 1: Observe etcd in a Running Cluster

**Objective:** Read Kubernetes objects directly from etcd and understand the storage format.

**Setup:** A kubeadm or kind cluster with etcd accessible.

**Tasks:**
1. Set up etcdctl with cluster TLS: `export ETCDCTL_API=3 ETCDCTL_ENDPOINTS=... ETCDCTL_CACERT=... ETCDCTL_CERT=... ETCDCTL_KEY=...`
2. List all keys: `etcdctl get /registry --prefix --keys-only | wc -l`.
3. Read a pod: `etcdctl get /registry/pods/default/$(kubectl get pod -o name | head -1 | cut -d/ -f2)`. Observe the protobuf binary.
4. Check cluster health and leader: `etcdctl endpoint status --cluster --write-out=table`.
5. Watch for changes while creating a pod in another terminal: `etcdctl watch /registry/pods/default --prefix`.

**Expected outcome:** Direct visibility into etcd's storage and the watch stream that feeds all Kubernetes control loops.

### Lab 2: Simulate etcd Failure and Recovery

**Objective:** Experience quorum loss and recovery in a safe environment.

**Setup:** A 3-member kind cluster or manual 3-node etcd setup.

**Tasks:**
1. Confirm 3-member health.
2. Stop one member: `docker stop <etcd-member-2-container>`.
3. Verify cluster still works: create a pod. Confirm quorum with 2 members.
4. Stop a second member: `docker stop <etcd-member-3-container>`.
5. Observe: `kubectl create pod` hangs (write quorum lost). `kubectl get pods` works (read from apiserver cache).
6. Restart both members. Observe cluster recovery.

**Expected outcome:** Visceral understanding of quorum — two failures cause a complete write outage while reads continue from memory.

### Lab 3: Backup and Restore

**Objective:** Practice the complete etcd disaster recovery workflow.

**Setup:** A kubeadm cluster with etcd on the control-plane node.

**Tasks:**
1. Create several test namespaces and deployments.
2. Take a snapshot: `etcdctl snapshot save /tmp/backup.db && etcdctl snapshot status /tmp/backup.db`.
3. Note current resource count: `kubectl get all -A | wc -l`.
4. Delete several namespaces: `kubectl delete ns test1 test2`.
5. Stop the apiserver (move static pod manifest): `mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/`.
6. Stop etcd. Restore: `etcdctl snapshot restore /tmp/backup.db --data-dir=/var/lib/etcd-new`.
7. Swap data directories. Restart etcd. Restore apiserver manifest.
8. Verify: `kubectl get ns test1 test2` — they are back.

**Expected outcome:** Confidence in the backup/restore procedure and understanding of what a restore actually recovers.

---

## Production Incidents

### Incident 1: etcd Disk Saturation Causes Cascading Control-Plane Outage

**Symptom:** At 14:00 UTC, apiserver latency for write operations spikes from 50ms to 15s p99. New pods fail to schedule. At 14:07, the first etcd leader election occurs. At 14:09, a second election. By 14:12, the cluster is effectively unavailable for writes. `kubectl get pods` still works.

**Investigation:** `etcdctl endpoint status --cluster` shows split responses — one member unreachable, leader changing. `etcd_disk_wal_fsync_duration_seconds` p99 spiked to 800ms at 13:58. Node inspection: a backup job writing to the OS disk started at 13:55. The etcd data directory is on the OS disk. Backup I/O saturated the disk (100% util, 500ms await), causing WAL fsyncs to back up. The leader couldn't send heartbeats within the election timeout. Followers elected new leaders repeatedly.

**Root cause:** etcd data directory co-located on the same disk as the OS, application logs, and a backup agent. Heavy backup I/O caused disk saturation, WAL fsync starvation, and cascading leader elections.

**Recovery:** Stop the backup job, disk utilization drops, WAL fsyncs recover, leader stabilizes, cluster health restored in 3 minutes. Long-term: move etcd to a dedicated SSD with no other workloads.

**Prevention:** Dedicate a separate NVMe disk for etcd. Alert on `etcd_disk_wal_fsync_duration_seconds` p99 > 25ms. Add `vm.dirty_ratio` tuning to prevent OS page cache from competing with etcd. Schedule backup jobs in off-peak windows and use a dedicated backup disk.

### Incident 2: etcd DB Quota Exceeded, Cluster Frozen

**Symptom:** All write operations cluster-wide fail with "etcdserver: mvcc: database space exceeded." New pods cannot be created, existing workloads continue running.

**Investigation:** `etcdctl endpoint status --write-out=table` shows `IS ALARMED: true`, `ALARM: NOSPACE`. DB size: 8.1 GiB against the default `--quota-backend-bytes=8589934592` (8 GiB). etcd enters a read-only alarm state and rejects all writes. Investigation of DB size: high event object churn (Kubernetes Events have a 1-hour TTL, but a misconfigured HPA was generating 50 events/second for weeks, creating millions of event objects that filled the DB).

**Recovery:** 
1. Defragment immediately (reduces file size): `etcdctl defrag --endpoints=...` (one at a time). DB size drops from 8.1 GiB to 2.1 GiB (95% was fragmentation and old revisions).
2. Compact: `etcdctl compact $(etcdctl endpoint status --write-out=json | python3 -c "...")`.
3. Clear the alarm: `etcdctl alarm disarm`. Writes resume.
4. Fix the HPA configuration causing event storms.

**Prevention:** Set `--quota-backend-bytes` explicitly to a known limit (e.g., 4 GiB) to prevent silent growth. Alert on `etcd_mvcc_db_total_size_in_bytes > 3Gi` (75% of 4 GiB limit). Schedule regular compaction and defragmentation. Limit Events per object via `--event-ttl` and API Priority and Fairness limits on Event creates.
