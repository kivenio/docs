# Kiven — Architecture Overview
## *Managed Data Services, On Your Infrastructure*
### *Version 2.0 — February 2026*

---

> **This document is the entry point for Kiven's architecture.**
> It provides a high-level overview and links to detailed documentation.

---

# PART I — EXECUTIVE SUMMARY

## 1.1 What Is Kiven

Kiven is a **fully managed data platform** that runs on the customer's own Kubernetes infrastructure. Starting with PostgreSQL (powered by CloudNativePG), Kiven delivers an Aiven-quality experience — but the data never leaves the customer's cluster.

**How it works:**
1. Customer signs up → grants Kiven access to their EKS (cross-account IAM Role)
2. Kiven provisions everything: dedicated nodes, storage, S3 backups, CNPG operator, PostgreSQL
3. Customer gets: a connection string + a dashboard
4. Kiven manages everything from that point: scaling, backups, monitoring, security, tuning

**The customer never touches kubectl, YAML, CNPG, or Kubernetes internals.**

### Value Proposition

| vs. Aiven | vs. Self-Managed CNPG | vs. Launchly |
|-----------|----------------------|-------------|
| Same UX, but on customer's infra | Same PostgreSQL, but fully managed | Same CNPG, but Aiven-level depth |
| 40-60% cheaper (no Aiven markup) | No need for K8s/CNPG expertise | Full infra management (nodes, storage) |
| Data never leaves customer's VPC | Risk eliminated by best practices | DBA intelligence built-in |

## 1.2 Scope

Kiven is designed for:
- **Scalability**: Support 100+ customer clusters across multiple EKS environments
- **Reliability**: RPO 1h, RTO 15min (Kiven SaaS); RPO 5min, RTO 5min (customer databases via CNPG)
- **Compliance**: GDPR (EU data residency), SOC2 (audit, RBAC, encryption)
- **Extensibility**: Provider/plugin architecture for multi-operator future (Kafka, Redis, Elasticsearch)
- **Lifespan**: 5+ years

### Non-Goals (Phase 1)
- Multi-cloud support (GKE, AKS) — Phase 3
- Non-PostgreSQL data services (Kafka, Redis) — Phase 3
- Self-hosted / air-gapped edition — Phase 3
- Mobile app

## 1.3 Key Parameters

| Parameter | Value | Impact |
|-----------|-------|--------|
| **RPO (Kiven SaaS)** | 1 hour | Hourly backups of product database |
| **RTO (Kiven SaaS)** | 15 minutes | Automated failover |
| **RPO (Customer DBs)** | Configurable (1min–24h) | Continuous WAL archiving via Barman |
| **RTO (Customer DBs)** | < 5 minutes | CNPG automatic failover, multi-AZ |
| **Provisioning time** | < 10 minutes | From "Create Database" to connection string |
| **Agent footprint** | < 50MB RAM, < 0.1 CPU | Minimal impact on customer cluster |
| **On-call team** | 5 people | Runbooks for both SaaS and customer infra |

## 1.4 Compliance Summary

| Standard | Key Requirements | Scope |
|----------|-----------------|-------|
| **GDPR** | EU data residency, right to erasure, DPA | Kiven SaaS (eu-west-1) + customer data stays in their infra |
| **SOC2** | RBAC, audit logging, encryption, incident response | Kiven SaaS operations + customer infra access audit trail |

> Note: PCI-DSS is NOT in scope. Kiven does not process payment card data. Customer compliance (HIPAA, PCI, etc.) is helped by data staying on their own infra.

## 1.5 Tech Stack Overview

### Kiven SaaS Platform

| Category | Choice | Rationale |
|----------|--------|-----------|
| **Cloud** | AWS (eu-west-1) | GDPR, proximity to EU customers |
| **Orchestration** | EKS + ArgoCD | GitOps, cloud-native |
| **Backend** | Go (stdlib + chi) | K8s ecosystem is Go, fast, small binaries |
| **Frontend** | Next.js 14+ (App Router) + Tailwind + shadcn/ui | Modern, fast, beautiful |
| **Agent** | Go (client-go + controller-runtime) | Native K8s SDK, single binary |
| **Agent Comms** | gRPC + mTLS | Secure, efficient, bidirectional streaming |
| **Product DB** | PostgreSQL (Aiven) | Dogfooding the ecosystem, managed |
| **Cache** | Valkey | Sessions, rate limiting, real-time state |
| **Messaging** | Kafka (Aiven) | Agent events, audit trail, async operations |
| **Edge/CDN** | Cloudflare | WAF, DDoS, Zero Trust, Tunnel |
| **Observability** | Prometheus / Loki / Tempo | Self-hosted, cost-efficient |
| **Secrets** | HashiCorp Vault | Dynamic secrets, rotation, IRSA |
| **CNI** | Cilium | mTLS, Gateway API, network policies |
| **Policies** | Kyverno | Admission control, pod security |
| **Billing** | Stripe | SaaS billing, per-cluster pricing |
| **CI/CD** | GitHub Actions | Already in place |
| **IaC** | Terraform | Infrastructure as Code |

### Customer-Side (Provisioned by Kiven)

