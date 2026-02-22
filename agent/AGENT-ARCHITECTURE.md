# Kiven Agent Architecture
## *The Bridge Between Kiven SaaS and Customer Kubernetes*

---

> **Back to**: [Architecture Overview](../EntrepriseArchitecture.md)

---

# What Is the Kiven Agent

The agent is a **single Go binary** deployed inside the customer's Kubernetes cluster. It is the only component Kiven runs in the customer's environment. Everything Kiven does on the customer's cluster goes through the agent.

```
Kiven SaaS (our infra)  ◄──── gRPC/mTLS (outbound from agent) ────  Agent (customer's K8s)
                                                                         │
                                                                         ├── Watches CNPG CRDs
                                                                         ├── Collects PG metrics
                                                                         ├── Executes commands
                                                                         ├── Aggregates logs
                                                                         └── Reports infra status
```

---

# Design Principles

| Principle | Implementation |
|-----------|---------------|
| **Outbound-only** | Agent initiates connection to Kiven SaaS. No inbound ports on customer's firewall. |
| **Minimal footprint** | < 50MB RAM, < 0.1 CPU. Must not impact customer's workloads. |
| **Fault-tolerant** | If agent loses connection, databases keep running. Agent auto-reconnects. |
| **Secure** | mTLS for all communication. ServiceAccount scoped to CNPG CRDs only. |
| **Single binary** | One Go binary, deployed via Helm chart. No dependencies. |
| **Multi-provider ready** | Plugin system: auto-detects installed operators, activates relevant modules. |

---

# Agent Components

```
┌─────────────────────────────────────────────────────────────────────┐
│                    KIVEN AGENT (Go binary)                           │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  Provider Registry                                          │   │
│  │  ├── CNPG Module (Phase 1)                                  │   │
│  │  │   ├── CNPG Watcher (informers on Cluster/Backup/Pooler) │   │
│  │  │   ├── PG Stats Collector (pg_stat_*, via PG connection)  │   │
│  │  │   └── PG Log Collector (pod logs from CNPG pods)         │   │
│  │  ├── Strimzi Module (Future)                                │   │
│  │  └── Redis Module (Future)                                  │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  Core Components                                            │   │
│  │  ├── Command Executor    — applies YAML, runs SQL           │   │
│  │  ├── Infra Reporter      — node status, EBS, resource usage │   │
│  │  ├── Health Monitor      — self-health, connectivity check  │   │
│  │  └── Config Manager      — agent config, hot reload         │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  Transport Layer                                            │   │
│  │  ├── gRPC Client (mTLS, outbound to svc-agent-relay)       │   │
│  │  ├── Event Buffer (in-memory, survives brief disconnects)   │   │
│  │  └── Heartbeat (every 30s to prove agent is alive)          │   │
│  └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

## CNPG Watcher

Uses Kubernetes **informers** (via controller-runtime) to watch CNPG CRDs:
- `Cluster` — status changes, failover events, replication lag
- `Backup` — backup start/complete/fail events
- `ScheduledBackup` — schedule status
- `Pooler` — PgBouncer status, connection stats

On any change → event streamed to Kiven SaaS via gRPC.

## PG Stats Collector

Connects to PostgreSQL directly (using credentials from CNPG-managed K8s Secret):
- `pg_stat_statements` — query performance (every 60s)
- `pg_stat_activity` — active queries, blocking (every 30s)
- `pg_stat_bgwriter` — checkpoint/write stats (every 60s)
- `pg_stat_user_tables` — table stats, dead tuples (every 300s)
- Custom queries for bloat detection, XID age (every 300s)

**Important**: Query parameter values are **never collected**. Only query templates (`SELECT * FROM users WHERE id = $1`).

## PG Log Collector

Tails PostgreSQL pod logs via Kubernetes API:
- Filters for ERROR, WARNING, FATAL, PANIC levels
- Applies **log scrubbing**: replaces parameter values with `$N`
- Batches and streams to Kiven SaaS
- Detects patterns: slow queries, connection rejections, OOM

## Command Executor

Receives commands from Kiven SaaS (via gRPC stream) and executes them:

| Command Type | What It Does | Example |
|-------------|-------------|---------|
| `apply_yaml` | Applies K8s manifest | Create/update CNPG Cluster, Pooler, Backup |
| `delete_resource` | Deletes K8s resource | Delete cluster on power-off (PVCs retained) |
| `run_sql` | Executes SQL via PG connection | CREATE USER, GRANT, ALTER SYSTEM |
| `install_helm` | Installs/upgrades Helm chart | Install CNPG operator |
| `collect_diagnostics` | Runs diagnostic checks | Prerequisites validation |

Every command is:
- **Logged** with full audit trail (who requested, what was executed, result)
- **Idempotent** where possible (apply is naturally idempotent)
- **Validated** before execution (schema validation for YAML)
- **Reported** with result (success/failure + output)

## Infra Reporter

Reports infrastructure-level information:
- Node status (Ready/NotReady, capacity, allocatable)
- EBS volume usage (via PVC status + df)
- Resource consumption (CPU/memory per CNPG pod)
- Kubernetes version, CNPG operator version
- Storage classes available
- Namespace resource quotas

---

# Communication Protocol

## gRPC Service Definition (Simplified)

```protobuf
service AgentRelay {
  // Agent → SaaS: bidirectional stream for status and metrics
  rpc Connect(stream AgentMessage) returns (stream ServerMessage);

  // Agent → SaaS: initial registration
  rpc Register(RegisterRequest) returns (RegisterResponse);
}

