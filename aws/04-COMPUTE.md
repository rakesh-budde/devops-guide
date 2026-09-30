# AWS COMPUTE — Deep Dive Interview Preparation

> **Scope:** Section 4 of 20 | Beginner → Expert progression | FAANG-level depth  
> **Coverage:** EC2 internals, Nitro, provisioning, Placement Groups, ASG, Spot, Reserved/Savings, Lambda, Fargate & App Runner

---

## Table of Contents

**Section 4: Compute**
1. [EC2 Instance Types & Families](#1-ec2-instance-types--families)
2. [Nitro System Architecture](#2-nitro-system-architecture)
3. [EC2 Provisioning & Boot Process](#3-ec2-provisioning--boot-process)
4. [Placement Groups](#4-placement-groups)
5. [Auto Scaling Groups & Launch Templates](#5-auto-scaling-groups--launch-templates)
6. [Spot Instances](#6-spot-instances)
7. [Reserved Instances & Savings Plans](#7-reserved-instances--savings-plans)
8. [AWS Lambda Deep Dive](#8-aws-lambda-deep-dive)
9. [AWS Fargate & App Runner](#9-aws-fargate--app-runner)

**Common**
15. [Interview Questions & Answers](#15-interview-questions--answers)
16. [Troubleshooting Scenarios](#16-troubleshooting-scenarios)
17. [Production Best Practices](#17-production-best-practices)
18. [Documentation Links](#18-documentation-links)

---

## 🗺️ Visual Overview

**In one line:** Compute is about how you rent CPU — instances, scaling, pricing, and serverless — and almost every interview question is really "pick the right service for this workload and defend the cost."

**Mind map — the compute half at a glance** (skim this first, revisit it last):

```mermaid
mindmap
  root((AWS Compute))
    EC2 Compute
      Families m c r i g t
      Generation and size
      Graviton ARM64
      AMIs and boot
      Nitro system
      Placement groups
    Scaling and Pricing
      Auto Scaling Groups
      Launch Templates
      Spot up to 90 off
      Reserved Instances
      Savings Plans
    Serverless
      Lambda per ms billing
      Cold start Firecracker
      Fargate containers
      App Runner
```

**Decision tree — the Auto Scaling lifecycle** (blue = launch, purple = hook control points, green = serving):

```mermaid
flowchart LR
    A["📈 Scale-out alarm<br/>CPU &gt; 70%"] --> B["🚀 RunInstances<br/>from Launch Template"]
    B --> C["⏳ Pending<br/>OS boot + app start"]
    C --> D["🛑 Lifecycle hook<br/>Pending:Wait<br/>init, warm cache"]
    D --> E["🩺 Grace period<br/>then ELB health check<br/>GET /health = 2xx"]
    E -->|"Healthy ✅"| F["✅ InService<br/>joins target group,<br/>serves traffic"]
    E -->|"Fails ❌"| G["❌ Terminated<br/>replaced automatically"]
    F --> H["📉 Scale-in"] --> I["🛑 Terminating:Wait<br/>drain conns,<br/>deregister from LB"]
    I --> J["🗑️ Terminated<br/>gracefully"]

    class A start
    class B,C,E proc
    class D,I ctrl
    class F good
    class G bad
    class J store
    classDef start fill:#e3f2fd,stroke:#1565c0,color:#0d47a1,stroke-width:2px;
    classDef proc fill:#fff9c4,stroke:#f9a825,color:#000,stroke-width:2px;
    classDef good fill:#c8e6c9,stroke:#2e7d32,color:#1b5e20,stroke-width:2px;
    classDef bad fill:#ffcdd2,stroke:#c62828,color:#b71c1c,stroke-width:2px;
    classDef ctrl fill:#e1bee7,stroke:#6a1b9a,color:#4a148c,stroke-width:2px;
    classDef store fill:#ffe0b2,stroke:#e65100,color:#000,stroke-width:2px;
```

> 🧠 **Memory hooks (mnemonics):**
> - **EC2 family letters:** *"My Cat Really Isn't Getting Tired"* → **M**=general purpose (balanced), **C**=**C**ompute-optimized, **R**=**R**AM/memory-optimized, **I**=**I**OPS/instance-storage, **G**=**G**PU/graphics, **T**=**T**iny/burstable. (Also: **X/z**=extreme memory, **D/H**=dense disk, **P/Inf/Trn**=ML accelerators.)
> - **Instance name decode:** `m7g.4xlarge` = **family**(m) + **generation**(7) + **attribute**(g=Graviton) + **size**(4xlarge). Read it left-to-right like a license plate.
> - **Spot vs Reserved:** **Spot** = cheap but can be **s**natched back (2-min notice); **Reserved/Savings** = you **r**eserve/commit for a discount.

---

## 1. EC2 Instance Types & Families

### Beginner Foundation

An **EC2 instance** is a virtual machine running in the AWS cloud. The instance type determines vCPU count, memory, storage, and network performance.

> **In one line:** The instance type name is a compact spec sheet — read `family + generation + attribute + size` left-to-right and you instantly know the workload it's tuned for and how big it is.

**Instance naming: `family + generation + [attribute] + size`**

Example: `m7g.4xlarge`
- `m` = General purpose family
- `7` = 7th generation
- `g` = Graviton (AWS ARM processor)
- `4xlarge` = 16 vCPU, 64 GiB RAM

### Intermediate Mechanics — Instance Families

| Family | Prefix | vCPU:RAM | Best for |
|---|---|---|---|
| General Purpose | m, t | 1:4 | Web servers, app servers, dev/test |
| Compute Optimized | c | 1:2 | Batch, HPC, gaming servers |
| Memory Optimized | r, x, z | 1:8 to 1:32 | In-memory databases, SAP HANA |
| Storage Optimized | i, d, h | High local NVMe | NoSQL, data warehouses, HDFS |
| Accelerated Computing | p, g, inf, trn | GPU | ML training, inference, video encoding |
| T-series (burstable) | t | Variable | Variable CPU workloads |

**T-series burstable deep dive:**

T-series instances earn CPU credits when running below baseline (e.g., `t3.medium` baseline = 20% of 2 vCPUs). When above baseline, credits are consumed. `t3` and newer are **unlimited** by default — burst indefinitely but surplus credits are charged. This surprises teams whose dev `t3.small` runs CPU-intensive jobs for days.

> ⚠️ **Gotcha:** A `t3`/`t3a` instance is **Unlimited by default**. A runaway CPU job doesn't throttle — it silently accrues surplus-credit charges. For steady CPU-heavy work, switch to `m`/`c` (fixed performance) or set the credit mode to `standard`.

```bash
# Check CPU credit balance
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name CPUCreditBalance \
  --dimensions Name=InstanceId,Value=i-0abc123 \
  --start-time $(date -d '1 hour ago' -u +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 300 --statistics Average
```

**Graviton (ARM64) instances:**

`m7g`, `c7g`, `r7g` — AWS-designed ARM chips. 20–40% better price/performance than equivalent x86. Requires ARM64-compiled code. Most modern runtimes (Java, Go, Python, Node.js, .NET) support ARM64 natively. Docker images must be multi-arch.

```bash
# Build multi-arch container image
docker buildx build --platform linux/amd64,linux/arm64 -t my-app:latest --push .
```

### Advanced Engineering

**Network bandwidth is per-instance-type:** `m5.large` = 1.25 Gbps; `m5.24xlarge` = 25 Gbps. EBS bandwidth is separate from network bandwidth — heavy EBS I/O can saturate the EBS throughput cap without affecting network. Monitor both `NetworkIn/Out` and `EBSWriteBytes` metrics separately.

> 💡 **Interview tip:** "Why did network throughput look fine while my disk-heavy job crawled?" — Because **EBS bandwidth and network bandwidth are separate caps** on Nitro. Always cite both `NetworkIn/Out` *and* `EBSRead/WriteBytes` when diagnosing throughput ceilings.

---

## 2. Nitro System Architecture

### Beginner Foundation

The **Nitro System** is AWS's custom hardware and software that offloads virtualization functions (networking, storage, security) to dedicated Nitro hardware cards, giving customer instances near bare-metal performance with < 1% overhead.

> **In one line:** Nitro moves the "hypervisor tax" (networking, storage, security) off the main CPU onto dedicated cards, so your instance gets ~100% of the cores and near bare-metal speed.

### Intermediate Mechanics

```mermaid
graph TB
    subgraph NitroHost["🖥️ Physical Nitro Host"]
        subgraph Guest["Customer EC2 Instance"]
            OS["🧑‍💻 Guest OS + Workload<br/>100% of vCPUs"]
        end
        NH["⚙️ Nitro Hypervisor<br/>thin KVM, CPU/memory only"]
        subgraph Cards["Dedicated Nitro Cards"]
            VPC["🌐 Nitro VPC Card ENA<br/>hardware packet processing"]
            EBS["💽 Nitro EBS Card NVMe<br/>storage I/O without CPU"]
            Sec["🔒 Nitro Security Chip<br/>hardware root of trust,<br/>blocks operator access"]
        end
    end
    OS --> NH
    NH --> Cards
    VPC --> Network["☁️ AWS VPC Network"]
    EBS --> Storage["🟠 EBS Volumes"]

    class OS good
    class NH ctrl
    class VPC,EBS,Sec proc
    class Network start
    class Storage store
    classDef start fill:#e3f2fd,stroke:#1565c0,color:#0d47a1,stroke-width:2px;
    classDef proc fill:#fff9c4,stroke:#f9a825,color:#000,stroke-width:2px;
    classDef good fill:#c8e6c9,stroke:#2e7d32,color:#1b5e20,stroke-width:2px;
    classDef bad fill:#ffcdd2,stroke:#c62828,color:#b71c1c,stroke-width:2px;
    classDef ctrl fill:#e1bee7,stroke:#6a1b9a,color:#4a148c,stroke-width:2px;
    classDef store fill:#ffe0b2,stroke:#e65100,color:#000,stroke-width:2px;
```

**Nitro Security Chip:** Enforces at hardware level that AWS operators cannot read customer instance memory, storage, or network traffic. This is a cryptographic hardware guarantee, not just a policy.

**Nitro Enclaves:** Isolated VMs within EC2 for processing sensitive data. No persistent storage, no network, no interactive access. AWS KMS releases keys only to verified Enclave attestation reports. Used for PII processing, HSM operations, ML inference on private data.

**Bare metal instances** (`m5.metal`, `c5.metal`): No hypervisor between OS and hardware. Used for VMware Cloud on AWS (nested virtualization), workloads requiring direct hardware access.

### Advanced Engineering

**EBS lazy loading from AMI snapshots:** When a volume is created from a snapshot, blocks are fetched from S3 on first access (150–200 ms vs < 1 ms for cached blocks). Pre-warm production volumes before traffic:

```bash
# Pre-warm all blocks on a newly created EBS volume
sudo fio --filename=/dev/xvda --rw=randread --bs=128k --iodepth=32 \
  --ioengine=libaio --direct=1 --name=pre-warm --runtime=600
```

AWS Fast Snapshot Restore (FSR) pre-initializes blocks — eliminates warm-up latency at extra cost (~$0.75/AZ/hour enabled).

---

## 3. EC2 Provisioning & Boot Process

### Intermediate Mechanics

> **In one line:** `RunInstances` is a control-plane pipeline — find capacity, place on a Nitro host, wire up networking + a lazily-loaded root volume, boot, then run your user-data and health checks.

**`RunInstances` control plane flow:**

```mermaid
flowchart TD
    A["📨 RunInstances<br/>API call"] --> B{"Capacity in<br/>requested AZ?"}
    B -->|"No ❌"| Z["🚫 InsufficientInstanceCapacity<br/>try another AZ / type"]
    B -->|"Yes ✅"| C["🎯 Scheduler picks<br/>Nitro host"]
    C --> D["🌐 ENI created<br/>private IP + security groups"]
    D --> E["🟠 EBS root volume<br/>from AMI snapshot<br/>lazy loading"]
    E --> F["⚙️ Nitro boots VM<br/>UEFI/BIOS runs"]
    F --> G["🔑 IMDS available<br/>169.254.169.254"]
    G --> H["📜 user-data runs<br/>cloud-init / EC2Launch"]
    H --> I["🩺 Status checks<br/>system + instance"]
    I --> J["✅ Instance ready"]

    class A start
    class B proc
    class Z bad
    class C,D,F,G,H,I proc
    class E store
    class J good
    classDef start fill:#e3f2fd,stroke:#1565c0,color:#0d47a1,stroke-width:2px;
    classDef proc fill:#fff9c4,stroke:#f9a825,color:#000,stroke-width:2px;
    classDef good fill:#c8e6c9,stroke:#2e7d32,color:#1b5e20,stroke-width:2px;
    classDef bad fill:#ffcdd2,stroke:#c62828,color:#b71c1c,stroke-width:2px;
    classDef store fill:#ffe0b2,stroke:#e65100,color:#000,stroke-width:2px;
```

1. **Capacity check:** Is the requested instance type available in the specified AZ? No → `InsufficientInstanceCapacity`.
2. **Scheduler:** Selects a Nitro host with capacity.
3. **ENI creation:** Private IP assigned from subnet CIDR; security groups attached.
4. **EBS root volume:** Created from AMI snapshot (lazy loading).
5. **Boot:** Nitro hypervisor starts the VM; UEFI/BIOS runs.
6. **IMDS available** at `169.254.169.254`.
7. **User data:** `cloud-init` / Windows EC2Launch executes user-data.
8. **Status checks:** System (host health) and instance (OS reachability) begin.

**Status checks:**

| Check | Monitors | If failing | Action |
|---|---|---|---|
| System status | Underlying Nitro host hardware | AWS host issue | Auto Recovery (moves to new host) |
| Instance status | OS reachability, IMDS health | OS crash, OOM, disk full | Your intervention (stop/start, SSM) |

> 💡 **Interview tip:** Memorize the split — **System** check red = *AWS's hardware* (fix with Auto Recovery); **Instance** check red = *your OS* (OOM, full disk — you fix it). Interviewers love this "whose fault is it?" distinction.

```bash
# Configure auto-recovery on system check failure
aws cloudwatch put-metric-alarm \
  --alarm-name "AutoRecover-i-0abc123" \
  --metric-name StatusCheckFailed_System \
  --namespace AWS/EC2 \
  --dimensions Name=InstanceId,Value=i-0abc123 \
  --statistic Minimum --period 60 --evaluation-periods 2 --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --alarm-actions "arn:aws:automate:us-east-1:ec2:recover"
```

---

## 4. Placement Groups

> **In one line:** Placement groups tell AWS *how to physically spread your instances* — **Cluster** packs them together for speed, **Spread** scatters them for fault isolation, **Partition** balances both for big distributed systems.

```mermaid
flowchart TD
    A["🧩 Placement<br/>strategy?"] --> B{"Priority?"}
    B -->|"Lowest latency 🏎️"| C["✅ Cluster<br/>same rack + AZ,<br/>&lt;1ms, HPC / MPI"]
    B -->|"Max fault isolation 🛡️"| D["✅ Spread<br/>distinct racks,<br/>max 7 per AZ"]
    B -->|"Big distributed system ⚖️"| E["✅ Partition<br/>up to 7 partitions,<br/>1000s of nodes, HDFS"]
    C -.->|"trade-off"| F["⚠️ single AZ,<br/>same family only"]

    class A start
    class B proc
    class C,D,E good
    class F bad
    classDef start fill:#e3f2fd,stroke:#1565c0,color:#0d47a1,stroke-width:2px;
    classDef proc fill:#fff9c4,stroke:#f9a825,color:#000,stroke-width:2px;
    classDef good fill:#c8e6c9,stroke:#2e7d32,color:#1b5e20,stroke-width:2px;
    classDef bad fill:#ffcdd2,stroke:#c62828,color:#b71c1c,stroke-width:2px;
    classDef store fill:#ffe0b2,stroke:#e65100,color:#000,stroke-width:2px;
```

**Three strategies:**

**Cluster:** All instances on the same rack (or adjacent racks), same AZ. Lowest inter-node latency (10 Gbps enhanced networking, < 1 ms). Required for HPC, MPI, distributed ML training.
- Limitation: Must use same instance family. Start all instances simultaneously for best placement. Cannot span AZs.

**Spread:** Each instance on a distinct hardware rack. Maximum 7 instances per AZ. Provides maximum fault isolation for small critical clusters (Kafka brokers, ZooKeeper, leader nodes).

**Partition:** Instances spread across logical partitions (each partition = distinct rack set). Up to 7 partitions per AZ, thousands of instances. Provides partition-ID metadata for HDFS rack-awareness:

```bash
# Get partition number from within instance
curl -s http://169.254.169.254/latest/meta-data/placement/partition-number
```

---

## 5. Auto Scaling Groups & Launch Templates

### Beginner Foundation

**ASG** maintains a desired number of EC2 instances, replaces unhealthy ones, and scales based on demand. **Launch Templates** are versioned configuration blueprints.

> **In one line:** An ASG is a self-healing thermostat for capacity — you set a desired count and scaling rules, and it launches, health-checks, and replaces instances automatically. (See the full lifecycle in the [Visual Overview](#-visual-overview).)

### Intermediate Mechanics

**Scaling policy types:**

| Policy | How it works | Best for |
|---|---|---|
| Target Tracking | Maintain metric at target (CPU = 50%) | Most workloads — simplest |
| Step Scaling | Scale by N when alarm crosses threshold | Known load patterns |
| Scheduled Scaling | Scale at specific times | Predictable cycles (business hours) |
| Predictive Scaling | ML forecast + proactive scaling | Predictable but variable patterns |

```hcl
# Target tracking example
resource "aws_autoscaling_policy" "cpu" {
  name                   = "cpu-target-tracking"
  autoscaling_group_name = aws_autoscaling_group.app.name
  policy_type            = "TargetTrackingScaling"

  target_tracking_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ASGAverageCPUUtilization"
    }
    target_value     = 50.0
    disable_scale_in = false
  }
}
```

**Always use ELB health checks for web application ASGs:**
```hcl
resource "aws_autoscaling_group" "app" {
  health_check_type         = "ELB"  # Not default "EC2"
  health_check_grace_period = 300
  target_group_arns         = [aws_lb_target_group.app.arn]
}
```

> ⚠️ **Gotcha:** The default `EC2` health check only confirms the *OS is up* — a crashed app on a live OS still passes and receives traffic. Always set `health_check_type = "ELB"` for web apps so a failing `/health` endpoint pulls the instance out.

**Lifecycle hooks (graceful shutdown):**
```bash
# User-data: signal completion after initialization
aws autoscaling complete-lifecycle-action \
  --lifecycle-hook-name warmup-hook \
  --auto-scaling-group-name my-asg \
  --lifecycle-action-result CONTINUE \
  --instance-id $(curl -s http://169.254.169.254/latest/meta-data/instance-id)
```

**Mixed Instance Policy + Spot:**
```hcl
resource "aws_autoscaling_group" "app" {
  mixed_instances_policy {
    instances_distribution {
      on_demand_base_capacity                  = 2
      on_demand_percentage_above_base_capacity = 20
      spot_allocation_strategy                 = "capacity-optimized"
    }
    launch_template {
      launch_template_specification {
        launch_template_id = aws_launch_template.app.id
        version            = "$Latest"
      }
      # Multiple instance types for Spot resilience
      override { instance_type = "m5.large" }
      override { instance_type = "m5a.large" }
      override { instance_type = "m6i.large" }
      override { instance_type = "c5.xlarge"; weighted_capacity = 2 }
    }
  }
}
```

### Advanced Engineering

**Static stability under AZ impairment:** If one of 3 AZs fails, 2/3 of instances remain. Set `min_size` so 2 AZs can handle 100% load:
- 3 AZs, 9 instances desired → 3 per AZ → need 6 minimum to serve full load → `min_size = 6`

> 💡 **Interview tip:** "Static stability" is a favorite phrase — it means surviving an AZ loss *without needing to launch new instances* (which might fail during a regional event). Size `min_size` so the surviving AZs already carry full load.

**Warm pools:** Pre-initialize instances in stopped state for < 30-second scale-out (vs. 5–10 min cold boot). Cost: stopped instances cost only EBS. Benefit: eliminates scale-out latency for predictable burst events.

**Instance Refresh (rolling update):**
```bash
aws autoscaling start-instance-refresh \
  --auto-scaling-group-name my-asg \
  --preferences '{"MinHealthyPercentage": 90, "InstanceWarmup": 300}'
```

---

## 6. Spot Instances

### Beginner Foundation

**Spot Instances** = unused EC2 capacity at up to 90% discount. AWS reclaims with **2-minute notice**. Viable for production with stateless workloads, multiple instance types, and graceful interruption handling.

> **In one line:** Spot is AWS renting you its spare capacity cheap — you save up to 90% in exchange for a 2-minute eviction notice, so it only fits workloads that can checkpoint, retry, or shed a node gracefully.

**Interruption rates:** Typically 1–5% of instance-hours. Varies by instance type, AZ, and time. Check Spot Interruption Advisor in the EC2 console.

### Intermediate Mechanics

**Handle interruptions via IMDS and EventBridge:**
```bash
# Poll for interruption notice from within instance (every 5 seconds)
while true; do
  TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
    -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
  NOTICE=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
    http://169.254.169.254/latest/meta-data/spot/instance-action 2>&1)
  if echo "$NOTICE" | grep -q "terminate"; then
    echo "Spot interruption incoming! Graceful shutdown..."
    # Checkpoint state, drain connections, deregister from LB
    break
  fi
  sleep 5
done
```

**Spot allocation strategies:**
- `capacity-optimized`: Selects pools with most available capacity → lowest interruption risk. **Recommended for production.**
- `price-capacity-optimized`: Balance between price and capacity. AWS recommended default.
- `lowest-price`: Highest interruption risk (all customers compete for cheapest pool).

**Always diversify instance types:** Specify 5+ types with similar vCPU/memory profiles. If one pool is exhausted, ASG/EC2 Fleet uses another.

### Advanced Engineering

**Spot with Karpenter (EKS):** Karpenter automatically handles Spot interruptions by pre-provisioning replacement nodes and draining the interrupted node:

```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: spot-workers
spec:
  template:
    spec:
      requirements:
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot"]
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64", "arm64"]
      nodeClassRef:
        name: default
  disruption:
    consolidationPolicy: WhenUnderutilized
    budgets:
      - nodes: "20%"  # Never disrupt more than 20% of nodes simultaneously
```

---

## 7. Reserved Instances & Savings Plans

> **In one line:** You trade a 1- or 3-year commitment for a discount — **Compute Savings Plans** are the flexible modern default (any family/region/OS), while classic Reserved Instances lock you to more specifics.

### Intermediate Mechanics

**Savings Plans (preferred):**

| Plan Type | Commitment | Flexibility |
|---|---|---|
| Compute Savings Plans | $/hr of any EC2/Lambda/Fargate | Any family, region, OS — most flexible |
| EC2 Instance Savings Plans | $/hr of specific family in region | Any OS, size in that family |
| SageMaker Savings Plans | $/hr of SageMaker | Any SageMaker instance |

**Purchasing strategy:**
1. Use Cost Explorer → Savings Plans recommendations.
2. Cover 70–80% of stable baseline (not peak).
3. Use Spot for variable/bursty above baseline.
4. Purchase Compute Savings Plans (most flexible) over Reserved Instances (more restrictive).

```bash
# Get Compute SP recommendation
aws savingsplans get-savings-plans-purchase-recommendation \
  --savings-plans-type COMPUTE_SP \
  --term-in-years ONE_YEAR \
  --payment-option PARTIAL_UPFRONT \
  --lookback-period-in-days SIXTY_DAYS
```

**Organization sharing:** Savings Plans and RIs in any member account apply to usage across the entire Organization (consolidated billing). Cannot restrict sharing per-account without opting out of RI sharing.

---

## 8. AWS Lambda Deep Dive

### Beginner Foundation

**Lambda** = event-driven FaaS. Upload code → configure trigger → Lambda executes on-demand. Pay per millisecond of execution. No servers to manage.

> **In one line:** Lambda runs your code in a fresh Firecracker microVM per concurrent request, bills per millisecond, and the whole latency conversation revolves around avoiding the **cold start** (spinning up that microVM + runtime + init code).

**Key limits:** 15 min max duration, 10 GB memory, 10 GB container image, 6 MB sync payload, 256 KB async payload, 1,000 concurrent executions/Region (default).

### Intermediate Mechanics

**Cold start anatomy:**

```mermaid
sequenceDiagram
    participant Trigger as 📨 Event Source
    participant Lambda as ⚙️ Lambda Service
    participant VM as 🔥 Firecracker microVM
    participant Code as 🧑‍💻 Function Code

    Trigger->>Lambda: Invoke
    Lambda->>VM: Create new microVM (cold start only)
    VM->>VM: Download & unpack code package
    VM->>Code: Start runtime (JVM/Python/Node interpreter)
    Code->>Code: Run global initialization (SDK clients, DB pools)
    Code->>Code: Execute handler
    Code-->>Trigger: Response
    Note over VM: 🟢 Warm for ~5-15 min; next invoke skips all above
```

**Cold start mitigation:**

| Cause | Solution |
|---|---|
| Java/Spring (3+ s) | Lambda SnapStart (snapshots init JVM) |
| Large package size | Reduce deps, use Lambda Layers |
| Slow global init | Move SDK clients to global scope |
| High p99 / spiky load | Provisioned Concurrency |

**Global scope optimization (critical):**
```python
import boto3

# Global scope: runs ONCE per execution environment
dynamodb = boto3.resource('dynamodb')  # SDK initialization ~50ms
table = dynamodb.Table('users')

def handler(event, context):
    # Per-invocation: reuses initialized client
    return table.get_item(Key={'user_id': event['user_id']})['Item']
```

> 💡 **Interview tip:** The single biggest "free" Lambda win is moving SDK clients and DB pools to **global scope** so they initialize once per environment, not once per invoke. Interviewers expect you to name this before reaching for Provisioned Concurrency.

**VPC Lambda considerations:** Lambda in VPC routes traffic via your NAT Gateway (for internet) or VPC Endpoints (for AWS services). Ensure Interface Endpoints for all accessed AWS services to avoid NAT Gateway egress costs and improve latency.

```hcl
resource "aws_lambda_function" "api" {
  function_name = "api-handler"
  runtime       = "python3.12"
  handler       = "handler.main"
  role          = aws_iam_role.lambda.arn
  filename      = "lambda.zip"

  vpc_config {
    subnet_ids         = [aws_subnet.private_a.id, aws_subnet.private_b.id]
    security_group_ids = [aws_security_group.lambda.id]
  }
}
```

### Advanced Engineering

**Provisioned Concurrency (eliminates cold starts):**
```hcl
resource "aws_lambda_provisioned_concurrency_config" "api" {
  function_name                  = aws_lambda_function.api.function_name
  qualifier                      = aws_lambda_alias.live.name
  provisioned_concurrent_executions = 10
}
```

**Lambda SnapStart (Java 11+):** Snapshots the initialized JVM state. Restores on cold start instead of re-running initialization. 3–5 s Java cold starts → < 200 ms.

**Lambda Power Tuning:** CPU scales linearly with memory. 512 MB at 1 s = same cost as 1,024 MB at 0.5 s. Always tune:
```bash
# Open-source Lambda Power Tuning tool
npx lambda-power-tuning --function-name my-function --payload '{}' --strategy cost
```

**Concurrency math:**
```
Concurrent executions = Requests/sec × Average duration (seconds)
Example: 500 req/sec × 0.2 s avg = 100 concurrent executions needed
At 1,000 account limit: leaves 900 for other functions
```

**Async invocations + destinations:**
```bash
# Route failed async invocations to SQS for inspection
aws lambda put-function-event-invoke-config \
  --function-name my-function \
  --destination-config '{"OnFailure":{"Destination":"arn:aws:sqs:us-east-1:123:failed-events"}}'
```

---

## 9. AWS Fargate & App Runner

**Fargate** = serverless compute for containers. Define CPU/memory per task/pod; AWS manages the underlying EC2 nodes.

> **In one line:** Fargate is "serverless containers" — you hand AWS a task/pod spec and it runs each one in its own microVM, so you never patch or scale nodes, at the cost of no DaemonSets, GPU, or EBS.

**Fargate vs. EC2 Nodes for EKS:**

| Dimension | Fargate | EC2 Managed Nodes |
|---|---|---|
| Node management | None | Must update node groups |
| Scale speed | Instant (pod = VM) | 2–5 min node launch |
| Isolation | Per-pod microVM | Shared node kernel |
| DaemonSets | Not supported | Supported |
| GPU | Not supported | Supported |
| EBS volumes | Not supported | Supported |
| Best for | Stateless, security-sensitive, bursty | Stateful, DaemonSet-dependent, GPU |

**App Runner:** Zero-configuration managed service. Provide ECR image or GitHub repo → App Runner builds, deploys, scales, and terminates containers. No ALB, no ASG, no VPC config required. Best for small teams prioritizing speed over control.

---

## 15. Interview Questions & Answers

---

### Question 1: What happens during an ASG scale-out event and how do you ensure new instances serve traffic only when ready?

**What the interviewer is testing:** ASG mechanics, health check integration, graceful rollout.

**Strong answer:**

**Scale-out flow:**
1. CloudWatch alarm fires (CPU > 70% for 2 consecutive 5-minute periods).
2. ASG increases desired capacity by N instances.
3. ASG selects AZs (balancing toward AZs with fewest instances).
4. Calls `RunInstances` with the Launch Template.
5. Instance transitions: pending → running.
6. **Health check grace period starts** (default 300 s) — no health checks during this time. Allows OS boot + application start.
7. After grace period: ELB health checks begin (HTTP GET `/health` → must return 2xx).
8. Only after health check passes does the instance join the target group and receive traffic.

**Critical config choices:**

`health_check_type = "ELB"` (not default "EC2"): EC2 health checks only verify the OS is up. ELB health checks verify the application is responding. A crashed application that still has an OS running would pass EC2 checks and receive traffic.

**For complex initialization (DB migration, cache warming):** Use lifecycle hooks to pause instances in `Pending:Wait`. Your code runs initialization, then calls:
```bash
aws autoscaling complete-lifecycle-action \
  --lifecycle-hook-name warmup-hook \
  --auto-scaling-group-name my-asg \
  --lifecycle-action-result CONTINUE \
  --instance-id $(curl -s http://169.254.169.254/latest/meta-data/instance-id)
```

**Scale-in protection:** During scale-in, lifecycle hook pauses the instance in `Terminating:Wait`. Your code drains active connections (deregister from target group, wait for deregistration delay), flushes logs, then signals completion.

**Likely follow-ups:**
1. *What is the termination policy for ASG scale-in?* — Default: OldestLaunchTemplate → AZ imbalance correction → OldestInstance. You can customize: `ClosestToNextInstanceHour` saves RI cost by terminating instances near their billing hour. `NewestInstance` is useful for canary rollbacks.
2. *How does Instance Refresh work for rolling AMI updates?* — Sets a `MinHealthyPercentage` and `InstanceWarmup`. ASG terminates old instances in batches, waiting for new ones to pass health checks before continuing. Equivalent to a controlled rolling deployment with automatic rollback if health checks fail.

---

### Question 3: What is Lambda cold start and how would you fix it for a customer-facing API with p99 latency SLO of < 100ms?

**What the interviewer is testing:** Lambda performance internals, optimization strategies, trade-off analysis.

**Strong answer:**

A cold start occurs when Lambda creates a new execution environment — a Firecracker microVM is spun up, the function package is downloaded and unpacked, the language runtime is started, and global initialization code runs. This takes 100 ms to 3+ seconds depending on runtime and package size.

**Diagnosing cold starts:**
```bash
aws logs start-query \
  --log-group-name "/aws/lambda/my-api" \
  --query-string 'filter @type = "REPORT"
    | stats 
        count(@initDuration) as coldStartCount,
        avg(@initDuration) as avgInitMs,
        max(@initDuration) as maxInitMs,
        count(*) as totalRequests
      by bin(5m)'
```

**For a < 100ms p99 SLO:**

100 ms p99 is extremely aggressive for Lambda with cold starts. The approach depends on traffic patterns:

**Option A — Provisioned Concurrency (best for consistent < 100ms p99):**
Pre-warm N execution environments. Cold starts eliminated for those N instances. Cost: ~$0.015/hour per provisioned instance at 1 GB memory.

```hcl
resource "aws_lambda_provisioned_concurrency_config" "api" {
  function_name                  = aws_lambda_function.api.function_name
  qualifier                      = aws_lambda_alias.live.name
  provisioned_concurrent_executions = 20  # Size based on concurrent traffic
}
```

Use Application Auto Scaling to scale PC up during peak hours and down overnight:
```hcl
resource "aws_appautoscaling_scheduled_action" "scale_up" {
  name               = "scale-up-business-hours"
  service_namespace  = "lambda"
  resource_id        = "function:${aws_lambda_function.api.function_name}:live"
  scalable_dimension = "lambda:function:ProvisionedConcurrency"
  schedule           = "cron(0 8 * * ? *)"  # 8 AM UTC
  scalable_target_action {
    min_capacity = 50
    max_capacity = 50
  }
}
```

**Option B — Lambda SnapStart (Java only, free):**
For Java functions, SnapStart snapshots the initialized JVM. 3+ s cold starts → < 200 ms. Note: random data (UUID, timestamps) generated in global scope during snapshot restore will be the same unless explicitly refreshed.

**Option C — Architecture change (if < 100ms is truly critical for ALL requests):**
Move to ECS/Fargate or EKS with always-on pods. Zero cold starts, consistent latency, but always-on cost. Better for < 50 ms p99 requirements.

**Optimize regardless of above:**
- Move all SDK/DB client initialization to global scope.
- Reduce package size (Lambda Layers, tree-shaking, exclude test dependencies).
- Use Python or Node.js for lower base cold start than Java/.NET.
- Increase memory (reduces cold start duration by speeding up package load and initialization).

**Likely follow-ups:**
1. *What happens when Lambda has more concurrent requests than provisioned concurrency instances?* — Requests above the provisioned count are handled by on-demand instances with cold starts. Use Application Auto Scaling on provisioned concurrency with target tracking on `ProvisionedConcurrencyUtilization` metric.
2. *How do you size provisioned concurrency?* — Concurrent Lambda invocations = requests/sec × avg duration. If 500 req/s at 50 ms avg: 500 × 0.05 = 25 concurrent. Set PC to 30 (20% buffer). Monitor `ConcurrentExecutions` and `ProvisionedConcurrencyUtilization` and adjust.

---

## 16. Troubleshooting Scenarios

### Scenario 1: "Lambda function suddenly 504 timeout errors from API Gateway. Function was working fine yesterday."

**Symptom:** API Gateway returns 504 Gateway Timeout. Lambda logs show no invocations after a certain time, or logs show execution exceeding 29 seconds (API Gateway's max integration timeout).

**Investigation:**

```bash
# Step 1: Check Lambda function timeout setting
aws lambda get-function-configuration --function-name my-api-function \
  --query '{Timeout:Timeout,MemorySize:MemorySize,VpcConfig:VpcConfig}'

# Step 2: Check recent Lambda errors and duration
aws cloudwatch get-metric-statistics \
  --namespace AWS/Lambda --metric-name Duration \
  --dimensions Name=FunctionName,Value=my-api-function \
  --start-time $(date -d '2 hours ago' -u +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 300 --statistics Maximum,p99

# Step 3: Check Lambda throttle errors
aws cloudwatch get-metric-statistics \
  --namespace AWS/Lambda --metric-name Throttles \
  --dimensions Name=FunctionName,Value=my-api-function \
  --period 300 --statistics Sum

# Step 4: Check Lambda logs for errors
aws logs filter-log-events \
  --log-group-name "/aws/lambda/my-api-function" \
  --start-time $(date -d '2 hours ago' +%s000) \
  --filter-pattern "ERROR Task timed out"

# Step 5: If Lambda is in VPC, check VPC connectivity
aws ec2 describe-nat-gateways \
  --filter Name=state,Values=available Name=vpc-id,Values=vpc-0abc123
```

**Plausible cause 1:** Lambda timeout increased API calls from an external API that became slow/down. The Lambda function waits indefinitely for the external API response, consuming its 29-second (or configured) timeout.

**Plausible cause 2:** Lambda in VPC — NAT Gateway became unhealthy. Lambda can't reach external APIs or AWS services (if no VPC endpoints configured). Previous invocations used cached TCP connections; new connections fail.

**Plausible cause 3:** Database connection pool exhaustion. Lambda scaled to high concurrency, each instance holding a DB connection. DB max_connections exceeded → Lambda waits for available connection → timeout.

**Root cause identification:**

If NAT Gateway issue:
```bash
# Check NAT Gateway metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/NATGateway \
  --metric-name ErrorPortAllocation \
  --dimensions Name=NatGatewayId,Value=nat-0abc123 \
  --period 60 --statistics Sum

# Check ENI errors in VPC Flow Logs
aws logs start-query \
  --log-group-name /vpc/flow-logs \
  --query-string 'filter action = "REJECT" and srcAddr like "10.0." | sort @timestamp desc | limit 20'
```

**Fix for DB connection exhaustion:** Use RDS Proxy (connection pooler) between Lambda and RDS. RDS Proxy maintains a pool of DB connections and multiplexes Lambda's ephemeral connections through the pool. Reduces DB connections from `concurrency × 1` to a manageable pool size.

---

## 17. Production Best Practices

**EC2:**
- Use Graviton (ARM64) instances as default for new workloads — 20–40% better price/performance.
- Require IMDSv2 via SCP: Deny `ec2:RunInstances` if `MetadataHttpTokens != required`.
- Always use Launch Templates (not Launch Configurations — deprecated).
- Enable EC2 Auto Recovery for stateful single-instance workloads.
- Use ASGs for all stateless workloads — even a "single-instance" stateless app benefits from auto-replacement.

**Auto Scaling:**
- `health_check_type = "ELB"` for all web ASGs.
- Set minimum capacity so remaining AZs handle full load after one AZ failure.
- Implement lifecycle hooks for all stateful lifecycle events (drain before termination, initialize before traffic).

**Lambda:**
- Initialize SDK clients and DB connections in global scope (runs once per execution environment).
- Use Provisioned Concurrency for customer-facing latency-sensitive functions.
- Run Lambda Power Tuning before going to production — often 2× memory = same cost with better performance.
- Always set DLQ or destination for async invocations.
- Use RDS Proxy to prevent DB connection exhaustion at high concurrency.

---

## 18. Documentation Links

| Topic | Official Link |
|---|---|
| EC2 Instance Types | https://aws.amazon.com/ec2/instance-types/ |
| Nitro System | https://aws.amazon.com/ec2/nitro/ |
| EC2 Auto Scaling | https://docs.aws.amazon.com/autoscaling/ec2/userguide/what-is-amazon-ec2-auto-scaling.html |
| Spot Instances | https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-spot-instances.html |
| Lambda Developer Guide | https://docs.aws.amazon.com/lambda/latest/dg/welcome.html |
| Lambda SnapStart | https://docs.aws.amazon.com/lambda/latest/dg/snapstart.html |
| Lambda Power Tuning | https://github.com/alexcasalboni/aws-lambda-power-tuning |
| Savings Plans | https://docs.aws.amazon.com/savingsplans/latest/userguide/what-is-savings-plans.html |
| RDS Proxy | https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/rds-proxy.html |

---

*Continue to [05-STORAGE.md](./05-STORAGE.md) for Section 5 (AWS Storage).*