| Component | Technology | Managed By |
|-----------|-----------|------------|
| **Kubernetes nodes** | EKS Managed Node Groups | Kiven (via AWS API) |
| **PostgreSQL** | CloudNativePG (CNPG) | Kiven (via agent) |
| **Connection pooling** | PgBouncer (CNPG Pooler CRD) | Kiven (via agent) |
| **Backups** | Barman → S3 | Kiven (via agent + AWS API) |
| **Storage** | EBS gp3 (encrypted, KMS) | Kiven (via AWS API) |
| **Backup storage** | S3 bucket (encrypted, lifecycle) | Kiven (via AWS API) |
| **TLS** | cert-manager + self-signed CA | Kiven (via agent) |
| **Monitoring agent** | Kiven Agent (Go) | Kiven |

---

# PART II — ARCHITECTURE

## 2.1 System Context (C4 Level 1)

```
┌──────────────────────────────────────────────────────────────────────────┐
│                              USERS                                        │
│         Developers (Simple Mode)    DevOps (Advanced Mode)               │
└──────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                         CLOUDFLARE EDGE                                   │
│                (DNS, WAF, DDoS, CDN, Zero Trust)                         │
└──────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                      KIVEN SaaS PLATFORM                                  │
│                    (AWS EKS — eu-west-1)                                  │
│                                                                           │
│  Dashboard + API + CLI + Terraform Provider                              │
│  Core Services: provisioner, infra, clusters, backups, monitoring...     │
│  Provider/Plugin: CNPG Provider (Phase 1), Strimzi (future)...          │
└──────────────────────────────────────────────────────────────────────────┘
              │                                       │
              │ gRPC/mTLS (Agent)                     │ Cross-Account
              │                                       │ IAM AssumeRole
              ▼                                       ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                    CUSTOMER'S AWS ACCOUNT / EKS                           │
│                                                                           │
│  ┌──── Managed by Kiven ──────────────────────────────────────────────┐  │
│  │  Node Group: kiven-db-nodes (dedicated, tainted, multi-AZ)        │  │
│  │  Namespace: kiven-system (agent + CNPG operator)                   │  │
│  │  Namespace: kiven-databases (PostgreSQL clusters)                  │  │
│  │  S3 Bucket: kiven-backups-{customer-id}                           │  │
│  │  IAM: IRSA roles for S3 access                                    │  │
│  │  CNPG: PostgreSQL Primary + Replicas + PgBouncer                  │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                                                           │
│  ┌──── Managed by Customer ───────────────────────────────────────────┐  │
│  │  Their app nodes, services, workloads                              │  │
│  │  Connect to: pg-main.kiven-databases.svc:5432                      │  │
│  └────────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────┘
```

## 2.2 Container Diagram (C4 Level 2) — Kiven SaaS

```
┌──────────────────────────────────────────────────────────────────────────┐
│                  KIVEN SaaS — AWS WORKLOAD ACCOUNT — eu-west-1           │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │                           EKS CLUSTER                                │ │
│  │                                                                      │ │
│  │   ┌──────────────────────────────────────────────────────────────┐  │ │
│  │   │ PLATFORM NODE POOL (taints: platform=true:NoSchedule)        │  │ │
│  │   │ • ArgoCD         • Cilium          • Vault Agent             │  │ │
│  │   │ • OTel Collector • Prometheus      • Grafana                 │  │ │
│  │   │ • Loki           • Tempo           • Kyverno                 │  │ │
│  │   └──────────────────────────────────────────────────────────────┘  │ │
│  │                                                                      │ │
│  │   ┌──────────────────────────────────────────────────────────────┐  │ │
│  │   │ APPLICATION NODE POOL (auto-scaling)                         │  │ │
│  │   │                                                              │  │ │
│  │   │  ┌── Core ─────────────────────────────────────────────┐    │  │ │
│  │   │  │ svc-api          svc-auth          svc-provisioner  │    │  │ │
│  │   │  │ svc-infra        svc-clusters      svc-agent-relay  │    │  │ │
│  │   │  └─────────────────────────────────────────────────────┘    │  │ │
│  │   │                                                              │  │ │
│  │   │  ┌── Data Services ────────────────────────────────────┐    │  │ │
│  │   │  │ svc-backups      svc-monitoring    svc-users        │    │  │ │
│  │   │  │ svc-yamleditor   svc-migrations                     │    │  │ │
│  │   │  └─────────────────────────────────────────────────────┘    │  │ │
│  │   │                                                              │  │ │
│  │   │  ┌── Business ────────────────────────────────────────┐     │  │ │
│  │   │  │ svc-billing      svc-audit         svc-notification│     │  │ │
│  │   │  └─────────────────────────────────────────────────────┘    │  │ │
│  │   └──────────────────────────────────────────────────────────────┘  │ │
│  │                                                                      │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
│                                    │                                      │
│                                    │ VPC Peering                          │
│                                    ▼                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │                          AIVEN VPC                                   │ │
│  │  • PostgreSQL (Kiven product database)                              │ │
│  │  • Kafka (agent events, audit trail, async ops)                     │ │
│  │  • Valkey (sessions, rate limiting, cache)                          │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
│                                                                           │
└──────────────────────────────────────────────────────────────────────────┘
```

## 2.3 Container Diagram (C4 Level 2) — Customer Side