message AgentMessage {
  oneof payload {
    Heartbeat heartbeat = 1;
    ClusterStatus cluster_status = 2;
    MetricsBatch metrics = 3;
    LogBatch logs = 4;
    EventReport event = 5;
    CommandResult command_result = 6;
    InfraReport infra_report = 7;
  }
}

message ServerMessage {
  oneof payload {
    Command command = 1;
    ConfigUpdate config_update = 2;
    Ack ack = 3;
  }
}
```

## Connection Lifecycle

```
Agent starts
  │
  ├── 1. Load mTLS certificates (from K8s Secret)
  ├── 2. Connect to svc-agent-relay (gRPC/mTLS)
  ├── 3. Register: send agent ID, cluster info, CNPG version
  ├── 4. Start bidirectional stream (Connect RPC)
  │
  │   ┌── Agent → SaaS ──────────────────────────────────┐
  │   │ Heartbeat every 30s                               │
  │   │ Cluster status on change (informer events)         │
  │   │ Metrics every 30-60s                               │
  │   │ Logs (filtered, scrubbed) on arrival               │
  │   │ Command results after execution                    │
  │   └───────────────────────────────────────────────────┘
  │
  │   ┌── SaaS → Agent ──────────────────────────────────┐
  │   │ Commands (apply_yaml, run_sql, etc.)              │
  │   │ Config updates (collection intervals, log level)   │
  │   │ Acknowledgements                                   │
  │   └───────────────────────────────────────────────────┘
  │
  └── On disconnect: buffer events, retry with exponential backoff
      Databases continue running. No data loss.
```

---

# Deployment

## Helm Chart

```bash
helm install kiven-agent kiven/agent \
  --namespace kiven-system \
  --create-namespace \
  --set agentToken=<token-from-kiven-dashboard> \
  --set relay.endpoint=agent-relay.kiven.io:443
```

## Kubernetes Resources Created

| Resource | Namespace | Purpose |
|----------|-----------|---------|
| Deployment (1 replica) | kiven-system | The agent pod |
| ServiceAccount | kiven-system | Identity for RBAC |
| ClusterRole | — | Read CNPG CRDs, read pods/logs, manage kiven-databases namespace |
| ClusterRoleBinding | — | Binds role to ServiceAccount |
| Secret | kiven-system | mTLS certificates + agent token |
| ConfigMap | kiven-system | Agent configuration (intervals, log level) |

## RBAC (Least Privilege)

```yaml
rules:
  # CNPG CRDs — full access (for provisioning)
  - apiGroups: ["postgresql.cnpg.io"]
    resources: ["clusters", "backups", "scheduledbackups", "poolers"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]

  # Pods/logs — read only (for metrics and log collection)
  - apiGroups: [""]
    resources: ["pods", "pods/log", "services", "secrets", "configmaps", "persistentvolumeclaims"]
    verbs: ["get", "list", "watch"]

  # Namespaces — manage kiven-databases
  - apiGroups: [""]
    resources: ["namespaces"]
    verbs: ["get", "list", "watch", "create"]

  # Network policies — create in kiven-databases
  - apiGroups: ["networking.k8s.io"]
    resources: ["networkpolicies"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]

  # Storage classes — read (for prerequisites check)
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list"]

  # Nodes — read (for infra reporting)
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "list"]
```

---

# Failure Modes

| Failure | Impact | Recovery |
|---------|--------|----------|
| **Agent pod crash** | Kiven dashboard shows "agent offline". Databases keep running. | K8s restarts pod automatically. Agent reconnects. |
| **gRPC connection lost** | Events buffered in memory. Dashboard shows stale data (with warning). | Agent retries with exponential backoff (1s, 2s, 4s, 8s... max 60s). |
| **Agent misconfigured** | Agent can't connect or authenticate. | Dashboard shows "agent not connected". Customer re-runs Helm install. |
| **CNPG operator not installed** | Agent reports "CNPG not found" during prerequisites check. | svc-provisioner installs CNPG operator via agent (install_helm command). |
| **Insufficient RBAC** | Agent commands fail with 403. | Agent reports permission error. Customer adjusts ClusterRoleBinding. |

**Key invariant**: Agent failure NEVER affects running databases. CNPG operator manages PG independently. Agent is only for Kiven management plane.

---

# Metrics Collected

| Category | Metrics | Interval |
|----------|---------|----------|
| **PostgreSQL** | connections, QPS, transactions, replication lag, cache hit ratio | 30s |
| **Queries** | top queries by time/calls, slow queries (> threshold), lock waits | 60s |
| **Tables** | size, dead tuples, seq scans, idx scans, bloat estimate | 300s |
| **System** | CPU, memory, disk usage (per PG pod) | 30s |
| **CNPG** | cluster phase, timeline, instances ready, failover count | On change |
| **Backups** | last backup time, duration, size, WAL archiving lag | On change |
| **PgBouncer** | active/idle/waiting connections, pool utilization | 30s |
| **Infrastructure** | node status, EBS IOPS, storage capacity | 60s |

---

*Maintained by: Agent Team*
*Last updated: February 2026*
