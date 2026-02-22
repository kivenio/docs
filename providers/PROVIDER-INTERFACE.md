# Provider Interface
## *Plugin Architecture for Multi-Operator Support*

---

> **Back to**: [Architecture Overview](../EntrepriseArchitecture.md)

---

# Why a Provider System

Kiven starts with PostgreSQL (CNPG) but will expand to Kafka (Strimzi), Redis, Elasticsearch, and more. Instead of hardcoding CNPG throughout the codebase, we define a **Provider Interface** — a Go interface that every data service must implement.

```
Core Engine (operator-agnostic)
  │
  │  Calls provider.Provision(), provider.Scale(), etc.
  │  Doesn't know or care which operator is underneath.
  │
  ▼
Provider Interface (Go interface)
  │
  ├── CNPG Provider    (Phase 1) ← implements interface for PostgreSQL
  ├── Strimzi Provider (Phase 3) ← implements interface for Kafka
  ├── Redis Provider   (Phase 3) ← implements interface for Redis
  └── ECK Provider     (Phase 3) ← implements interface for Elasticsearch
```

---

# The Interface

```go
package provider

import "context"

// Provider is the interface every data service must implement.
// Core services (svc-provisioner, svc-clusters, etc.) call these methods
// without knowing which operator is underneath.
type Provider interface {
    // Metadata
    Name() string                    // "cnpg", "strimzi", "redis"
    DisplayName() string             // "PostgreSQL", "Kafka", "Redis"
    Version() string                 // Provider version ("1.0.0")
    SupportedVersions() []string     // Data service versions ("15", "16", "17")

    // Discovery (used by agent)
    Detect(ctx context.Context) (*DetectResult, error)
    // Returns: operator installed? version? CRDs registered?

    // Prerequisites
    CheckPrerequisites(ctx context.Context, plan ServicePlan) (*PrereqReport, error)
    // Returns: what's ready, what's missing, what needs fixing

    // Lifecycle
    Provision(ctx context.Context, spec ClusterSpec) (*ClusterStatus, error)
    Scale(ctx context.Context, id string, spec ScaleSpec) error
    Upgrade(ctx context.Context, id string, targetVersion string) error
    Delete(ctx context.Context, id string, retainVolumes bool) error
    PowerOff(ctx context.Context, id string) error
    PowerOn(ctx context.Context, id string) error

    // Status
    GetStatus(ctx context.Context, id string) (*ClusterStatus, error)
    ListClusters(ctx context.Context) ([]ClusterSummary, error)

    // YAML Generation (for Advanced Mode)
    GenerateYAML(ctx context.Context, spec ClusterSpec) ([]YAMLResource, error)
    ValidateYAML(ctx context.Context, yaml string) (*ValidationResult, error)
    DiffYAML(ctx context.Context, id string, newYAML string) (*DiffResult, error)

    // Users & Access
    ListUsers(ctx context.Context, id string) ([]DatabaseUser, error)
    CreateUser(ctx context.Context, id string, spec UserSpec) (*DatabaseUser, error)
    DeleteUser(ctx context.Context, id string, username string) error
    UpdatePermissions(ctx context.Context, id string, username string, perms Permissions) error

    // Databases
    ListDatabases(ctx context.Context, id string) ([]Database, error)
    CreateDatabase(ctx context.Context, id string, spec DatabaseSpec) (*Database, error)
    DeleteDatabase(ctx context.Context, id string, dbName string) error

    // Backups
    ListBackups(ctx context.Context, id string) ([]Backup, error)
    TriggerBackup(ctx context.Context, id string) (*Backup, error)
    Restore(ctx context.Context, id string, target RestoreTarget) error
    VerifyBackup(ctx context.Context, id string, backupID string) (*VerificationResult, error)

    // Metrics
    CollectMetrics(ctx context.Context, id string) (*MetricsSnapshot, error)
    GetConnectionInfo(ctx context.Context, id string) (*ConnectionInfo, error)

    // Configuration
    GetConfig(ctx context.Context, id string) (*ServiceConfig, error)
    UpdateConfig(ctx context.Context, id string, params map[string]string) error

    // Extensions / Plugins (service-specific)
    ListExtensions(ctx context.Context, id string) ([]Extension, error)
    EnableExtension(ctx context.Context, id string, extName string) error
    DisableExtension(ctx context.Context, id string, extName string) error
}
```

---

# Key Types

