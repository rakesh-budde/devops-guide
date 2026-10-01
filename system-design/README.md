# System Design — Interview Preparation Guide

> **Target audience:** Senior DevOps Engineers, SREs, and Platform Engineers preparing for FAANG-level system design interviews.
>
> **Scope:** Distributed systems taught from first principles — the interview framework, scalability math, building blocks (LB/cache/CDN/queues), data storage and sharding, scalability patterns (microservices, event-driven, CQRS, saga), worked case studies, and the reliability/trade-off reasoning that separates a hire from a no-hire. Every section is **interview-focused**: internals, trade-offs, failure modes, and probing Q&A — no filler.

---

## 🗺️ Visual Overview

**In one line:** A scalable system is a request flowing through DNS → CDN → load balancer → stateless app tier → cache → database, with queues for async work — master each hop and every design question becomes a composition of the same building blocks.

```mermaid
mindmap
  root((System Design))
    Framework
      Requirements functional and non functional
      Back of envelope estimation
      High level design
      Deep dive and bottlenecks
      Trade offs and wrap up
    Building Blocks
      DNS and reverse proxy
      Load balancers
      Caching and CDN
      Message queues
      API gateway
    Data Storage
      SQL vs NoSQL
      Replication
      Sharding and partitioning
      Indexing
      Quorums and consistency
    Scalability Patterns
      Microservices
      Event driven
      CQRS
      Rate limiting
      Idempotency
      Saga
    Case Studies
      URL shortener
      News feed
      Chat messaging
      Rate limiter
    Reliability
      Failure handling
      Redundancy
      Multi region
      Disaster recovery
      Observability
```

**Scalable web architecture — the reference request path** (memorize this left-to-right flow):

```mermaid
flowchart LR
    U["👤 Client<br/>Browser / App"] --> CDN["🌍 CDN<br/>static + edge cache"]
    CDN --> LB["⚖️ Load Balancer<br/>round robin / least conn"]
    LB --> A1["🖥️ App Server 1"]
    LB --> A2["🖥️ App Server 2"]
    LB --> A3["🖥️ App Server 3"]
    A1 --> C["⚡ Cache<br/>Redis"]
    A2 --> C
    A3 --> C
    C -->|"miss"| DBP["🗄️ Primary DB<br/>writes"]
    DBP -->|"replicate"| DBR["🗄️ Read Replicas<br/>reads"]
    A1 -.->|"async"| MQ["📨 Message Queue<br/>Kafka / SQS"]
    MQ --> W["🛠️ Workers"]
    class U,CDN start
    class LB ctrl
    class A1,A2,A3,W proc
    class C,DBP,DBR store
    class MQ ctrl
    classDef start fill:#e3f2fd,stroke:#1565c0,color:#0d47a1,stroke-width:2px;
    classDef proc fill:#fff9c4,stroke:#f9a825,color:#000,stroke-width:2px;
    classDef good fill:#c8e6c9,stroke:#2e7d32,color:#1b5e20,stroke-width:2px;
    classDef bad fill:#ffcdd2,stroke:#c62828,color:#b71c1c,stroke-width:2px;
    classDef ctrl fill:#e1bee7,stroke:#6a1b9a,color:#4a148c,stroke-width:2px;
    classDef store fill:#ffe0b2,stroke:#e65100,color:#000,stroke-width:2px;
```

**The five interview phases — drive the conversation in this order** (never draw boxes before stating scale):

```mermaid
flowchart LR
    S1["📋 1. Requirements<br/>3-5 min<br/>functional +<br/>non-functional"] --> S2["🔢 2. Estimation<br/>2-3 min<br/>users, RPS,<br/>storage, bandwidth"]
    S2 --> S3["🏗️ 3. High-Level Design<br/>10-15 min<br/>components, data flow,<br/>APIs, storage"]
    S3 --> S4["🔬 4. Deep Dive<br/>15-20 min<br/>scale, failures,<br/>bottlenecks, security"]
    S4 --> S5["✅ 5. Wrap Up<br/>3-5 min<br/>trade-offs +<br/>future work"]
    class S1 start
    class S2 proc
    class S3 ctrl
    class S4 store
    class S5 good
    classDef start fill:#e3f2fd,stroke:#1565c0,color:#0d47a1,stroke-width:2px;
    classDef proc fill:#fff9c4,stroke:#f9a825,color:#000,stroke-width:2px;
    classDef good fill:#c8e6c9,stroke:#2e7d32,color:#1b5e20,stroke-width:2px;
    classDef bad fill:#ffcdd2,stroke:#c62828,color:#b71c1c,stroke-width:2px;
    classDef ctrl fill:#e1bee7,stroke:#6a1b9a,color:#4a148c,stroke-width:2px;
    classDef store fill:#ffe0b2,stroke:#e65100,color:#000,stroke-width:2px;
```