```
┌──────────────────────────────────────────────────────────────────────────┐
│                  CUSTOMER'S EKS CLUSTER                                   │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │ NODE GROUP: kiven-db-nodes (Managed by Kiven)                       │ │
│  │ Instance: r6g.medium–r6g.2xlarge (memory-optimized)                 │ │
│  │ Taint: kiven.io/role=database:NoSchedule                            │ │
│  │ Multi-AZ: primary in AZ-a, replica in AZ-b                         │ │
│  │                                                                      │ │
│  │   ┌── Namespace: kiven-system ──────────────────────────────────┐   │ │
│  │   │ Kiven Agent (Go)         — gRPC → Kiven SaaS               │   │ │
│  │   │ CNPG Operator            — manages PG clusters              │   │ │
│  │   │ cert-manager (optional)  — TLS certificates                 │   │ │
│  │   └─────────────────────────────────────────────────────────────┘   │ │
│  │                                                                      │ │
│  │   ┌── Namespace: kiven-databases ───────────────────────────────┐   │ │
│  │   │                                                             │   │ │
│  │   │  CNPG Cluster: pg-production-main                           │   │ │
│  │   │  ├─ Pod: pg-production-main-1 (Primary, AZ-a)              │   │ │
│  │   │  ├─ Pod: pg-production-main-2 (Replica, AZ-b)              │   │ │
│  │   │  ├─ Pod: pg-production-main-3 (Replica, AZ-c)              │   │ │
│  │   │  ├─ Service: pg-production-main-rw (read-write)            │   │ │
│  │   │  ├─ Service: pg-production-main-ro (read-only)             │   │ │
│  │   │  └─ Pooler: pg-production-main-pooler (PgBouncer)          │   │ │
│  │   │                                                             │   │ │
│  │   │  ScheduledBackup → S3: kiven-backups-{customer-id}         │   │ │
│  │   │  NetworkPolicy: only kiven-databases + customer-app-ns      │   │ │
│  │   └─────────────────────────────────────────────────────────────┘   │ │
│  │                                                                      │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
│                                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │ NODE GROUP: customer-app-nodes (Managed by Customer)                │ │
│  │ • Customer's application pods                                       │ │
│  │ • Connect to: pg-production-main-pooler.kiven-databases.svc:5432   │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
│                                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │ AWS Resources (Managed by Kiven via cross-account IAM)              │ │
│  │ • EBS gp3 volumes (encrypted, KMS)                                  │ │
│  │ • S3 bucket: kiven-backups-{customer-id}                            │ │
│  │ • IAM IRSA role: kiven-cnpg-backup-role                             │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
│                                                                           │
└──────────────────────────────────────────────────────────────────────────┘
```

## 2.4 Core Services

### Service Catalog

| Service | Responsibility | Language | Priority |
|---------|---------------|----------|----------|
| **svc-api** | REST + GraphQL gateway, request routing | Go | P0 |
| **svc-auth** | OIDC (Google/GitHub/SAML), RBAC, API keys, org/team model | Go | P0 |
| **svc-provisioner** | **THE BRAIN** — Orchestrates full provisioning pipeline (nodes → storage → S3 → CNPG → PG) | Go | P0 |
| **svc-infra** | AWS resource management in customer accounts (EC2, EBS, S3, IAM, KMS) | Go | P0 |
| **svc-clusters** | Cluster lifecycle via provider interface (status, scale, upgrade, delete) | Go | P0 |
| **svc-backups** | Backup/restore management, PITR, fork/clone, backup verification | Go | P0 |
| **svc-monitoring** | Metrics ingestion from agents, DBA intelligence, alerts engine | Go | P0 |
| **svc-users** | Database user/role management, permissions, pg_hba rules | Go | P0 |
| **svc-agent-relay** | gRPC server, multiplexes all customer agent connections | Go | P0 |
| **svc-yamleditor** | YAML generation, schema validation, diff engine, change history | Go | P0 |
| **svc-migrations** | Import from Aiven/RDS/bare PG into Kiven-managed clusters | Go | P1 |
| **svc-billing** | Stripe integration, usage tracking, per-cluster pricing | Go | P1 |
| **svc-audit** | Immutable audit log of all operations on customer infra | Go | P1 |
| **svc-notification** | Alerts via Slack, email, webhook, PagerDuty | Go | P1 |
| **agent** | In-cluster binary — CNPG controller, PG stats, command executor, log aggregator | Go | P0 |

### Provider/Plugin Architecture

The core engine is **operator-agnostic**. Each data service is a **provider** implementing a standard Go interface. Phase 1 ships the CNPG provider only. Future providers (Strimzi, Redis, ECK) plug in without rewriting core services.

```
Core Engine (operator-agnostic)
  ├── svc-provisioner → calls provider.Provision()
  ├── svc-clusters    → calls provider.Scale(), provider.Status()
  ├── svc-backups     → calls provider.Backup(), provider.Restore()
  ├── svc-monitoring  → calls provider.CollectMetrics()
  └── svc-users       → calls provider.CreateUser()
        │
        ▼
  Provider Interface (Go interface)
        │
  ┌─────┴───────────────────────────────┐
  │ CNPG Provider    (Phase 1 — PG)     │
  │ Strimzi Provider (Phase 3 — Kafka)  │
  │ Redis Provider   (Phase 3 — Redis)  │
  │ ECK Provider     (Phase 3 — ES)     │
  └─────────────────────────────────────┘
```

## 2.5 Data Flow — Provisioning

