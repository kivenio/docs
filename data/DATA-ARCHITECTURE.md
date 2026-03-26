# Data Architecture
## *Kiven — Product Database, Kafka, Cache & Customer Database Model*

---

> **Back to**: [Architecture Overview](../EntrepriseArchitecture.md)

---

# Table of Contents

1. [Two Data Domains](#two-data-domains)
2. [Kiven Product Database (SaaS)](#kiven-product-database-saas)
3. [Customer Databases (Managed by Kiven)](#customer-databases-managed-by-kiven)
4. [Kafka Topics](#kafka-topics)
5. [Cache Architecture (Valkey)](#cache-architecture-valkey)
6. [Data Isolation Principle](#data-isolation-principle)

---

# Two Data Domains

Kiven has **two completely separate data domains** that must never mix:

```
┌─────────────────────────────────────┐  ┌─────────────────────────────────────┐
│  DOMAIN 1: Kiven Product Data       │  │  DOMAIN 2: Customer Database Data   │
│  (lives in Kiven's AWS account)     │  │  (lives in customer's AWS account)  │
│                                     │  │                                     │
│  PostgreSQL (Aiven) — product DB    │  │  PostgreSQL (CNPG on customer EKS)  │
│  Kafka (Aiven) — events             │  │  Barman backups → customer's S3     │
│  Valkey (Aiven) — cache             │  │                                     │
│                                     │  │  Kiven NEVER accesses row data.     │
│  Contains: orgs, users, clusters,   │  │  Agent collects only: pg_stat_*,   │
│  billing, audit, agent metadata     │  │  logs, CRD status, metrics.         │
└─────────────────────────────────────┘  └─────────────────────────────────────┘
```

**Golden rule: Customer data never touches Kiven's infrastructure.**

---

# Kiven Product Database (SaaS)

## Aiven Configuration

| Service | Plan | Config | Estimated Cost |
|---------|------|--------|----------------|
| **PostgreSQL** | Business-4 | Primary + Read Replica, 100GB | ~300 EUR/mo |
| **Kafka** | Business-4 | 3 brokers, 100GB retention | ~400 EUR/mo |
| **Valkey** | Business-4 | 2 nodes, 10GB, HA | ~150 EUR/mo |

**Total estimated Aiven cost: ~850 EUR/mo**

## Database Configuration

| Aspect | Choice | Rationale |
|--------|--------|-----------|
| **Replication** | Aiven managed (async) | RPO 1h acceptable for product DB |
| **Backup** | Aiven automated hourly | RPO 1h |
| **Failover** | Aiven automated | RTO < 15min |
| **Connection** | VPC Peering (private) | No public internet |
| **Pooling** | PgBouncer (Aiven built-in) | Connection efficiency |

## Schema Ownership

### Core Tables

| Table | Owner Service | Description |
|-------|---------------|-------------|
| `organizations` | svc-auth | Customer organizations |
| `users` | svc-auth | Dashboard users, roles, teams |
| `api_keys` | svc-auth | API key management |
| `clusters` | svc-clusters | Managed CNPG cluster metadata |
| `cluster_configs` | svc-yamleditor | YAML history, versions, diffs |
| `databases` | svc-users | PostgreSQL databases within clusters |
| `database_users` | svc-users | PostgreSQL roles within clusters |
| `backups` | svc-backups | Backup records, status, PITR points |
| `backup_verifications` | svc-backups | Restore test results |
| `agents` | svc-agent-relay | Registered agents, heartbeat status |
| `provisioning_jobs` | svc-provisioner | Provisioning pipeline state machine |
| `infra_resources` | svc-infra | Customer AWS resources (node groups, EBS, S3, IAM) |
| `service_plans` | svc-clusters | Plan definitions (Hobbyist, Startup, Business...) |
| `metrics_snapshots` | svc-monitoring | Aggregated metrics for dashboard display |
| `alerts` | svc-monitoring | Alert rules and status |
| `dba_recommendations` | svc-monitoring | Performance advisor suggestions |
| `audit_log` | svc-audit | Immutable audit trail |
| `billing_subscriptions` | svc-billing | Stripe subscriptions, usage |
| `invoices` | svc-billing | Invoice records |
| `migrations` | svc-migrations | Migration jobs (from Aiven/RDS) |

### Power Schedule Tables

| Table | Owner Service | Description |
|-------|---------------|-------------|
| `power_schedules` | svc-clusters | Scheduled power on/off rules |
| `power_events` | svc-clusters | Power on/off event history |

**Rule: 1 table = 1 owner. Cross-service communication = gRPC or Kafka events, never JOINs.**

## Connection Best Practices

| Parameter | Recommended Value | Rationale |
|-----------|-------------------|-----------|
| **pool_size** | 20 | Connections per service pod |
| **max_overflow** | 10 | Extra connections at peak |
| **pool_timeout** | 30s | Max wait for connection |
| **pool_recycle** | 1800s | Recycle connections every 30min |
| **ssl** | require | Always encrypted |

---

# Customer Databases (Managed by Kiven)

## What Kiven Provisions

For each customer database, Kiven creates:

| Resource | Type | Where | Managed By |
|----------|------|-------|------------|
| CNPG Cluster CR | Kubernetes CRD | Customer K8s | Kiven agent |
| PostgreSQL pods | Pods (Primary + Replicas) | Customer K8s | CNPG operator |
| PgBouncer Pooler | Kubernetes CRD | Customer K8s | CNPG operator |
| EBS volumes | AWS EBS gp3 | Customer AWS | Kiven svc-infra |
| S3 backup bucket | AWS S3 | Customer AWS | Kiven svc-infra |
| IRSA role | AWS IAM | Customer AWS | Kiven svc-infra |
| ScheduledBackup CR | Kubernetes CRD | Customer K8s | Kiven agent |
| NetworkPolicy | Kubernetes | Customer K8s | Kiven agent |

## Service Plan → Infrastructure Mapping

| Plan | Node Type | Instances | Storage | Backup Freq | PgBouncer Pool |
|------|-----------|-----------|---------|-------------|----------------|
| **Hobbyist** | t3.small | 1 | 10GB gp3 | Daily | 25 |
| **Startup** | r6g.medium | 2 | 50GB gp3 | 6h | 50 |
| **Business** | r6g.large | 3 | 100GB gp3 (3000 IOPS) | 1h | 100 |
| **Premium** | r6g.xlarge | 3 | 500GB gp3 (6000 IOPS) | 30min | 200 |
| **Custom** | Any | 1-5 | Custom | Custom | Custom |

## Auto-Tuned postgresql.conf per Plan

| Parameter | Hobbyist | Startup | Business | Premium |
|-----------|----------|---------|----------|---------|
| `shared_buffers` | 256MB | 1GB | 4GB | 8GB |
| `effective_cache_size` | 768MB | 3GB | 12GB | 24GB |
| `work_mem` | 4MB | 16MB | 32MB | 64MB |
| `maintenance_work_mem` | 64MB | 256MB | 512MB | 1GB |
| `max_connections` | 50 | 100 | 200 | 400 |
| `wal_buffers` | 8MB | 16MB | 32MB | 64MB |
| `random_page_cost` | 1.1 | 1.1 | 1.1 | 1.1 |
| `effective_io_concurrency` | 200 | 200 | 200 | 200 |
| `checkpoint_completion_target` | 0.9 | 0.9 | 0.9 | 0.9 |

These values are the **defaults per plan**. The DBA intelligence engine adjusts them based on real workload over time.

## What Kiven Collects (Metadata Only — Never Row Data)

| Data Collected | Source | Purpose | Contains PII? |
|----------------|--------|---------|---------------|
| `pg_stat_statements` | PG catalog | Query performance analysis | No (queries anonymized) |
| `pg_stat_activity` | PG catalog | Active connections, blocking | No |
| `pg_stat_bgwriter` | PG catalog | Checkpoint/write performance | No |
| `pg_stat_user_tables` | PG catalog | Table size, seq/idx scans | No |
| CNPG Cluster status | K8s CRD | Cluster health, replication lag | No |
| Pod metrics | Kubelet | CPU, memory, disk usage | No |
| PG logs | Pod logs | Error detection, slow queries | Potentially (log scrubbing applied) |
| Node status | K8s API | Node health, capacity | No |
| EBS metrics | CloudWatch | Disk IOPS, latency | No |

**Log scrubbing**: The agent strips potential PII from PG logs before sending to Kiven (query parameter values replaced with `$N`).

---

# Kafka Topics

## Topic Configuration

| Topic | Producer | Consumers | Retention | Purpose |
|-------|----------|-----------|-----------|---------|
| `agent.status.v1` | Agent (via relay) | svc-clusters, svc-monitoring | 7 days | Cluster status updates |
| `agent.metrics.v1` | Agent (via relay) | svc-monitoring | 3 days | PG metrics stream |
| `agent.logs.v1` | Agent (via relay) | svc-monitoring | 3 days | PG log stream |
| `agent.events.v1` | Agent (via relay) | svc-clusters, svc-notification | 7 days | Failover, backup, error events |
| `provisioning.commands.v1` | svc-provisioner | Agent (via relay) | 1 day | Commands to execute in customer K8s |
| `provisioning.status.v1` | svc-provisioner | svc-api, dashboard | 7 days | Provisioning pipeline progress |
| `audit.actions.v1` | All services | svc-audit | 30 days | Immutable audit trail |
| `billing.usage.v1` | svc-monitoring | svc-billing | 30 days | Per-cluster usage metrics |
| `alerts.triggered.v1` | svc-monitoring | svc-notification | 7 days | Alert events for dispatch |
| `dba.recommendations.v1` | svc-monitoring | svc-api, dashboard | 7 days | DBA intelligence suggestions |

## Topic Naming Convention

```
{domain}.{entity}.{version}

Examples:
  agent.status.v1
  provisioning.commands.v1
  audit.actions.v1
```

## Kafka Monitoring

| Metric | Alert Threshold | Severity |
|--------|----------------|----------|
| **Consumer Lag** | > 1000 messages | P2 |
| **Under-replicated Partitions** | > 0 | P1 |
| **Active Controller Count** | != 1 | P1 |
| **Offline Partitions** | > 0 | P1 |
| **Request Latency P99** | > 100ms | P2 |

---

# Cache Architecture (Valkey)

## Cache Stack

| Component | Tool | Hosting | Estimated Cost |
|-----------|------|---------|----------------|
| **Distributed cache** | Valkey (Redis-compatible) | Aiven | ~150 EUR/mo |
| **Local cache (L1)** | Go `bigcache` | In-memory per pod | 0 EUR |

## Cache Use Cases

| Use Case | Strategy | TTL | Invalidation |
|----------|----------|-----|--------------|
| **Session data** | Write-through | 24h | Explicit logout |
| **Cluster status** | Cache-aside | 30s | Agent event |
| **Org/team config** | Read-through | 5min | TTL + manual |
| **Rate limiting** | Write-through | Sliding window | Auto-expire |
| **API response cache** | Cache-aside | 1min | TTL |
| **Agent connection state** | Write-through | Heartbeat interval | Agent disconnect |
| **Service plan definitions** | Read-through | 1h | Manual invalidation |

## Cache Key Naming Convention

```
{service}:{entity}:{id}:{version}

Examples:
  auth:session:sess_abc123
  clusters:status:cluster_456:v1
  monitoring:metrics:cluster_456:latest
  ratelimit:api:org_789:minute
  plans:definition:business:v1
```

## Cache Metrics

| Metric | Alert Threshold | Action |
|--------|----------------|--------|
| **Hit Rate** | < 80% | Review TTL, preloading |
| **Latency P99** | > 10ms | Check network, cluster size |
| **Memory Usage** | > 80% | Eviction analysis, scale up |
| **Connection Errors** | > 0 | Check connectivity |

---

# Data Isolation Principle

```
┌──────────────────────────────────────────────────────────────────────────┐
│                      DATA ISOLATION MODEL                                 │
│                                                                           │
│  ┌─── Kiven SaaS ───────────────────────────────────────────────────┐   │
│  │                                                                   │   │
│  │  Product DB (Aiven PG)    Kafka (Aiven)    Valkey (Aiven)        │   │
│  │  ├─ organizations         ├─ agent events   ├─ sessions          │   │
│  │  ├─ clusters (metadata)   ├─ audit trail    ├─ rate limits       │   │
│  │  ├─ audit_log             ├─ alerts         ├─ cache             │   │
│  │  └─ billing               └─ billing usage  └─ agent state       │   │
│  │                                                                   │   │
│  │  CONTAINS: Metadata, config, status, metrics aggregates           │   │
│  │  NEVER CONTAINS: Customer's actual database rows                  │   │
│  └───────────────────────────────────────────────────────────────────┘   │
│                                                                           │
│  ┌─── Customer A's AWS ──────┐  ┌─── Customer B's AWS ──────┐          │
│  │                            │  │                            │          │
│  │  CNPG PostgreSQL           │  │  CNPG PostgreSQL           │          │
│  │  ├─ Their app data         │  │  ├─ Their app data         │          │
│  │  └─ Their users            │  │  └─ Their users            │          │
│  │                            │  │                            │          │
│  │  S3: Their backups         │  │  S3: Their backups         │          │
│  │  EBS: Their volumes        │  │  EBS: Their volumes        │          │
│  │                            │  │                            │          │
│  │  KIVEN NEVER READS THIS    │  │  KIVEN NEVER READS THIS    │          │
│  └────────────────────────────┘  └────────────────────────────┘          │
│                                                                           │
└──────────────────────────────────────────────────────────────────────────┘
```

This isolation is **fundamental to Kiven's value proposition**: the customer's data never leaves their infrastructure. Kiven only manages the infrastructure and configuration around it.

---

*Maintained by: Platform Team + Backend Team*
*Last updated: February 2026*
