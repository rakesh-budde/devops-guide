# Azure DevOps — Interview Preparation Guide

> **300+ Azure DevOps interview questions and deep-dive internals for Senior DevOps Engineer, SRE, and Platform Engineer roles at FAANG-level companies.**
>
> **Scope:** Six section-wise files covering the full Azure DevOps platform — Boards, Repos, Pipelines, Artifacts & Release, Security, and Troubleshooting — each with interview-worthy Q&A, execution-model internals, colorful Mermaid diagrams, and memory hooks.

---

## 🗺️ Repo-Wide Mind Map

**Skim this first, revisit it last — the whole guide at a glance:**

```mermaid
mindmap
  root((Azure DevOps))
    Overview and Boards
      Five services
      Organization and projects
      Work items and backlogs
      Process models
      Traceability
    Repos
      Git and TFVC
      Branch policies
      Pull requests
      Merge strategies
    Pipelines
      Classic vs YAML
      Stages jobs steps tasks
      Agents and pools
      Variables and expressions
      Templates
    Artifacts and Release
      Feeds and upstream
      Environments
      Approvals and gates
      Deployment strategies
    Security
      Service connections
      Workload Identity Federation
      Secrets and Key Vault
      Agent security
    Troubleshooting
      Pipeline failures
      Agent problems
      Permissions
      Flaky and stuck runs
```

---

## 📚 Master Table of Contents

| # | Section | File | Key Topics |
|---|---------|------|------------|
| 1 | **Overview & Boards** | [01-OVERVIEW-BOARDS.md](./01-OVERVIEW-BOARDS.md) | The five services, organization/project/team hierarchy, work items, backlogs & sprints, process models (Basic/Agile/Scrum/CMMI), area & iteration paths, WIQL, end-to-end traceability |
| 2 | **Azure Repos** | [02-REPOS.md](./02-REPOS.md) | Git vs TFVC, branch policies, pull-request workflow & votes, build validation on the merge result, required/auto reviewers, merge strategies, forks |
| 3 | **Azure Pipelines** | [03-PIPELINES.md](./03-PIPELINES.md) | Classic vs YAML, Stage→Job→Step→Task model, agents & pools, triggers, variables & variable groups, compile-time vs runtime expressions, templates & `extends`, enterprise templating |
| 4 | **Artifacts & Release** | [04-ARTIFACTS-RELEASE.md](./04-ARTIFACTS-RELEASE.md) | Feeds, upstream sources & caching, views & retention, environments, approvals/checks/gates, deployment strategies (runOnce/rolling/canary/blue-green) |
| 5 | **Security** | [05-SECURITY.md](./05-SECURITY.md) | Service connections, Workload Identity Federation (OIDC), Key Vault secrets, pipeline authorization scope, fork protections, agent security, RBAC |
| 6 | **Troubleshooting** | [06-TROUBLESHOOTING.md](./06-TROUBLESHOOTING.md) | The diagnostic method, pipeline failures, agent/pool problems, permission errors, flaky/stuck runs, expression-timing bugs, dependency failures, debugging tools |

---

## 🎯 How to Use This Guide

Each section file follows a consistent, interview-focused structure:

- **`## 🗺️ Visual Overview`** — a Mermaid `mindmap` of the section plus 2–3 colorful flow diagrams of the hard-to-visualize mechanics (pipeline execution, OIDC token exchange, approval gating), with **memory hooks** (mnemonics).
- **Depth topics** — each tagged with an **`🎯 Interview weight`**, an **`In one line`** summary, then tables, bold terms, and `🧠 / 💡 / ⚠️ / 🔍` callouts that separate internals from filler.
- **`## Interview Questions and Answers`** — interview-worthy questions only, each answered as **answer → internals → follow-up**.
- **Troubleshooting / Best Practices / Documentation Links** — practical close-out and official references.
- **Navigation footer** — move linearly through the six sections.

---

## 🧭 Recommended Study Order

The files are ordered as a natural learning path — read them 1 → 6:

```mermaid
flowchart LR
    S1["1️⃣ Overview<br/>and Boards"] --> S2["2️⃣ Repos"]
    S2 --> S3["3️⃣ Pipelines"]
    S3 --> S4["4️⃣ Artifacts<br/>and Release"]
    S4 --> S5["5️⃣ Security"]
    S5 --> S6["6️⃣ Trouble<br/>shooting"]
    class S1 start
    class S2 good
    class S3 proc
    class S4 store
    class S5 ctrl
    class S6 bad
    classDef start fill:#e3f2fd,stroke:#1565c0,color:#0d47a1,stroke-width:2px;
    classDef proc  fill:#fff9c4,stroke:#f9a825,color:#000,stroke-width:2px;
    classDef good  fill:#c8e6c9,stroke:#2e7d32,color:#1b5e20,stroke-width:2px;
    classDef bad   fill:#ffcdd2,stroke:#c62828,color:#b71c1c,stroke-width:2px;
    classDef ctrl  fill:#e1bee7,stroke:#6a1b9a,color:#4a148c,stroke-width:2px;
    classDef store fill:#ffe0b2,stroke:#e65100,color:#000,stroke-width:2px;
```

| If you have… | Focus on |
|---|---|
| **1 evening** | Sections 3 (Pipelines) + 5 (Security) — the highest-yield interview topics |
| **A weekend** | Sections 2–5 (Repos → Pipelines → Artifacts/Release → Security) |
| **A week** | All six, in order, doing every Q&A before reading the answer |
| **Panel tomorrow** | The `🗺️ Visual Overview` + memory hooks of every section, then re-read `🎯 Interview weight: HIGH` topics |

> 🧠 **One-line platform summary:** Azure DevOps is one **organization** of **projects**, each bundling five services — **B**oards, **R**epos, **P**ipelines, **A**rtifacts, **T**est Plans — wired together by shared identity and an end-to-end traceability chain from work item to production deployment.

---

**[← Back to Main README](../README.md)** | **[Start: Overview & Boards →](./01-OVERVIEW-BOARDS.md)**