```go
// ClusterSpec defines what to provision
type ClusterSpec struct {
    Name            string
    ServiceVersion  string            // "17" for PG 17
    Plan            ServicePlan       // Hobbyist, Startup, etc.
    Instances       int               // Number of instances (1-5)
    StorageSize     string            // "50Gi"
    StorageIOPS     int               // 3000
    BackupSchedule  string            // "0 */6 * * *"
    BackupRetention int               // days
    Parameters      map[string]string // postgresql.conf overrides
    Extensions      []string          // pg_vector, PostGIS...
    PoolerEnabled   bool
    PoolerMode      string            // "transaction"
    PoolerPoolSize  int               // 100
    TLSEnabled      bool
    Namespace       string
    Labels          map[string]string
}

// ClusterStatus is the current state
type ClusterStatus struct {
    ID              string
    Name            string
    Phase           string    // "Healthy", "Provisioning", "Failing", "PoweredOff"
    Instances       int
    ReadyInstances  int
    PrimaryPod      string
    ReplicaPods     []string
    ReplicationLag  []ReplicaLag
    StorageUsed     string
    StorageTotal    string
    ServiceVersion  string
    CreatedAt       time.Time
    ConnectionInfo  ConnectionInfo
}

// ConnectionInfo for the customer
type ConnectionInfo struct {
    Host         string  // pg-main-rw.kiven-databases.svc
    Port         int     // 5432
    ReadOnlyHost string  // pg-main-ro.kiven-databases.svc
    PoolerHost   string  // pg-main-pooler.kiven-databases.svc
    Database     string
    Username     string
    PasswordRef  string  // K8s secret reference
    SSLMode      string  // "require"
}

// YAMLResource for Advanced Mode
type YAMLResource struct {
    Kind     string // "Cluster", "Pooler", "ScheduledBackup"
    Name     string
    YAML     string // The full YAML content
    Checksum string // For diff detection
}
```

---

# CNPG Provider Implementation (Phase 1)

The CNPG provider is the first (and currently only) implementation:

```go
type CNPGProvider struct {
    kubeClient client.Client     // K8s client (via agent)
    pgClient   *pgxpool.Pool     // PG connection (for stats, users)
}

func (p *CNPGProvider) Name() string        { return "cnpg" }
func (p *CNPGProvider) DisplayName() string { return "PostgreSQL" }
```

### How It Maps to CNPG CRDs

| Provider Method | CNPG Action |
|----------------|-------------|
| `Provision()` | Create `Cluster` CR + `Pooler` CR + `ScheduledBackup` CR |
| `Scale()` | Update `Cluster.spec.instances` |
| `Upgrade()` | Update `Cluster.spec.imageName` (rolling update) |
| `Delete()` | Delete `Cluster` CR (PVCs retained if `retainVolumes=true`) |
| `PowerOff()` | Delete `Cluster` CR with `retainVolumes=true`, agent reports to svc-infra to scale nodes to 0 |
| `PowerOn()` | svc-infra scales nodes up, then re-apply `Cluster` CR with existing PVCs |
| `TriggerBackup()` | Create `Backup` CR |
| `Restore()` | Create new `Cluster` CR with `bootstrap.recovery` |
| `CreateUser()` | Execute SQL: `CREATE ROLE ... LOGIN PASSWORD ...` |
| `UpdateConfig()` | Update `Cluster.spec.postgresql.parameters` |
| `EnableExtension()` | Update `Cluster.spec.postgresql.shared_preload_libraries` + SQL `CREATE EXTENSION` |
| `GenerateYAML()` | Render CNPG CRD templates with ClusterSpec values |

---

# Adding a New Provider (Future)

To add Strimzi (Kafka) support:

1. **Create** `provider-strimzi/` repository
2. **Implement** the `Provider` interface for Strimzi CRDs
3. **Map** Strimzi CRDs to provider methods:
   - `Provision()` → Create `Kafka` CR
   - `Scale()` → Update `Kafka.spec.kafka.replicas`
   - `CreateUser()` → Create `KafkaUser` CR
   - etc.
4. **Register** the provider in the provider registry
5. **Update** the agent to watch Strimzi CRDs (auto-detected)
6. **Add** Kafka-specific UI components to dashboard
7. Core services (provisioner, billing, audit) work automatically — they call the interface, not the implementation.

---

# Provider Registry

```go
// Registry holds all available providers
type Registry struct {
    providers map[string]Provider
}

func NewRegistry() *Registry {
    r := &Registry{providers: make(map[string]Provider)}
    r.Register(cnpg.NewProvider())     // Phase 1
    // r.Register(strimzi.NewProvider()) // Phase 3
    // r.Register(redis.NewProvider())   // Phase 3
    return r
}

func (r *Registry) Get(name string) (Provider, error) {
    p, ok := r.providers[name]
    if !ok {
        return nil, fmt.Errorf("provider %q not found", name)
    }
    return p, nil
}

func (r *Registry) List() []Provider {
    // Returns all registered providers
}
```

---

# Design Decisions

| Decision | Rationale |
|----------|-----------|
| **Go interface** | Type-safe, compile-time verification, idiomatic for K8s ecosystem |
| **Single interface for all providers** | Core services don't need provider-specific code |
| **Provider methods are high-level** | Providers handle CRD-specific details internally |
| **YAML generation in provider** | Each provider knows its CRD schema |
| **Agent auto-detection** | Agent discovers which operators are installed, activates relevant modules |

---

*Maintained by: Backend Team*
*Last updated: February 2026*