```
Customer clicks "Create Database"
         │
         ▼
┌─── svc-api ───┐     ┌─── svc-auth ──┐
│ Validate req  │────▶│ Check RBAC    │
└───────┬───────┘     └───────────────┘
        │
        ▼
┌─── svc-provisioner (THE BRAIN) ──────────────────────────────────────┐
│                                                                       │
│  1. svc-infra → AssumeRole → Create node group (kiven-db-nodes)     │
│  2. svc-infra → AssumeRole → Create StorageClass (gp3, encrypted)   │
│  3. svc-infra → AssumeRole → Create S3 bucket (backups)             │
│  4. svc-infra → AssumeRole → Create IRSA role (CNPG → S3)          │
│  5. agent    → Install CNPG operator (Helm)                          │
│  6. agent    → Apply CNPG Cluster YAML (generated by svc-clusters)   │
│  7. agent    → Apply PgBouncer Pooler YAML                           │
│  8. agent    → Apply ScheduledBackup YAML                            │
│  9. agent    → Apply NetworkPolicy YAML                              │
│ 10. agent    → Wait for cluster healthy                              │
│ 11. svc-users → Create initial database + user                       │
│ 12. Return connection string to customer                              │
│                                                                       │
│  Status updates streamed via agent gRPC → svc-agent-relay             │
│  Dashboard shows real-time provisioning progress                      │
└───────────────────────────────────────────────────────────────────────┘
```

## 2.6 Data Flow — Steady State

```
┌─── Kiven Agent (in customer K8s) ─────────────────────────────┐
│                                                                │
│  CNPG Controller ──── watches Cluster/Backup/Pooler CRDs      │
│  PG Stats Collector ─ pg_stat_statements, pg_stat_activity     │
│  Log Aggregator ───── PG logs from all pods                    │
│  Infra Reporter ───── node status, EBS usage, pod health       │
│                                                                │
│  Every 30s: streams metrics + status to svc-agent-relay        │
│  On event: immediately reports (failover, backup done, error)  │
└────────────────────────┬───────────────────────────────────────┘
                         │ gRPC/mTLS (outbound only)
                         ▼
┌─── svc-agent-relay ───────────────────────────────────────────┐
│  Multiplexes connections from all customer agents              │
│  Routes events to: svc-monitoring, svc-clusters, svc-audit    │
└───────────────────────────────────────────────────────────────┘
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
   svc-monitoring   svc-clusters   svc-audit
   (DBA intelligence, (status update) (immutable log)
    alert engine)
```

## 2.7 Service Plans

Each database is provisioned with a **plan** that determines compute, memory, storage, and HA configuration:

| Plan | CPU | RAM | Storage | Instances | HA | Node Type | Use Case |
|------|-----|-----|---------|-----------|-----|-----------|----------|
| **Hobbyist** | 1 vCPU | 1 GB | 10 GB | 1 | No | t3.small | Testing, personal projects |
| **Startup** | 2 vCPU | 4 GB | 50 GB | 2 | Yes | r6g.medium | Small apps, dev/staging |
| **Business** | 4 vCPU | 16 GB | 100 GB | 3 | Yes | r6g.large | Production, medium traffic |
| **Premium** | 8 vCPU | 32 GB | 500 GB | 3 | Yes | r6g.xlarge | High-performance, analytics |
| **Custom** | User-defined | User-defined | User-defined | 1-5 | Configurable | Any | Specific requirements |

Each plan includes:
- Pre-tuned `postgresql.conf` (shared_buffers, work_mem, etc. sized for the plan)
- Appropriate PgBouncer pool size and mode
- Right backup frequency and retention
- Resource limits and requests matching the node type

Plans can be **upgraded or downgraded** at any time from the dashboard (triggers a rolling update via CNPG).

## 2.8 Power Off / Power On

Databases can be **paused** to eliminate compute costs while preserving data. This is a fundamental advantage of the "managed on your infra" model — something Aiven cannot offer because they own the infrastructure.

### Power Off (Pause)

```
Customer clicks "Power Off"
  │
  ├─ 1. svc-clusters → agent: Delete CNPG Cluster CR
  │     PVC reclaim policy = RETAIN → EBS volumes preserved
  │
  ├─ 2. CNPG pods terminated, K8s services removed
  │     EBS volumes detached but retained in AWS
  │
  ├─ 3. svc-infra → AWS API: Scale node group to 0
  │     No more EC2 cost
  │
  └─ 4. Dashboard: "Paused — Data safe, no compute cost"
        S3 backups and EBS volumes remain
```

### Power On (Resume)

```
Customer clicks "Resume"
  │
  ├─ 1. svc-infra → AWS API: Scale node group back up
  │     Wait for nodes ready (~2-3 min)
  │
  ├─ 2. svc-clusters → agent: Apply CNPG Cluster CR
  │     References existing PVCs (same EBS volume IDs)
  │
  ├─ 3. CNPG starts PostgreSQL with existing data
  │     Primary elected, replicas sync (~1-2 min)
  │
  └─ 4. Dashboard: "Running — Resumed"
        Connection strings unchanged, total resume time ~3-5 min
```

### Scheduled Power Off/On

Automate power schedules for non-production environments:
- Example: Mon-Fri 8am-6pm ON, nights and weekends OFF
- Savings: 60-70% on dev/staging compute costs
- Configured via dashboard, API, CLI, or Terraform

### Cost Impact

| Scenario | Always On | Scheduled (10h/day, weekdays) | Savings |
|----------|-----------|-------------------------------|---------|
| Startup plan (2×r6g.medium) | ~$180/mo | ~$55/mo | 70% |
| Business plan (3×r6g.large) | ~$450/mo | ~$140/mo | 69% |
| Paused (storage only) | — | ~$10/mo | 94% |

## 2.9 Two UX Modes

### Simple Mode (Default) — "Aiven Experience"

