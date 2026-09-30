# Section 2: Kubernetes Architecture

This section explains Kubernetes as a distributed, declarative control system. Every design choice — from the centralized API server to the level-triggered reconciliation loops and the watch-based informer pattern — exists to handle the unavoidable reality that distributed systems fail partially, frequently, and in unexpected orders. Understanding these design principles is what separates a candidate who can use Kubernetes from one who can operate, extend, and debug it at scale.

## Subtopic Index

- [History and Evolution](#history-and-evolution)
- [Design Principles](#design-principles)
- [Declarative Architecture](#declarative-architecture)
- [Desired State Model](#desired-state-model)
- [Control Loops](#control-loops)
- [Reconciliation](#reconciliation)
- [Control Plane](#control-plane)
- [Data Plane](#data-plane)
- [Worker Nodes](#worker-nodes)
- [Request Lifecycle](#request-lifecycle)
- [Cluster Startup Sequence](#cluster-startup-sequence)
- [Component Interactions](#component-interactions)

---

## 🗺️ Visual Overview

**Mind map — the whole section at a glance** (skim first, revisit last):

```mermaid
mindmap
  root((Kubernetes Architecture))
    History
      Google Borg
      Google Omega
      Open sourced 2014
      CNCF 2016
    Design Principles
      Declarative desired state
      Level triggered reconciliation
      API first loose coupling
    Declarative Model
      Spec is desired
      Status is observed
      Controllers close the diff
      Idempotent and self healing
    Control Loops
      Informer watches API
      Work queue dedup and rate limit
      Reconcile the key
      Requeue with backoff
    Reconciliation
      Compute the diff
      Apply changes idempotently
      Finalizers for cleanup
      Optimistic concurrency 409
    Control Plane
      kube apiserver
      etcd Raft store
      kube scheduler
      controller manager
      cloud controller manager
    Data Plane
      Worker nodes
      Container runtime
      kube proxy
      CNI and CSI
      Static stability
    Request Lifecycle
      TLS then authn
      Authorization RBAC
      APF fairness
      Admission webhooks
      etcd write and watch
    Cluster Startup
      etcd first
      apiserver next
      controllers and scheduler
      kubelet and CNI
      CoreDNS and kube proxy
    Component Interactions
      All talk through apiserver
      No direct component calls
      Watch based coordination
```

**The control loop — the single most important idea in Kubernetes** (every controller is this cycle):

```mermaid
flowchart LR
    A["👀 Informer<br/>watches API server"]:::start --> B["📥 Work Queue<br/>dedup + rate-limit"]:::store
    B --> C["⚙️ Reconcile the key<br/>read from cache"]:::proc
    C --> D{"🔍 Desired == Actual?"}:::proc
    D -->|"yes ✅"| E["😌 No-op<br/>converged"]:::good
    D -->|"no ⚠️"| F["🔧 Create / update / delete<br/>via API server"]:::proc
    F --> G["📝 Update status"]:::store
    G --> A
    F -->|"error 🔁"| H["⏱️ Requeue<br/>exponential backoff"]:::bad
    H --> B
    classDef start fill:#e3f2fd,stroke:#1565c0,color:#0d47a1,stroke-width:2px;
    classDef proc fill:#fff9c4,stroke:#f9a825,color:#000,stroke-width:2px;
    classDef good fill:#c8e6c9,stroke:#2e7d32,color:#1b5e20,stroke-width:2px;
    classDef bad fill:#ffcdd2,stroke:#c62828,color:#b71c1c,stroke-width:2px;
    classDef ctrl fill:#e1bee7,stroke:#6a1b9a,color:#4a148c,stroke-width:2px;
    classDef store fill:#ffe0b2,stroke:#e65100,color:#000,stroke-width:2px;
```

**The API request gauntlet — each phase has its own failure code**:

```mermaid
flowchart TB
    A["📥 Client HTTPS request"]:::start --> B["🔐 TLS handshake"]:::proc
    B --> C{"🪪 Authentication"}:::proc
    C -->|"none match ❌"| E401["401 Unauthorized"]:::bad
    C -->|"identity ✅"| D{"🛡️ Authorization RBAC"}:::proc
    D -->|"denied ❌"| E403["403 Forbidden"]:::bad
    D -->|"allowed ✅"| F{"🚦 APF fairness"}:::proc
    F -->|"overloaded ❌"| E429["429 Too Many Requests"]:::bad
    F -->|"admitted ✅"| G["🧬 Mutating webhooks"]:::ctrl
    G --> H{"🧪 Defaulting + validation"}:::proc
    H -->|"invalid ❌"| E422["422 Invalid"]:::bad
    H -->|"valid ✅"| I["✔️ Validating webhooks"]:::ctrl
    I --> J["💾 etcd write<br/>CAS on resourceVersion"]:::store
    J -->|"conflict ❌"| E409["409 Conflict"]:::bad
    J -->|"success ✅"| K["📡 Emit watch event<br/>+ 201 / 200 response"]:::good
    classDef start fill:#e3f2fd,stroke:#1565c0,color:#0d47a1,stroke-width:2px;
    classDef proc fill:#fff9c4,stroke:#f9a825,color:#000,stroke-width:2px;
    classDef good fill:#c8e6c9,stroke:#2e7d32,color:#1b5e20,stroke-width:2px;
    classDef bad fill:#ffcdd2,stroke:#c62828,color:#b71c1c,stroke-width:2px;
    classDef ctrl fill:#e1bee7,stroke:#6a1b9a,color:#4a148c,stroke-width:2px;
    classDef store fill:#ffe0b2,stroke:#e65100,color:#000,stroke-width:2px;
```

**Control plane vs data plane — the brain decides, the muscles run**:

```mermaid
flowchart TB
    subgraph CP["🧠 Control Plane — decides"]
        API["🚪 kube-apiserver"]:::ctrl
        ETCD["💾 etcd Raft store"]:::store
        SCHED["📌 kube-scheduler"]:::ctrl
        CM["🔄 controller-manager"]:::ctrl
        CCM["☁️ cloud-controller-manager"]:::ctrl
    end
    subgraph DP["💪 Data Plane — runs workloads"]
        KUBELET["🤖 kubelet"]:::proc
        RUNTIME["📦 containerd"]:::proc
        PROXY["🕸️ kube-proxy"]:::proc
        PODS["🚀 Pods"]:::good
    end
    API <--> ETCD
    SCHED --> API
    CM --> API
    CCM --> API
    KUBELET --> API
    KUBELET --> RUNTIME
    RUNTIME --> PODS
    PROXY --> PODS
    classDef start fill:#e3f2fd,stroke:#1565c0,color:#0d47a1,stroke-width:2px;
    classDef proc fill:#fff9c4,stroke:#f9a825,color:#000,stroke-width:2px;
    classDef good fill:#c8e6c9,stroke:#2e7d32,color:#1b5e20,stroke-width:2px;
    classDef bad fill:#ffcdd2,stroke:#c62828,color:#b71c1c,stroke-width:2px;
    classDef ctrl fill:#e1bee7,stroke:#6a1b9a,color:#4a148c,stroke-width:2px;
    classDef store fill:#ffe0b2,stroke:#e65100,color:#000,stroke-width:2px;
```

> 🧠 **Memory hooks (mnemonics):**
> - **Request lifecycle order:** *"Tall Aunts Always Admit Everyone Willingly"* → **T**LS → **A**uthn → **A**uthz → **A**PF → **A**dmission → **E**tcd → **W**atch.
> - **Startup order:** *"Eight Angry Cats Sleep Kindly Chasing Dogs Playfully"* → **E**tcd, **A**piserver, **C**ontroller-manager, **S**cheduler, **K**ubelet, **C**NI, **D**NS (CoreDNS), **P**roxy.
> - **Three design principles:** *"Declare, Reconcile, Decouple"* → declarative desired state, level-triggered reconciliation, API-first loose coupling.
> - **Spec vs Status:** *"Spec is the wish, Status is the truth."*
> - **Static stability:** *"The data plane keeps dancing even when the brain sleeps."*

---

## History and Evolution

> 🎯 **Interview weight: Low** — know the Borg/Omega lineage and the "why portable API" story; don't memorize version numbers.

**In one line:** Kubernetes is Google's third-generation cluster manager — distilling ~15 years of Borg/Omega lessons into a clean, versioned, cloud-portable API.

**Where it came from:** open-sourced by Google in June 2014 (announced at DockerCon), Kubernetes draws directly from two internal systems:

- **Borg** — production orchestration for nearly everything at Google. Its core lesson: at scale, treat the cluster as a **unified resource pool managed by automation**, not a herd of individually administered machines.
- **Omega** — a research redesign that introduced **shared-state scheduling**.

**What Borg contributed** (the DNA you still see today):

| Borg concept | Kubernetes descendant |
|---|---|
| Labels on tasks/jobs | Labels + selectors |
| Health-check & replace | Self-healing controllers |
| Resource classes | Requests & limits |
| Alloc | Pod |

**What Kubernetes added on top:** a clean **versioned REST/HTTP API**, a pluggable **extensibility model** (CRDs, admission webhooks, custom controllers), and **cloud portability** from day one — a direct answer to the vendor lock-in fears of the Docker era.

**The milestones that matter:**

| Version / Year | Milestone |
|---|---|
| July 2015 (1.0) | Core architecture set: apiserver, etcd, scheduler, controller-manager, kubelet |
| 2016 | CNCF founding project |
| 1.6 → 1.8 | RBAC introduced then stabilized |
| 1.7 | CRDs |
| 1.9 | Admission webhooks |
| 1.16 | Server-side apply |
| 1.24 | dockershim removed → CRI standardization complete |

> 💡 **Interview tip:** If asked "why is Kubernetes so extensible?", tie it to the lock-in lesson — Google deliberately built a provider-neutral API so workloads could move across clouds.

---

## Design Principles

> 🎯 **Interview weight: High** — these three principles explain almost every "why does Kubernetes behave like this?" question.

**In one line:** Three principles — **declarative desired state**, **level-triggered reconciliation**, and **API-first loose coupling** — explain every non-obvious design decision.

🧠 **Mental model:** You *declare a wish*, controllers *continuously close the gap*, and *everyone coordinates through one shared switchboard* (the API server).

**The three principles:**

- **Declarative desired state** — users express *what* they want to be true (six replicas), not *how* to achieve it. The manifest becomes an API object stored durably in etcd; controllers turn that declaration into real-world actions.
- **Level-triggered reconciliation** — every controller *periodically compares current vs desired state* and closes the gap, regardless of how many events fired or whether some were lost. Contrast with edge-triggered systems (webhooks, imperative scripts) that react to *transitions* — miss one event (crash, partition, queue overflow) and they never recover. Level-triggered is inherently **self-healing**: after a restart it re-reads everything and reconciles from scratch.
- **API-first loose coupling** — every component talks *only* through the versioned API server, never directly to another component. The scheduler doesn't call kubelet — it writes a Binding. A controller doesn't call the runtime — it creates Pod objects. So any component can be restarted, scaled, or replaced independently.

> ⚠️ **Gotcha:** These principles explain the confusing bits:
> - A successful `kubectl apply` does **not** mean the workload is running — you stored *desired* state, not *achieved* state.
> - Kubernetes is **eventually consistent** — controllers run asynchronously.
> - Kubernetes survives component failures — the loop re-reconciles from durable state.

---

## Declarative Architecture

> 🎯 **Interview weight: High** — the spec-vs-status contract underpins all of operations and monitoring.

**In one line:** You describe the *target* state; the system figures out how to get there from wherever it currently is.

**Imperative vs declarative:**

| | Imperative | Declarative (Kubernetes) |
|---|---|---|
| You provide | Step-by-step actions | Target state |
| On failure | You must know current state to resume | System recomputes the diff and continues |
| Idempotency | Hard | Built-in |

**How Kubernetes models it** — every object carries two halves:

- **`spec`** — *what you want* (written by users/operators).
- **`status`** — *what the system observed* (written by controllers/kubelet).

A controller reads both, computes the **diff**, and takes actions to close it. Because controllers are **idempotent**, the architecture tolerates retries, duplicate events, and concurrent reconciliation.

> ⚠️ **Gotcha:** A resource in etcd with a valid spec does **not** mean the app is running. `spec.replicas: 6` is a *desire*; `status.readyReplicas` is the *fact*. Alert on `readyReplicas < spec.replicas` sustained over time — not just on API errors.

```yaml
# Example: desired vs observed separation
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payments
  generation: 3              # incremented on spec change
spec:
  replicas: 6                # DESIRED STATE — user writes this
  template:
    spec:
      containers:
      - name: app
        image: payments:v3
status:
  observedGeneration: 3      # controller has processed this generation
  replicas: 6                # total pod count
  readyReplicas: 4           # pods passing readiness probe ← actual state
  updatedReplicas: 6
  conditions:
  - type: Available
    status: "True"
```

---

## Desired State Model

> 🎯 **Interview weight: Medium** — "eventually" and `observedGeneration` are common follow-ups.

**In one line:** A correctly built controller will *eventually* converge the cluster to desired state after any single failure — including its own crash.

**Why "eventually" is the key word:** convergence is not instantaneous. A controller restart, a slow etcd write, a slow CNI plugin, or a node restart all delay it. The system is always *making progress* toward desired state — it just may not be there yet at any given instant.

🔍 **Under the hood — tracking progress with generations:**

- Each `spec` change increments `metadata.generation`.
- When the controller processes that spec, it sets `status.observedGeneration` to match.
- If `observedGeneration < generation`, the controller hasn't yet acted on the latest spec.

> 💡 **Interview tip:** Poll rollout progress with **condition checks** (`observedGeneration == generation`), never time-based `sleep`.

**OwnerReferences & garbage collection:** ownership forms a hierarchy — a Deployment owns ReplicaSets, a ReplicaSet owns Pods (each child's `ownerReference` points up). Deleting a Deployment **cascades** deletion down to ReplicaSets then Pods (foreground or background mode). **Finalizers** can block deletion until pre-deletion cleanup completes.

---

## Control Loops

> 🎯 **Interview weight: High** — informer → work queue → reconcile is *the* controller design question.

**In one line:** Every controller runs a feedback loop — measure current state, compare to desired, act to shrink the gap, repeat.

**Two everyday examples:**

- **Deployment controller** — list owned ReplicaSets → compare pod count to desired replicas → create/scale/delete → update status.
- **kubelet** — list pods assigned to this node → compare to running containers → start/stop/restart → update pod status.

🔍 **Under the hood — the work queue is the heart of the loop:** an informer event does **not** act immediately. The handler enqueues a key (`namespace/name`) into a rate-limited work queue; worker goroutines dequeue keys and call `Reconcile(key)`. The queue gives you:

- **Deduplication** — rapid events for the same object collapse to one, so the reconciler always sees the *latest* state.
- **Rate limiting** — a thrashing controller can't overwhelm the API server.
- **Backoff** — a failed reconcile is requeued with exponential backoff.

```mermaid
flowchart TD
    I["👀 Informer<br/>watches API server"]:::start --> H["📨 Event handler<br/>Add / Update / Delete"]:::proc
    H --> Q["📥 Work Queue<br/>dedup + rate-limit"]:::store
    Q --> W["🔧 Worker goroutine<br/>dequeues key"]:::proc
    W --> R["⚙️ Reconcile the key"]:::proc
    R --> L["📖 Read object from lister cache"]:::proc
    L --> C["🧮 Compute desired children"]:::proc
    C --> A["🚀 Create / update / delete via API"]:::good
    A --> S["📝 Update status"]:::store
    R -->|"error 🔁"| Q
    classDef start fill:#e3f2fd,stroke:#1565c0,color:#0d47a1,stroke-width:2px;
    classDef proc fill:#fff9c4,stroke:#f9a825,color:#000,stroke-width:2px;
    classDef good fill:#c8e6c9,stroke:#2e7d32,color:#1b5e20,stroke-width:2px;
    classDef bad fill:#ffcdd2,stroke:#c62828,color:#b71c1c,stroke-width:2px;
    classDef ctrl fill:#e1bee7,stroke:#6a1b9a,color:#4a148c,stroke-width:2px;
    classDef store fill:#ffe0b2,stroke:#e65100,color:#000,stroke-width:2px;
```

<sub>ASCII reference of the same loop:</sub>

```
     Informer (watches API server)
          │
          ▼ AddFunc / UpdateFunc / DeleteFunc
     Work Queue  ←───── deduplication, rate-limit
          │
          ▼ worker goroutine dequeues key
     Reconcile(namespace/name)
       ├─ read object from lister (cache)
       ├─ compute desired children
       ├─ create / update / delete via API
       └─ update status
```

---

## Reconciliation

> 🎯 **Interview weight: High** — idempotency + optimistic concurrency (409) are frequent deep-dives.

**In one line:** Reconciliation computes and applies the *diff* between actual and desired state — and it must be idempotent because Kubernetes guarantees **at-least-once**, not exactly-once, event delivery.

🧠 **Mental model:** A reconciler is a pure function of `(desired, actual) → actions`. Run it twice with the same inputs and nothing extra happens.

**The standard reconcile pattern:**

1. Read the owner object (e.g., Deployment) from the lister cache.
2. If the object has a `DeletionTimestamp`, perform cleanup (e.g., remove external resources) and remove the finalizer. Return.
3. Add a finalizer if not present (for cleanup on deletion).
4. List all child objects (e.g., ReplicaSets) with owner references pointing to this object.
5. Compute the desired child state.
6. For each child: if it should exist and doesn't, create it. If it exists but needs updating, patch it. If it shouldn't exist, delete it.
7. Update the parent object's status based on observed children.

> 🔍 **Under the hood — optimistic concurrency:** updates carry a `resourceVersion`. If two controller replicas (or a restart) both write the same object, only one wins; the other gets **`409 Conflict`** and requeues. This prevents split-brain updates *without* distributed locking.

---

## Control Plane

> 🎯 **Interview weight: High** — you must be able to name every component and its one job.

**In one line:** The control plane is the set of processes that *decide* — serving the API, persisting state, scheduling, and running reconciliation loops.

**Where it runs:** on self-managed clusters, as **static Pods** on control-plane nodes (kubelet reads `/etc/kubernetes/manifests/`). On managed clusters (EKS/AKS/GKE), the cloud provider runs and hides it — you get no node access.

**The components at a glance:**

| Component | One job | Key facts |
|---|---|---|
| **kube-apiserver** | Front door + only etcd client | Stateless; any replica serves any request; does authn/authz/admission + version conversion + watch delivery |
| **etcd** | Durable, consistent state store | Raft consensus; apiserver is its *only* client |
| **kube-scheduler** | Place pods on nodes | Watches pods with no `spec.nodeName`, filters + scores, writes a Binding; leader-elected |
| **kube-controller-manager** | Run ~30 built-in loops | Deployment, ReplicaSet, StatefulSet, Job, Node, Endpoint…; leader-elected |
| **cloud-controller-manager** | Cloud-specific glue | LoadBalancer provisioning, cloud routes, node lifecycle |

> 🔍 **Under the hood:** the apiserver is **stateless** — all durable state lives in etcd, which is why you scale it horizontally for throughput and HA, and why any request can hit any replica.

```
Control Plane Node
┌─────────────────────────────────────────────────────────┐
│  kube-apiserver  (port 6443, HTTPS)                     │
│  ↕ only component touching etcd                         │
│  etcd  (port 2379/2380)                                 │
│  kube-scheduler  (leader-elected)                       │
│  kube-controller-manager  (leader-elected, ~30 loops)   │
│  cloud-controller-manager  (cloud-specific)             │
└─────────────────────────────────────────────────────────┘
```

---

## Data Plane

> 🎯 **Interview weight: High** — "static stability" is a favorite senior-level question.

**In one line:** The data plane *runs the workloads and carries their traffic* — and is designed to keep serving even if the control plane is entirely down.

**What it contains:** worker nodes, container runtimes, network (CNI) plugins, and storage (CSI) plugins.

🧠 **Mental model — static stability:** the brain (control plane) can sleep; the muscles (data plane) keep working.

**What keeps running during a full control-plane outage:**

- ✅ Running containers keep executing.
- ✅ kube-proxy's iptables/IPVS rules stay in place → Service traffic still routes.
- ✅ CNI routes / eBPF maps stay installed.
- ✅ CSI-mounted volumes stay accessible.

**What stops:**

- ❌ New pod scheduling.
- ❌ Restarting/replacing crashed pods (kubelet can't fetch new spec).
- ❌ Status updates.
- ❌ Secret/configmap rolling updates.
- ❌ HPA/autoscaler responses.

> ⚠️ **Gotcha:** A cluster of 10,000 pods survives a 30-minute apiserver outage fine — *but* any pod that crashes can't restart, short-lived tokens may expire (breaking apiserver calls), and the HPA can't absorb a traffic spike. This "blast radius" reasoning is exactly what you use to plan control-plane upgrades.

---

## Worker Nodes

> 🎯 **Interview weight: Medium** — node registration + heartbeat mechanics show up in troubleshooting rounds.

**In one line:** A worker node is a compute host that registers with the cluster and advertises its capacity so the scheduler can place pods on it.

**What runs on it:** kubelet, a container runtime (containerd), kube-proxy (or an eBPF dataplane), plus the configured CNI/CSI plugins.

🔍 **Under the hood — registration:** a node self-registers via `POST /api/v1/nodes`, reporting:

- **Labels** — `kubernetes.io/hostname`, `topology.kubernetes.io/zone` & `region`, instance type, OS/arch.
- **Capacity** — `cpu`, `memory`, `pods`, `ephemeral-storage`.
- **Allocatable** = capacity − kubelet-reserved − system-reserved.

The cloud-controller-manager adds further cloud-specific labels and taints.

**How node health is signaled — three mechanisms:**

| Mechanism | Where | Meaning |
|---|---|---|
| **Lease** | `kube-node-lease` ns, renewed every 10s | Missing 40s → NodeNotReady |
| **NodeConditions** | node `status` | `MemoryPressure`, `DiskPressure`, `PIDPressure`, `NetworkUnavailable`, `Ready` |
| **Auto taints** | applied by node lifecycle controller | `node.kubernetes.io/not-ready:NoExecute`, `...unreachable:NoExecute` |

```bash
kubectl get nodes -o custom-columns=\
NAME:.metadata.name,\
STATUS:.status.conditions[-1].type,\
READY:.status.conditions[-1].status,\
CPU:.status.capacity.cpu,\
MEM:.status.capacity.memory

kubectl describe node <name>  # shows allocatable, conditions, taints, and running pods
```

---

## Request Lifecycle

> 🎯 **Interview weight: High** — knowing which phase produces which HTTP code is a classic rapid-fire round.

**In one line:** Every API request runs a gauntlet — TLS → authn → authz → APF → admission → validation → etcd write → watch — and each phase fails with its own status code.

**The phases and their failure codes:**

| Phase | What it does | Failure code |
|---|---|---|
| **TLS** | Terminate mutual/one-way TLS | — |
| **Authentication** | *Who are you?* First authenticator to succeed wins (client cert, token, SA JWT, OIDC, webhook) | **401** if none match |
| **Authorization** | *Are you allowed?* verb × group × resource × subresource × namespace; RBAC + Node authorizer | **403** if denied |
| **APF** | Fair-queue in-flight requests by FlowSchema so one client can't starve leader election | **429** if throttled |
| **Mutating admission** | MutatingAdmissionWebhooks (alphabetical order) | 400/403 if rejected |
| **Defaulting + validation** | Schema & field rules | **422** if invalid |
| **Validating admission** | ValidatingAdmissionWebhooks | 400/403 if rejected |
| **etcd write** | CAS on `resourceVersion`, then emit watch event | **409** on conflict |

> 🔍 **Under the hood:** on a mutating request the apiserver does a **compare-and-swap** on `resourceVersion` when writing to etcd; success mints a new resourceVersion and fans a **watch event** out to every informer *before* the HTTP response returns.

> ⚠️ **Gotcha:** **Node authorization** is a specialized authorizer that limits each kubelet to only *its own* node's pods, secrets, and configmaps — a key blast-radius control if a node is compromised.

```mermaid
flowchart TB
    A["📥 Client HTTPS request"]:::start --> B["🔐 TLS handshake"]:::proc
    B --> C{"🪪 Authentication"}:::proc
    C -->|"none match ❌"| E401["401 Unauthorized"]:::bad
    C -->|"identity ✅"| D{"🛡️ Authorization RBAC"}:::proc
    D -->|"denied ❌"| E403["403 Forbidden"]:::bad
    D -->|"allowed ✅"| F{"🚦 APF fairness"}:::proc
    F -->|"overloaded ❌"| E429["429 Too Many Requests"]:::bad
    F -->|"admitted ✅"| G["🧬 Mutating webhooks"]:::ctrl
    G --> H{"🧪 Defaulting + validation"}:::proc
    H -->|"invalid ❌"| E422["422 Invalid"]:::bad
    H -->|"valid ✅"| I["✔️ Validating webhooks"]:::ctrl
    I -->|"rejected ❌"| E400["400 / 403"]:::bad
    I -->|"accepted ✅"| J["💾 etcd write<br/>CAS on resourceVersion"]:::store
    J -->|"conflict ❌"| E409["409 Conflict"]:::bad
    J -->|"success ✅"| K["📡 Emit watch event to all informers<br/>+ 201 / 200 response"]:::good
    classDef start fill:#e3f2fd,stroke:#1565c0,color:#0d47a1,stroke-width:2px;
    classDef proc fill:#fff9c4,stroke:#f9a825,color:#000,stroke-width:2px;
    classDef good fill:#c8e6c9,stroke:#2e7d32,color:#1b5e20,stroke-width:2px;
    classDef bad fill:#ffcdd2,stroke:#c62828,color:#b71c1c,stroke-width:2px;
    classDef ctrl fill:#e1bee7,stroke:#6a1b9a,color:#4a148c,stroke-width:2px;
    classDef store fill:#ffe0b2,stroke:#e65100,color:#000,stroke-width:2px;
```

<sub>ASCII reference of the same lifecycle:</sub>

```
client HTTPS request
  │
  ├─ TLS (mutual or one-way)
  ├─ Authentication → 401 if none match
  ├─ Authorization → 403 if denied
  ├─ APF queue → 429 if overloaded
  ├─ Mutating Admission Webhooks → 400/403 if rejected
  ├─ Object defaulting + validation → 422 if invalid
  ├─ Validating Admission Webhooks → 400/403 if rejected
  ├─ etcd write (CAS on resourceVersion) → 409 if conflict
  └─ response (201 Created / 200 OK / error)
          │
          └─ watch event → all informers watching this resource
```

---

## Cluster Startup Sequence

> 🎯 **Interview weight: Medium** — dependency ordering explains most kubeadm bootstrap failures.

**In one line:** Components boot in strict dependency order — etcd first, everything else layered on top — which is why a failure early in the chain cascades.

🧠 **Mental model:** You need Kubernetes to start Kubernetes — the **static Pod** mechanism breaks that chicken-and-egg by letting the kubelet start the control plane from local manifest files, no apiserver required.

**The ordered boot chain:**

```mermaid
flowchart LR
    E["💾 etcd<br/>forms quorum"]:::store --> A["🚪 apiserver<br/>connects to etcd"]:::ctrl
    A --> C["🔄 controller-manager<br/>wins Lease, starts loops"]:::ctrl
    C --> S["📌 scheduler<br/>watches unscheduled pods"]:::ctrl
    S --> K["🤖 kubelet<br/>registers node"]:::proc
    K --> N["🕸️ CNI DaemonSet<br/>node becomes Ready"]:::proc
    N --> D["🔤 CoreDNS<br/>cluster DNS works"]:::good
    D --> P["🔀 kube-proxy<br/>Service routing live"]:::good
    classDef start fill:#e3f2fd,stroke:#1565c0,color:#0d47a1,stroke-width:2px;
    classDef proc fill:#fff9c4,stroke:#f9a825,color:#000,stroke-width:2px;
    classDef good fill:#c8e6c9,stroke:#2e7d32,color:#1b5e20,stroke-width:2px;
    classDef bad fill:#ffcdd2,stroke:#c62828,color:#b71c1c,stroke-width:2px;
    classDef ctrl fill:#e1bee7,stroke:#6a1b9a,color:#4a148c,stroke-width:2px;
    classDef store fill:#ffe0b2,stroke:#e65100,color:#000,stroke-width:2px;
```

1. **etcd starts first.** The etcd cluster forms quorum and begins accepting reads and writes. On a fresh cluster, the schema is empty. On restart, etcd reads its WAL and snapshot to restore state.

2. **kube-apiserver starts.** It connects to etcd. It serves health endpoints. Other components are not yet connected. On startup, the apiserver applies CRD schemas and bootstrap configurations.

3. **kube-controller-manager starts.** It connects to the apiserver, acquires the leader election Lease, and starts reconciliation loops. On a fresh cluster it sets up default ClusterRoles, Namespaces, and default ServiceAccount tokens.

4. **kube-scheduler starts.** It connects to the apiserver, acquires its Lease, and begins watching for unscheduled pods.

5. **kubelet starts on each node.** It registers the node, starts pulling pod specs for any static Pods (from `/etc/kubernetes/manifests/`), then watches for dynamically assigned pods.

6. **CNI plugin DaemonSet starts.** Kubelet cannot mark the node Ready until the network plugin reports readiness. CoreDNS pods cannot start until the node is Ready.

7. **CoreDNS starts.** After CoreDNS is ready, cluster DNS works and pods can resolve service names.

8. **kube-proxy or eBPF dataplane starts** (often as a DaemonSet). Service routing becomes active once kube-proxy has programmed iptables/IPVS rules.

During the bootstrapping phase (kubeadm init / managed cluster creation), control-plane components start as static Pods managed by the kubelet reading local manifest files — no apiserver is required to tell the kubelet to start them. This chicken-and-egg problem is resolved by the static Pod mechanism.

> 💡 **Interview tip:** When a kubeadm cluster "won't come up," walk the chain top-down: is **etcd** healthy first? Then apiserver, then the leader-elected controllers, then kubelet/CNI. A stuck node that never goes Ready is almost always the **CNI DaemonSet** (step 6) not reporting readiness.

---

## Component Interactions

> 🎯 **Interview weight: High** — "how do components coordinate?" → the API-server-as-hub answer.

**In one line:** *All* coordination happens through the API server as the single shared state store — no component ever calls another directly.

🧠 **Mental model — hub and spoke:** the apiserver is a switchboard. The scheduler doesn't RPC the kubelet; it writes a Binding via the apiserver. The Deployment controller doesn't call the scheduler; it creates Pod objects. The kubelet isn't *told* to report — it *watches* what it should run and *reports* status back.

**Why this is resilient:** any component can restart without breaking others' ability to read and act on objects. The one true single point of failure is the apiserver itself — and even that only stops *new* scheduling and mutations, not running workloads.

The sequence below traces a single `kubectl apply` through the whole hub-and-spoke cascade:

```mermaid
sequenceDiagram
  participant U as User/kubectl
  participant A as apiserver
  participant E as etcd
  participant D as Deployment controller
  participant R as ReplicaSet controller
  participant S as Scheduler
  participant K as kubelet
  participant RT as Runtime/CNI/CSI

  U->>A: POST /apis/apps/v1/deployments
  A->>E: write Deployment
  A-->>U: 201 Created
  D->>A: watch ADDED Deployment
  D->>A: POST ReplicaSet
  R->>A: watch ADDED ReplicaSet
  R->>A: POST Pod (no nodeName)
  S->>A: watch ADDED Pod (unscheduled)
  S->>A: POST Binding (nodeName=node-1)
  K->>A: watch MODIFIED Pod (nodeName=node-1)
  K->>RT: PullImage + RunSandbox + CNI + CSI + Start
  K->>A: PATCH pod/status (Running, Ready)
  A->>E: write updated Pod status
```

### Key commands
```bash
# Observe component leader election
kubectl -n kube-system get lease kube-controller-manager -o yaml
kubectl -n kube-system get lease kube-scheduler -o yaml

# See all component versions and health
kubectl version
kubectl get componentstatuses  # deprecated but still useful on some clusters

# Check control plane pod health on self-managed clusters
kubectl -n kube-system get pods -l tier=control-plane

# Observe the full lifecycle of a Deployment
kubectl apply -f deployment.yaml
kubectl get deploy,rs,pod -w   # watch the creation cascade in real time
kubectl rollout status deployment/payments

# Node registration and lease
kubectl get nodes
kubectl -n kube-node-lease get leases | head -10
kubectl describe node <node> | grep -A30 "Conditions:"
```

---

## Interview Questions

### Conceptual / Internals (8 questions)

**1. Why does Kubernetes use level-triggered reconciliation instead of event-triggered mutation, and what are the failure resilience implications?**

Level-triggered means the controller observes current state and desired state and converges from wherever things are, not by replaying a specific event. If the controller crashes after seeing "pod A was deleted" but before acting on it, an event-triggered system never re-runs that deletion reaction. A level-triggered system restarts, reads all objects, finds that Pod A should not exist but a ReplicaSet says it should have 3 replicas, and creates a replacement. The reconciliation loop is self-healing by nature: missed events become visible as drift between actual and desired on the next loop iteration. The cost is that the controller must re-read the full object state on every reconcile, which is why informers and listers (local cache) are critical for performance — without them, every reconcile would hit the apiserver.

**2. What is the role of `generation` and `observedGeneration` in a Kubernetes resource, and how should they be used in automation?**

`metadata.generation` is incremented by the apiserver every time the `spec` of an object changes (not metadata, not status). It is monotonically increasing. `status.observedGeneration` is set by the controller to the generation it last reconciled. If `observedGeneration < generation`, the controller has not yet processed the latest spec. Automation that polls for rollout completion should check that `status.observedGeneration == metadata.generation` AND `status.conditions[?(@.type=="Available")].status == "True"` AND `status.readyReplicas == spec.replicas`, rather than just checking `readyReplicas`. A Deployment rollout script that only checks `readyReplicas` may declare success when an old generation's pods are ready but the new template hasn't been applied yet.

**3. Explain how static Pods solve the chicken-and-egg problem of Kubernetes bootstrapping.**

The kubelet can start pods from manifest files in a local directory (default `/etc/kubernetes/manifests/`) without the apiserver being available. These are static Pods: the kubelet reads the manifest, creates the pod using the runtime directly, and creates a mirror pod object in the apiserver (once the apiserver is available) to make it visible. On a new cluster, kubeadm writes control-plane manifests to this directory before starting anything. The kubelet starts etcd, then apiserver, then controller-manager and scheduler — all as static Pods — without any Kubernetes control plane involvement. This solves the bootstrapping problem: you need Kubernetes to start Kubernetes, but you can start the pieces by hand via manifest files.

**4. How does a Kubernetes controller handle a `409 Conflict` from the apiserver, and what is the underlying mechanism that produces it?**

A `409 Conflict` on an update or patch means the object's `resourceVersion` in the request does not match the current `resourceVersion` in etcd. The apiserver performs a compare-and-swap (CAS) when writing: it reads the current resource version, checks it matches the version in the request, and only then writes. If another writer modified the object between the controller's read and write, the CAS fails with 409. The controller should handle 409 by re-reading the object from the lister (getting the current version), recomputing the desired state, and retrying the write. client-go's retry utilities handle this automatically for most operations. This optimistic concurrency is how Kubernetes prevents lost updates without distributed locking.

**5. What happens to running pods when the control plane is completely unavailable for 30 minutes?**

Already-running containers continue executing uninterrupted — the control plane does not participate in container I/O, memory access, or CPU scheduling. iptables/IPVS rules for Services remain in place because kube-proxy installed them in the kernel on its last run. CNI routes remain active. CSI-mounted volumes remain accessible because the mount was done at pod start time and is maintained by the kernel, not the control plane. What stops: (1) new pod scheduling (scheduler cannot write bindings); (2) pod replacements for crashed pods (kubelet cannot fetch pod specs for new pods, though it can restart containers per the restart policy using the cached spec if the pod already existed on the node); (3) secret/configmap volume updates (projected volumes served from kubelet cache, which may be stale); (4) HPA/CA scaling decisions. The critical concern is certificate expiration: short-lived service account tokens will expire and pods that need to call the apiserver will fail.

**6. How does the Kubernetes watch mechanism work, and what causes a watch to receive a 410 Gone response?**

A client (e.g., a controller informer) first does a LIST request with `resourceVersion=""` (or a specific version) to get the current state of objects. The apiserver responds with a list that includes the current `resourceVersion`. The client then opens a WATCH request starting at that resourceVersion: `GET /api/v1/pods?watch=1&resourceVersion=1234`. The apiserver maintains a watch cache (an in-memory ring buffer of recent events per resource type). As long as the client's requested resource version falls within the cache window, events are streamed. If the client's resource version has been compacted out of the cache (the watch cache ring buffer has overflowed, or the client reconnected after a long pause), the apiserver returns `410 Gone`. The client's informer then performs a full relist (LIST with `resourceVersion=""`) to get fresh state, then restarts the watch. This is why thundering herd after a mass reconnect (e.g., apiserver restart) creates heavy LIST load on etcd.

**7. Explain the difference between foreground and background cascading deletion in Kubernetes.**

When you delete an owner object (e.g., a Deployment), the garbage collector must also delete owned objects (ReplicaSets → Pods). In **background deletion** (default), the owner is immediately deleted from etcd. The garbage collector runs asynchronously and deletes orphaned children. The owner disappears from `kubectl get` immediately, but pods may persist briefly. In **foreground deletion**, the owner gets a `DeletionTimestamp` but is NOT removed yet; it also gets the `foregroundDeletion` finalizer. The garbage collector first deletes all children. When all children are gone, it removes the `foregroundDeletion` finalizer, and the owner is finally deleted. Foreground ensures all children are cleaned up before the parent disappears — important for resources with external cleanup hooks (CSI volumes, load balancers). The `--cascade=foreground` flag on `kubectl delete` controls this.

**8. Why does Kubernetes have separate `kube-controller-manager` and `cloud-controller-manager` binaries, and what would happen if cloud-specific code were in the core controller-manager?**

The original `kube-controller-manager` included cloud-provider logic (for AWS, GCP, Azure, etc.) compiled directly into the binary. This meant Kubernetes releases were coupled to cloud-provider API updates, and adding a new cloud provider required changes to the core Kubernetes binary. The cloud-controller-manager (CCM) extracts cloud-specific controllers (Node lifecycle, Route management, Service LoadBalancer provisioning) into a separate binary that cloud providers can release independently of core Kubernetes. This is the "out-of-tree cloud provider" model. It also means the apiserver and core controllers are not dependent on any cloud provider SDK, reducing the attack surface and simplifying auditing of the core. If cloud code were in the core, a vulnerability in an AWS SDK would affect all clusters regardless of which cloud they run on.

---

### Scenario / Troubleshooting (6 questions)

**9. You apply a Deployment with `replicas: 10` but only 3 pods are running after 10 minutes. How do you diagnose?**

`kubectl describe deployment <name>` — check `Conditions` and `Events`. Then `kubectl get rs` — find the ReplicaSet owned by the Deployment and check its `Events`. Then `kubectl get pods -l app=<label>` and look at status columns for pending/failing pods. For pending pods, `kubectl describe pod <pending-pod>` — read the `Events` section. Common causes: resource limits exceeded on all nodes (events say "Insufficient cpu/memory"), taints blocking pods (events say "didn't match node selector taint"), quota exceeded on the namespace (events say "exceeded quota"), PVC not binding (events say "persistentvolumeclaim not found"), or image pull failures (events say "ImagePullBackOff"). Each event gives a distinct diagnosis path.

**10. After a control-plane node fails, you notice etcd is refusing writes with "leader not found." What has happened and what are your recovery steps?**

A 3-node etcd cluster requires 2 nodes for a write quorum. With one node down, 2 remain — still a quorum — so this symptom usually means 2 nodes failed, or the remaining nodes cannot reach each other. Check `etcdctl endpoint health --cluster` from a surviving node. If the cluster has lost quorum, etcd enters read-only mode. Recovery options: (1) if the failed node is recoverable, restore it and let it rejoin by restarting etcd with the same cluster config — it will pull the delta from the leader; (2) if data is lost, restore from an etcd snapshot: `etcdctl snapshot restore`, then start etcd with `--force-new-cluster` on one node to bootstrap a single-node cluster, restore the snapshot, then add back members; (3) in managed Kubernetes (EKS/AKS/GKE), the control plane including etcd is managed by the provider — contact support and show them the timeline.

**11. A Deployment rollout is stuck at 50% because new pods are not becoming Ready. How do you investigate?**

A rolling update with `maxUnavailable: 1` and `maxSurge: 1` will not proceed past a certain point if new pods are not Ready — because the controller won't scale down old pods until new ones are Ready. `kubectl rollout status deployment/<name>` will show it waiting. `kubectl describe pod <new-pod>` — check readiness probe failures in Events (e.g., "Readiness probe failed: HTTP probe failed with statuscode 500"). Use `kubectl logs <new-pod>` and `kubectl exec <new-pod> -- curl localhost:8080/healthz` to see whether the application itself is unhealthy. The issue could be a misconfigured probe, a startup latency not covered by `initialDelaySeconds`, a broken environment variable for a new config key, or a dependency (database, cache) that isn't reachable from the new pod version. Fix the root cause, not the probe timeout.

**12. A node is in `NotReady` state. What are the first five commands you run and why?**

1. `kubectl describe node <node>` — shows Conditions (what pressure is active), Events (what happened and when), and the last resource version. Condition `Ready: False` with message "node was unable to contact API server" indicates a network or kubelet issue.
2. `kubectl -n kube-node-lease get lease <node> -o yaml` — check `renewTime`. If it's old, the kubelet has not renewed its heartbeat. If it's current, the node is still alive but conditions are wrong.
3. On the node: `systemctl status kubelet` — is kubelet running? What error?
4. `journalctl -u kubelet --since "10m ago" | tail -100` — read kubelet logs for the failure cause: network plugin not ready, certificate error, runtime unavailable.
5. `systemctl status containerd` + `crictl info` — is the container runtime healthy? A runtime crash causes kubelet to fail pod sync and eventually mark itself NotReady.

**13. After deploying a new version of a StatefulSet, pods are not rolling: all old pods remain. Why?**

StatefulSets default to `updateStrategy: RollingUpdate`, which rolls pods from highest ordinal to lowest. But the roll only proceeds if each pod passes its readiness check before the next pod is updated. If the highest-ordinal pod (e.g., `pod-2`) fails readiness after its container is updated, the rollout stalls. Check `kubectl describe pod <statefulset-2>` for probe failures. Also check that `updateStrategy.rollingUpdate.partition` is not set to a value higher than 0 — a partition tells the StatefulSet controller to only update pods with an ordinal >= the partition value, keeping lower ordinals on the old version intentionally (used for manual canary on StatefulSets). Run `kubectl get statefulset <name> -o yaml | grep partition`.

**14. A cluster-wide admission webhook is causing all pod creates to fail with "connection refused." How do you recover?**

If `failurePolicy: Fail`, the webhook being unavailable causes all matching pod creates to fail — including Deployments, DaemonSets, and jobs. Immediate recovery: patch the webhook to `failurePolicy: Ignore` to make it non-blocking: `kubectl patch mutatingwebhookconfiguration <name> --type=json -p='[{"op":"replace","path":"/webhooks/0/failurePolicy","value":"Ignore"}]'`. If even that requires a pod to start (which fails), use `kubectl delete mutatingwebhookconfiguration <name>` to remove it entirely — this requires only a control-plane API call, no pod creation. Then fix the webhook service/deployment and re-add the configuration. Prevention: narrow `namespaceSelector` to exclude system namespaces, use `failurePolicy: Ignore` for non-critical webhooks, maintain multiple replicas with a PDB for critical webhooks, and test webhook unavailability as part of runbooks.

---

### FAANG-Level Deep Dive (6 questions)

**15. How does the kube-apiserver implement multi-version APIs, and how does etcd store objects that are served in multiple versions simultaneously?**

The apiserver maintains an "internal" (hub) type for each API group plus all versioned types. When a v1beta1 object is submitted, the apiserver converts it to the internal type for processing, then converts it to the configured storage version (often v1) before writing to etcd. When a client requests a v1beta1 object that was stored as v1, the apiserver reads v1 from etcd and converts to v1beta1 before responding. Conversion functions (generated by code-gen or written manually) handle each version pair. For CRDs, conversion webhooks handle multi-version conversion without changing the main apiserver binary. etcd stores only one version of each object (the storage version). The `status.storedVersions` field on a CRD tracks all versions ever stored, which must remain served until migrated.

**16. Explain the Raft leader election algorithm as used in etcd, including what happens during a network partition where neither partition has a majority.**

In Raft, a node starts as a Follower and transitions to Candidate if it doesn't receive a heartbeat from a leader within the election timeout (150-300ms random). As a Candidate, it increments its term and sends RequestVote RPCs to all peers. A node grants a vote to a candidate if: it hasn't voted in the current term, and the candidate's log is at least as up-to-date as the voter's. If the candidate receives votes from a majority (⌊n/2⌋ + 1), it becomes Leader and begins sending AppendEntries heartbeats. In a 3-node cluster partitioned 1|2: the partition with 2 nodes has a majority, elects a leader, and continues accepting writes. The 1-node partition cannot elect a leader (it needs 2 votes, can only get 1), so it remains in Candidate/Follower state in an election loop. When the partition heals, the 1-node partition sees the leader's higher term via heartbeats, reverts to Follower, and catches up via log replication. Importantly: during the partition, the 1-node side makes no progress (no writes accepted), but the 2-node side can continue accepting writes. etcd does not split-brain because it always requires a quorum write.

**17. Walk through the source code path in client-go from a watch event arriving over the HTTP response body to a controller's Reconcile function being called.**

`Reflector.ListAndWatch()` opens the HTTP watch stream. Responses are decoded by `WatchDecoder.Decode()` using the registered codec for the resource. Decoded events are pushed into `DeltaFIFO` (a FIFO queue with deduplication, implemented in `cache/delta_fifo.go`). The `Informer`'s `processLoop` goroutine calls `DeltaFIFO.Pop()` in a loop. Each popped delta is processed: the store (a thread-safe `cache.Store`) is updated (Add/Update/Delete), and registered EventHandlers (`ResourceEventHandlerFuncs`) are called. The `AddFunc`/`UpdateFunc`/`DeleteFunc` handlers (registered by the controller) typically call `queue.Add(key)` where `key = namespace/name`. The work queue (`workqueue.RateLimitingInterface`) receives the key. Worker goroutines (`controller.worker()`) call `queue.Get()`, look up the object via the lister (which reads the cache), and call `Reconcile(ctx, request)`. After reconcile returns, `queue.Done(key)` is called; if reconcile failed, `queue.AddRateLimited(key)` requeues with backoff.

**18. How would you design a Kubernetes controller that manages 100,000 custom resources (CRs) at extremely low latency without overwhelming the apiserver?**

Key design decisions: (1) Use `SharedIndexInformer` with label/field selector filtering to limit the watch scope — avoid a cluster-wide informer if the CRs are namespaced. (2) Use `MetadataInformer` (metadata-only watch) if the controller only needs to react to creation/deletion without reading the full spec until reconcile time. (3) Use multiple work-queue worker goroutines (50-200) to parallelize reconciliation; use `workqueue.NewRateLimitingQueue` with a `ItemExponentialFailureRateLimiter` to avoid thundering herds on errors. (4) Batch status updates with a dedicated status updater goroutine and aggregation window to avoid per-reconcile status PATCH calls. (5) Set a leader election lock with a short renew interval so failover is fast. (6) Use server-side apply with field manager for idempotent updates — no need for a read-modify-write cycle in the controller. (7) Implement a resync period greater than the expected reconcile time to avoid overwhelming the queue with periodic re-syncs.

**19. How does the Kubernetes scheduler achieve parallelism during filtering and scoring while preventing race conditions on node state?**

The scheduler takes a snapshot of node state at the start of each scheduling cycle (via `snapshot()` on the node cache). This snapshot is immutable for the duration of the cycle. The snapshot includes node capacity, existing pod requests, and conditions. Filter plugins run in parallel (one goroutine per node) against this snapshot — since the snapshot is read-only, no locking is needed during filtering. Score plugins also run in parallel per node against the same snapshot. After scoring, the scheduler selects the winner, then executes the `Reserve` phase (which tentatively "reserves" the resources in the live cache, not the snapshot, under a mutex), then issues the binding. The Reserve step is what prevents two pods from being simultaneously scheduled to the same node in concurrent scheduling cycles — the live cache is updated under lock before the bind is confirmed, so the next cycle's snapshot includes the reserved resources.

**20. Explain the OwnerReference garbage collection mechanism in detail, including the difference between orphan deletion, foreground deletion, and background deletion, and how the garbage collector resolves circular references.**

`OwnerReferences` form a DAG (not necessarily a tree; multiple owners are allowed). The garbage collector (GC) runs in the controller-manager. It maintains a directed graph of all objects and their owner references. When an object is deleted and has `OwnerReferences`, the GC processes them. In **orphan mode** (`--cascade=orphan`), the owner is deleted but children lose their owner reference and continue existing as orphans. In **background mode** (default), the owner is immediately deleted, and the GC asynchronously identifies and deletes children by scanning all objects for matching owner UID + namespace. In **foreground mode**, the owner gets a `DeletionTimestamp` and the `foregroundDeletion` finalizer; the GC deletes all children (in dependency order, respecting their own finalizers) and removes the finalizer only when the owner's `blockOwnerDeletion`-referencing children are all gone. Circular references are pathological and not supported — the GC assumes a DAG. If you create circular owner references, deletion will never complete (each object is blocked by the other). Kubernetes admission prevents obvious cycles for built-in resources but cannot prevent them for CRDs.

---

## Hands-On Labs

### Lab 1: Observe the Deployment Reconciliation Cascade

**Objective:** Watch the entire object lifecycle from `kubectl apply` to Ready pods.

**Setup:** Any Kubernetes cluster with `kubectl`.

**Tasks:**
1. Open four terminals. In terminal 1: `kubectl get deploy,rs,pod -w`. In terminal 2: `kubectl get events -w --sort-by=.lastTimestamp`. In terminal 3: `kubectl -n kube-system logs -l component=kube-controller-manager --tail=0 -f 2>/dev/null`.
2. Apply a Deployment in terminal 4: `kubectl apply -f https://k8s.io/examples/application/deployment.yaml`.
3. Observe in real time: the Deployment is created, the Deployment controller creates a ReplicaSet, the ReplicaSet controller creates Pods, the scheduler binds Pods to nodes (visible in events), kubelet starts containers and updates pod status.
4. Pause and resume: `kubectl scale deployment nginx-deployment --replicas=0`, wait for pods to terminate, then `kubectl scale deployment nginx-deployment --replicas=5`. Observe the reconciliation cascade again.

**Expected outcome:** You observe that the cascade is asynchronous and each object creation triggers the next controller independently.

### Lab 2: Simulate Control Plane Failure

**Objective:** Prove that data-plane stability persists during control-plane unavailability.

**Setup:** A local kind cluster.

**Tasks:**
1. Deploy a web application and confirm it serves traffic: `kubectl port-forward svc/my-app 8080:80 &`.
2. Confirm the pod serves: `curl localhost:8080`.
3. Pause the kube-controller-manager: `docker pause <kind-control-plane-container>` then manually pause just the controller-manager: on the control-plane node, find the controller-manager pid and send SIGSTOP.
4. Confirm the pod still serves: `curl localhost:8080` should still succeed.
5. Attempt `kubectl scale deployment my-app --replicas=3` — it will hang (scheduler/controller are unavailable but apiserver is up) or fail.
6. Kill the application pod manually: `kubectl delete pod <pod>`. Observe that no replacement starts while the controller-manager is stopped.
7. Resume the controller-manager. Observe the replacement pod being created and starting.

**Expected outcome:** Confirms static stability: data plane continues serving while control plane is partially down.

### Lab 3: Explore Leader Election

**Objective:** Understand how Kubernetes components use Lease-based leader election.

**Setup:** A multi-replica control plane or a local cluster.

**Tasks:**
1. Inspect the scheduler lease: `kubectl -n kube-system get lease kube-scheduler -o yaml`. Note `holderIdentity`, `leaseDurationSeconds`, `acquireTime`, `renewTime`.
2. Write a script that monitors renewTime: `while true; do kubectl -n kube-system get lease kube-scheduler -o jsonpath='{.spec.renewTime}'; echo; sleep 2; done`.
3. If using a kind cluster with multiple nodes, stop the scheduler pod and observe: renewTime stops updating, then another scheduler pod (if available) acquires the lease.
4. Write a simple leader-election program using client-go's `leaderelection` package that acquires the lease and logs "I am the leader" every second.

**Expected outcome:** You understand that leader election is simply optimistic locking on a Kubernetes Lease object, not a network consensus protocol.

---

## Production Incidents

### Incident 1: Mass Eviction Storm After etcd Compaction

**Symptom:** 200 pods evicted simultaneously across the cluster during a low-traffic period. PagerDuty alert: "Too many pods not running." Recovery took 15 minutes as pods were rescheduled and restarted.

**Investigation:** `kubectl get events -A --sort-by=.lastTimestamp` shows mass eviction at 03:42 UTC. Control-plane logs from etcd show a compaction job completed at 03:41. Apiserver logs show a burst of 410 Gone responses from etcd watch cache around 03:42. Controller-manager logs show thousands of reconcile loop invocations starting at 03:42. All controllers triggered their reconciliation at once, flooding the API server with LIST calls, creating an API server overload (HTTP 429 responses). New pod creates were delayed, and existing pods that needed replacement were slow to restart. The scheduler queue backed up.

**Root cause:** etcd compaction (removing historical revisions below the threshold) caused the apiserver watch cache to serve 410 Gone responses to all open watches. Every informer in the controller-manager relisted from scratch simultaneously. The thundering herd of LIST requests overloaded the apiserver and etcd.

**Recovery:** The storm subsided after 8 minutes as controllers spread their retries. Manual scaling of the apiserver to 5 replicas absorbed the burst. Pods rescheduled within 15 minutes.

**Prevention:** Run etcd compaction during low-traffic windows. Stagger compaction intervals across etcd nodes. Consider increasing the apiserver watch cache size to absorb more revisions before 410 responses. Add `--etcd-compaction-interval` tuning. Implement circuit breakers in custom controllers to prevent API storms.

### Incident 2: Control Plane Certificate Expiration

**Symptom:** On a Monday morning, all API calls begin failing with "x509: certificate has expired or is not yet valid." Cluster is unreachable. `kubectl get nodes` returns "Unable to connect."

**Investigation:** The kube-apiserver serving certificate is valid for one year, and this cluster was set up 365 days ago by a former engineer. The certificate was generated by kubeadm but certificate rotation was never configured. The control-plane is running but TLS handshakes fail for all clients.

**Root cause:** Kubernetes API server serving certificate expired. kubeadm auto-rotates certificates only on upgrade; this cluster had not been upgraded in 12 months.

**Recovery:** On the control-plane node, run `kubeadm certs renew all`. Restart all control-plane components (as static Pods, this requires restarting kubelet). Distribute the new kubeconfig to users: `scp /etc/kubernetes/admin.conf <users>`.

**Prevention:** Set a calendar reminder and a Prometheus alert: `apiserver_certificate_expiration_seconds < (86400 * 30)`. Run `kubeadm certs check-expiration` as a CronJob from inside the cluster. Adopt managed Kubernetes (EKS, AKS, GKE) which manages certificate rotation automatically. If self-managed, upgrade clusters at least annually, which triggers kubeadm cert renewal.