> 🧠 **Memory hooks (mnemonics):**
> - **Design framework — "REBHW":** *"Real Engineers Build High Walls"* → **R**equirements → **E**stimation → **B**ase (high-level) design → **H**andle deep-dive → **W**rap-up trade-offs.
> - **CAP — "you can't have your CAP and eat it":** Partition tolerance is **mandatory** on a network, so you really pick **C or A**.
> - **Scale out, not up:** stateless app tiers scale **horizontally forever**; the database is almost always the real bottleneck — cache reads, replicate reads, shard writes.
> - **Request path — "Do Cats Love Any Cake Daily":** **D**NS → **C**DN → **L**oad balancer → **A**pp → **C**ache → **D**atabase.

---

## 📚 Master Table of Contents

| # | Section | File | Est. study time |
|---|---------|------|-----------------|
| 1 | **Fundamentals** — framework, estimation, latency vs throughput, CAP/PACELC, consistency models, availability math | [01-FUNDAMENTALS.md](01-FUNDAMENTALS.md) | 2.5 h |
| 2 | **Building Blocks** — load balancers, caching, CDN, message queues, API gateway, reverse proxy, DNS | [02-BUILDING-BLOCKS.md](02-BUILDING-BLOCKS.md) | 3 h |
| 3 | **Data & Storage** — SQL vs NoSQL, replication, sharding, partitioning, indexing, quorums, CAP in practice | [03-DATA-STORAGE.md](03-DATA-STORAGE.md) | 3 h |
| 4 | **Scalability Patterns** — microservices, event-driven, CQRS, rate limiting, idempotency, backpressure, saga | [04-SCALABILITY-PATTERNS.md](04-SCALABILITY-PATTERNS.md) | 2.5 h |
| 5 | **Case Studies** — URL shortener, news feed, chat/messaging, distributed rate limiter (full walkthroughs) | [05-CASE-STUDIES.md](05-CASE-STUDIES.md) | 4 h |
| 6 | **Trade-offs & Reliability** — failure handling, redundancy, multi-region, DR, observability, interview checklist | [06-TRADEOFFS-RELIABILITY.md](06-TRADEOFFS-RELIABILITY.md) | 2.5 h |

---

## 🧭 Suggested Study Order

1. **Start with [Fundamentals](01-FUNDAMENTALS.md)** — the framework and estimation vocabulary you'll use in every single interview. Memorize the latency ladder and the CAP trade-off here.
2. **Learn the [Building Blocks](02-BUILDING-BLOCKS.md)** — load balancers, caches, CDNs, and queues are the Lego bricks every design is assembled from.
3. **Go deep on [Data & Storage](03-DATA-STORAGE.md)** — the database is where almost every system bottlenecks; sharding and replication are the highest-value topics.
4. **Then [Scalability Patterns](04-SCALABILITY-PATTERNS.md)** — microservices, event-driven, CQRS, and saga show how to decompose and decouple at scale.
5. **Apply it in [Case Studies](05-CASE-STUDIES.md)** — this is where it all comes together; practice the Requirements → Estimation → Design → Deep-dive → Trade-offs flow out loud.
6. **Finish with [Trade-offs & Reliability](06-TRADEOFFS-RELIABILITY.md)** — failure handling and multi-region reasoning are the deep-dive questions that decide senior/staff offers; best reviewed last and before interviews.

> 💡 **How to use each section:** Every file opens with a **Visual Overview** (mind map + colorful diagrams + memory hooks) — skim it first and revisit it last. Topics carry an **Interview weight** marker and an **In one line** summary. Each closes with **Interview Questions & Answers** (answer → reasoning → follow-up).

---

## 🎯 What Makes This Interview-Focused

- **Framework over guessing** — you'll drive every design from requirements → estimation → design → deep-dive → trade-offs, the exact structure interviewers score against.
- **Trade-offs & failure modes** — every topic covers when *not* to use something and how it breaks under partition, load, and region failure.
- **Colorful Mermaid diagrams** for the hardest flows (request path, sharded+replicated DB, write strategies, case-study architectures).
- **Memory hooks** (mnemonics) so numbers and patterns actually stick under interview pressure.

---

**[← Back to Main README](../README.md)** | **[Start: Fundamentals →](01-FUNDAMENTALS.md)**