For developers who just need a database. Forms, sliders, buttons. No YAML visible.
- Create database → pick plan → get connection string
- Manage users, backups, config via UI forms
- See metrics, alerts, logs in clean dashboards

### Advanced Mode — "Lens Experience"

For DevOps/Platform engineers who want full control. Like Lens for Kubernetes.
- View the generated YAML for every resource (CNPG Cluster, Pooler, Backup, etc.)
- Edit YAML directly in Monaco editor (VS Code-like) with CNPG schema validation
- Diff view before applying changes
- Change history (git-like timeline of all YAML changes)
- Rollback to any previous YAML version
- Toggle between modes at any time

---

# PART III — DELIVERY MODEL

## 3.1 Git Strategy

**Trunk-Based Development with Cherry-Pick**

```
                    main (trunk)
                        │
        ┌───────────────┼───────────────┐
        │               │               │
    feature/A       feature/B       feature/C
        │               │               │
        └───────────────┼───────────────┘
                        │
                    merge to main
                        │
            ┌───────────┴───────────┐
            │                       │
            ▼                       ▼
    maintenance/v1.x.x      maintenance/v2.x.x
    (cherry-pick with       (cherry-pick with
     label: backport-v1)     label: backport-v2)
```

| Branch | Usage | Policy |
|--------|-------|--------|
| `main` | Main trunk | All PRs merge here |
| `maintenance/v*.x.x` | Version maintenance | Cherry-pick from main only |
| `feature/*` | Development | Short-lived, merge to main |

## 3.2 GitOps Flow (ArgoCD)

- **Centralized ArgoCD**: Single instance managing all environments
- **App-of-Apps pattern**: ApplicationSets with Git + Matrix generators
- **Auto-sync**: Dev auto-sync, Staging/Prod manual approval

## 3.3 Environments

| Environment | Account | Cluster | Sync Policy |
|-------------|---------|---------|-------------|
| **dev** | kiven-dev | eks-dev | Auto-sync |
| **staging** | kiven-staging | eks-staging | Manual |
| **prod** | kiven-prod | eks-prod | Manual + Approval |

## 3.4 CI/CD & Bootstrap

> Detailed documentation: [bootstrap/BOOTSTRAP-GUIDE.md](bootstrap/BOOTSTRAP-GUIDE.md)

---

# PART IV — REPOSITORY & OWNERSHIP MODEL

## 4.1 Repository Tiers

| Tier | Repos | Description | Owner |
|------|-------|-------------|-------|
| **T0 — Foundation** | `bootstrap/` | AWS Landing Zone, Account Factory | Platform Team |
| **T1 — Platform** | `platform-*` | GitOps, Networking, Security, Observability | Platform Team |
| **T2 — Contracts** | `contracts-proto`, `sdk-*` | gRPC APIs, Go SDK, CLI | Platform + Backend |
| **T3 — Core Services** | `svc-*` | Kiven backend services | Backend Team |
| **T4 — Agent** | `agent/` | Customer-deployed agent | Agent Team |
| **T5 — Frontend** | `dashboard/` | Next.js dashboard (Simple + Advanced modes) | Frontend Team |
| **T6 — Providers** | `provider-*` | CNPG provider, Strimzi provider (future) | Backend Team |
| **T7 — Quality** | `e2e-scenarios`, `chaos-*` | Tests, chaos engineering | QA + Platform |
| **T8 — Documentation** | `docs/` | Centralized documentation | All Teams |

## 4.2 Ownership Matrix

| Tier | Owner Team | Approvers | Change Process |
|------|------------|-----------|----------------|
| **T0 — Foundation** | Platform | Platform Lead + Security | ADR + RFC required |
| **T1 — Platform** | Platform | Platform Team (2 reviewers) | ADR if breaking change |
| **T2 — Contracts** | Platform + Backend | Tech Lead | Buf breaking detection |
| **T3 — Core Services** | Backend | Team Lead | Standard PR review |
| **T4 — Agent** | Agent / Backend | Agent Lead + Security | Security review required |
| **T5 — Frontend** | Frontend | Frontend Lead | Standard PR review |
| **T6 — Providers** | Backend | Tech Lead | Provider interface compliance |
| **T7 — Quality** | QA + Platform | QA Lead | Standard PR review |
| **T8 — Documentation** | All | Tech Lead | Standard PR review |

## 4.3 Repository Index

### Tier 0 — Foundation

| Repo | Description |
|------|-------------|
| `bootstrap/` | AWS Landing Zone, Account Factory, SCPs, SSO |

### Tier 1 — Platform

| Repo | Description |
|------|-------------|
| `platform-gitops/` | ArgoCD, ApplicationSets |
| `platform-networking/` | Cilium, Gateway API |
| `platform-observability/` | OTel, Prometheus, Loki, Tempo, Grafana |
| `platform-security/` | Vault, External-Secrets, Kyverno |

### Tier 2 — Contracts

| Repo | Description |
|------|-------------|
| `contracts-proto/` | Protobuf definitions (agent ↔ SaaS, inter-service) |
| `sdk-go/` | Go SDK for Kiven API |
| `kiven-cli/` | CLI tool (`kiven clusters list`, `kiven backup trigger`) |
| `terraform-provider-kiven/` | Terraform provider for Kiven |

### Tier 3 — Core Services

| Repo | Description |
|------|-------------|
| `svc-api/` | REST + GraphQL gateway |
| `svc-auth/` | Authentication, RBAC, API keys |
| `svc-provisioner/` | Provisioning orchestrator (THE BRAIN) |
| `svc-infra/` | AWS resource management in customer accounts |
| `svc-clusters/` | Cluster lifecycle (CNPG management) |
| `svc-backups/` | Backup/restore, PITR, fork/clone |
| `svc-monitoring/` | Metrics, DBA intelligence, alerts |
| `svc-users/` | Database user/role management |
| `svc-agent-relay/` | gRPC server for agent connections |
| `svc-yamleditor/` | YAML generation, validation, diff, history |
| `svc-migrations/` | Import from Aiven/RDS/bare PG |
| `svc-billing/` | Stripe billing |
| `svc-audit/` | Immutable audit log |
| `svc-notification/` | Alerts (Slack, email, webhook, PagerDuty) |

### Tier 4 — Agent

| Repo | Description |
|------|-------------|
| `kiven-agent/` | In-cluster agent (CNPG controller, PG stats, command executor) |
| `kiven-agent-helm/` | Helm chart for agent deployment |

### Tier 5 — Frontend

| Repo | Description |
|------|-------------|
| `dashboard/` | Next.js dashboard (Simple + Advanced mode) |

### Tier 6 — Providers

| Repo | Description |
|------|-------------|
| `provider-cnpg/` | CloudNativePG provider (Phase 1) |
| `provider-strimzi/` | Strimzi/Kafka provider (Phase 3 — future) |
| `provider-redis/` | Redis Operator provider (Phase 3 — future) |

### Tier 7 — Quality

| Repo | Description |
|------|-------------|
| `e2e-scenarios/` | End-to-end tests (provisioning, backup, failover) |
| `chaos-experiments/` | Chaos Mesh experiments (node failure, network partition) |

---

# PART V — PLATFORM BASELINES

## 5.1 Security Baseline

**Defense in Depth**: 7 layers of security

| Layer | Component | Protection |
|-------|-----------|------------|
| **Edge** | Cloudflare | WAF, DDoS, Bot protection |
| **Gateway** | Cilium Gateway API | TLS termination, routing |
| **Network** | Cilium | NetworkPolicies, default deny |
| **Identity** | IRSA + Vault | Dynamic secrets, mTLS, OIDC |
| **Workload** | Kyverno | Pod security, image signing |
| **Data** | KMS + EBS encryption | Encryption at rest/transit |
| **Customer Access** | Cross-account IAM + Audit | Least privilege, CloudTrail, revocable |

> Detailed documentation: [security/SECURITY-ARCHITECTURE.md](security/SECURITY-ARCHITECTURE.md)

## 5.2 Observability Baseline

| Signal | Tool | Retention | Cost |
|--------|------|-----------|------|
| **Metrics** | Prometheus + Remote Write S3 | 15d local, 1y S3 | ~5 EUR/mo |
| **Logs** | Loki | 30 days (GDPR) | Self-hosted |
| **Traces** | Tempo | 7 days | Self-hosted |
| **Profiling** | Pyroscope | 7 days | Self-hosted |
| **Errors** | Sentry (self-hosted) | 30 days | Self-hosted |

> Detailed documentation: [observability/OBSERVABILITY-GUIDE.md](observability/OBSERVABILITY-GUIDE.md)

## 5.3 Networking Baseline

| Component | Role | Configuration |
|-----------|------|---------------|
| **Cloudflare** | Edge, WAF, Tunnel | Pro tier |
| **Cilium** | CNI, mTLS, Gateway API | WireGuard encryption |
| **VPC Peering** | Aiven connectivity (Kiven product DB) | Private, no internet |
| **Route53** | Private DNS, backup | Internal zones |
| **Cross-Account** | Customer EKS access | IAM AssumeRole, kubeconfig |

> Detailed documentation: [networking/NETWORKING-ARCHITECTURE.md](networking/NETWORKING-ARCHITECTURE.md)

## 5.4 Data Baseline

### Kiven Product Database (SaaS side)

| Service | Provider | Purpose | Cost Estimate |
|---------|----------|---------|---------------|
| **PostgreSQL** | Aiven | Product DB (orgs, clusters, audit) | ~300 EUR/mo |
| **Kafka** | Aiven | Agent events, async operations | ~400 EUR/mo |
| **Valkey** | Aiven | Sessions, rate limiting, cache | ~150 EUR/mo |

### Customer Databases (managed by Kiven)

| Service | Technology | Where | Cost |
|---------|-----------|-------|------|
| **PostgreSQL** | CloudNativePG on EKS | Customer's AWS | Customer's AWS bill |
| **Backups** | Barman → S3 | Customer's AWS | Customer's S3 costs |

**Golden rule**: Kiven product DB and customer databases are **completely separate**. Customer data never touches Kiven's infrastructure.

> Detailed documentation: [data/DATA-ARCHITECTURE.md](data/DATA-ARCHITECTURE.md)

---

# PART VI — TESTING & QUALITY

## 6.1 Test Pyramid

| Layer | Test Types | Frequency |
|-------|-----------|-----------|
| **Base** | Static analysis, linting (golangci-lint) | Pre-commit |
| **Unit** | Service logic, provider interface | PR |
| **Integration** | Agent ↔ CNPG, svc-infra ↔ AWS (LocalStack), DB (Testcontainers) | PR |
| **Contract** | gRPC contracts (Buf), agent protocol | PR |
| **E2E** | Full provisioning pipeline (kind + CNPG) | Nightly |
| **Performance** | Load testing, provisioning time (k6) | Weekly |
| **Chaos** | Node failure, agent disconnection, CNPG failover (Chaos Mesh) | Weekly |

## 6.2 Performance Targets

| Metric | Target | Alert |
|--------|--------|-------|
| **API Latency P50** | < 50ms | > 100ms |
| **API Latency P95** | < 100ms | > 200ms |
| **API Latency P99** | < 200ms | > 500ms |
| **Error Rate** | < 0.1% | > 1% |
| **Provisioning Time** | < 10min | > 15min |
| **Agent Reconnection** | < 30s | > 60s |
| **Backup Success Rate** | > 99.9% | < 99% |

> Detailed documentation: [testing/TESTING-STRATEGY.md](testing/TESTING-STRATEGY.md)

---

# PART VII — RESILIENCE & DR

## 7.1 Failure Modes

### Kiven SaaS Failures

| Failure | Detection | Recovery | RTO |
|---------|-----------|----------|-----|
| Pod crash | Liveness probe | K8s restart | < 30s |
| Node failure | Node NotReady | Pod reschedule | < 2min |
| AZ failure | Multi-AZ detect | Traffic shift | < 5min |
| Product DB failure | Aiven health | Automatic failover | < 5min |
| Kafka broker failure | Aiven health | Automatic rebalance | < 2min |
| Full region failure | Manual | DR procedure | 4h (target) |

### Customer Database Failures (Handled by Kiven)

| Failure | Detection | Recovery | RTO |
|---------|-----------|----------|-----|
| PG pod crash | CNPG + Agent | CNPG automatic restart | < 30s |
| Primary failure | CNPG failover | Automatic promotion of replica | < 30s |
| DB node failure | Agent + AWS | Pod reschedule to healthy node | < 2min |
| EBS volume issue | Agent monitoring | Alert + manual intervention | < 15min |
| Agent disconnection | SaaS heartbeat | Agent auto-reconnects; DB keeps running | Immediate (DB unaffected) |
| Backup failure | Agent monitoring | Retry + alert to customer + Kiven ops | < 1h |
| Data corruption | Backup verification | PITR restore to last good point | < 30min |

## 7.2 Backup Strategy

### Kiven SaaS

| Data | Method | Frequency | Retention |
|------|--------|-----------|-----------|
| Product DB | Aiven automated | Hourly | 7 days |
| Product DB PITR | Aiven WAL | Continuous | 24h |
| Kafka | Topic retention | N/A | 7 days |
| Terraform state | S3 versioning | Every apply | 90 days |

### Customer Databases (Managed by Kiven)

| Data | Method | Frequency | Retention |
|------|--------|-----------|-----------|
| PostgreSQL | Barman (CNPG) → S3 | Configurable (default: 6h) | Configurable (default: 30 days) |
| PostgreSQL PITR | WAL archiving → S3 | Continuous | Configurable (default: 7 days) |
| Backup verification | Automated restore test | Weekly | Report stored 90 days |

> Detailed documentation: [resilience/DR-GUIDE.md](resilience/DR-GUIDE.md)

---

# PART VIII — PLATFORM CONTRACTS

## 8.1 Golden Path (New Kiven Service Checklist)

| Step | Action | Validation |
|------|--------|------------|
| 1 | Create repo from Go service template | Structure compliant |
| 2 | Define protos in contracts-proto | `buf lint` pass |
| 3 | Implement service (Go) | Unit tests > 80% |
| 4 | Configure K8s manifests | Kyverno policies pass |
| 5 | Configure External-Secret | Secrets resolved from Vault |
| 6 | Add ServiceMonitor | Metrics visible in Grafana |
| 7 | Create HTTPRoute or gRPC route | Traffic routable |
| 8 | PR review | Merge → Auto-deploy dev |

## 8.2 SLI/SLO/Error Budgets

| Service | SLI | SLO | Error Budget |
|---------|-----|-----|--------------|
| **svc-api** | Availability | 99.9% | 43 min/month |
| **svc-api** | Latency P99 | < 200ms | N/A |
| **svc-provisioner** | Provisioning success rate | 99.5% | N/A |
| **svc-agent-relay** | Agent connection uptime | 99.9% | 43 min/month |
| **Agent** | Metrics delivery | 99.9% | 43 min/month |
| **Customer DB** | Backup success rate | 99.9% | N/A |
| **Platform** | Availability | 99.5% | 3.6h/month |

## 8.3 On-Call Structure

| Role | Responsibility | Rotation |
|------|---------------|----------|
| **Primary** | First responder, triage (SaaS + customer infra) | Weekly |
| **Secondary** | Escalation, deep expertise | Weekly |
| **Incident Commander** | Coordination for P1 (customer data at risk) | On-demand |

> Detailed documentation: [platform/PLATFORM-ENGINEERING.md](platform/PLATFORM-ENGINEERING.md)

---

# PART IX — ROADMAP

## 9.1 Build Sequence

| Phase | Focus | Duration |
|-------|-------|----------|
| **1** | Bootstrap Layer 0-1 (IAM, VPC, EKS) | 3 weeks |
| **2** | Platform GitOps (ArgoCD) | 1 week |
| **3** | Platform Networking (Cilium, Gateway API) + Cloudflare | 2 weeks |
| **4** | Platform Security (Vault, Kyverno) | 2 weeks |
| **5** | Platform Observability (Prometheus, Loki, Tempo) | 2 weeks |
| **6** | Agent framework + gRPC protocol + agent-relay | 3 weeks |
| **7** | CNPG Provider (provider-cnpg) | 2 weeks |
| **8** | svc-provisioner (THE BRAIN) + svc-infra (AWS resources) | 4 weeks |
| **9** | svc-clusters + svc-backups + svc-users | 3 weeks |
| **10** | svc-monitoring + DBA intelligence (basic) | 3 weeks |
| **11** | Dashboard — Simple Mode (Next.js) | 4 weeks |
| **12** | Dashboard — Advanced Mode (YAML editor) | 2 weeks |
| **13** | svc-auth (OIDC, RBAC, org model) | 2 weeks |
| **14** | CLI + API + Terraform Provider | 3 weeks |
| **15** | svc-billing (Stripe) + svc-audit | 2 weeks |
| **16** | svc-migrations (Aiven/RDS import) | 2 weeks |
| **17** | Testing (E2E, chaos, performance) | 2 weeks |
| **18** | Compliance audit (GDPR, SOC2) | 2 weeks |

**Total estimated: ~43 weeks (~10 months)**

## 9.2 Pre-Start Checklist

### Accounts & Access
- [ ] AWS account created, billing configured
- [ ] Aiven account created (product database)
- [ ] Cloudflare account created
- [ ] GitHub organization created
- [ ] Stripe account created (billing)
- [ ] DNS domain acquired (kiven.io or similar)

### Decisions Validated
- [ ] RPO 1h / RTO 15min (SaaS)
- [ ] AWS eu-west-1
- [ ] Go as backend language
- [ ] Next.js as frontend
- [ ] CNPG as PostgreSQL engine
- [ ] Agent-based connectivity (gRPC/mTLS)
- [ ] Cross-account IAM for customer infra access
- [ ] Provider/plugin architecture for multi-operator future
- [ ] Aiven for Kiven product DB + Kafka
- [ ] ArgoCD centralized
- [ ] Cilium + Gateway API
- [ ] Kyverno
- [ ] HashiCorp Vault self-hosted

---

# APPENDIX

## A. Glossary

> [GLOSSARY.md](GLOSSARY.md)

## B. ADR Index

| ADR | Title | Status |
|-----|-------|--------|
| 001 | Landing Zone: Control Tower + Terraform | Accepted |
| 002 | CNPG as PostgreSQL Engine | Accepted |
| 003 | Agent-Based Connectivity | Accepted |
| 004 | Provider/Plugin Architecture | Accepted |
| ... | ... | ... |

> [adr/](adr/)

## C. Change Management Process

### Architecture Changes
1. **ADR Required**: Any decision impacting >1 service
2. **Review**: Platform Team + Tech Lead
3. **Communication**: Slack #platform-updates

### Breaking Changes
1. RFC required (`docs/rfc/`)
2. Migration path documented
3. Announce 2 sprints before

### Emergency Changes
1. Incident Commander approval
2. Post-mortem required
3. Retroactive ADR within 48h

---

# Documentation Index

| Document | Description | Path |
|----------|-------------|------|
| **Bootstrap Guide** | AWS setup, Account Factory | [bootstrap/BOOTSTRAP-GUIDE.md](bootstrap/BOOTSTRAP-GUIDE.md) |
| **Security Architecture** | Defense in depth, IAM, cross-account, Vault | [security/SECURITY-ARCHITECTURE.md](security/SECURITY-ARCHITECTURE.md) |
| **Observability Guide** | Metrics, logs, traces, APM, dashboards | [observability/OBSERVABILITY-GUIDE.md](observability/OBSERVABILITY-GUIDE.md) |
| **Networking Architecture** | VPC, Cloudflare, Gateway API, customer connectivity | [networking/NETWORKING-ARCHITECTURE.md](networking/NETWORKING-ARCHITECTURE.md) |
| **Data Architecture** | Product DB, Kafka, customer DB model | [data/DATA-ARCHITECTURE.md](data/DATA-ARCHITECTURE.md) |
| **Testing Strategy** | Pyramid, E2E, chaos, provisioning tests | [testing/TESTING-STRATEGY.md](testing/TESTING-STRATEGY.md) |
| **Platform Engineering** | Contracts, Golden Path, on-call, CI/CD | [platform/PLATFORM-ENGINEERING.md](platform/PLATFORM-ENGINEERING.md) |
| **DR Guide** | Backup, recovery, SaaS DR + customer DB DR | [resilience/DR-GUIDE.md](resilience/DR-GUIDE.md) |
| **Agent Architecture** | Agent design, gRPC protocol, deployment | [agent/AGENT-ARCHITECTURE.md](agent/AGENT-ARCHITECTURE.md) |
| **Customer Infra Management** | Nodes, storage, S3, IAM, cross-account | [infra/CUSTOMER-INFRA-MANAGEMENT.md](infra/CUSTOMER-INFRA-MANAGEMENT.md) |
| **Customer Onboarding** | CloudFormation, EKS discovery, provisioning | [onboarding/CUSTOMER-ONBOARDING.md](onboarding/CUSTOMER-ONBOARDING.md) |
| **Provider Interface** | Plugin architecture, Go interface, adding providers | [providers/PROVIDER-INTERFACE.md](providers/PROVIDER-INTERFACE.md) |
| **Glossary** | All terminology | [GLOSSARY.md](GLOSSARY.md) |

---

*Maintained by: Kiven Platform Team*
*Last updated: February 2026*
