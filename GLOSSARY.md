# Glossary
## *Kiven Platform Terminology*

---

> **Back to**: [Architecture Overview](EntrepriseArchitecture.md)

---

# 1. Kiven-Specific Terms

| Term | Definition |
|------|------------|
| **Kiven** | Managed data services platform. "Aiven, but on your Kubernetes infrastructure." Finnish for "stone" — solid ground for your database. |
| **Kiven Agent** | Lightweight Go binary deployed in the customer's K8s cluster. Executes commands, collects metrics/logs, reports status to Kiven SaaS via gRPC/mTLS. |
| **Kiven SaaS** | The management platform running in Kiven's AWS account (eu-west-1). Dashboard, API, core services. |
| **Provider** | Plugin that implements the Kiven provider interface for a specific K8s operator (e.g., CNPG Provider, Strimzi Provider). |
| **CNPG Provider** | The first Kiven provider. Manages PostgreSQL via the CloudNativePG operator. |
| **Service Plan** | Predefined resource tier (Hobbyist, Startup, Business, Premium, Custom) that maps to EC2 instance type, storage, instances, and postgresql.conf tuning. |
| **Power Off / Power On** | Feature to pause a database by deleting compute (nodes + pods) while retaining data (EBS volumes + S3 backups). Saves 60-70% on non-production environments. |
| **Power Schedule** | Automated schedule for power on/off (e.g., Mon-Fri 8am-6pm). |
| **Simple Mode** | Default dashboard UX for developers. Forms, sliders, buttons. No YAML visible. Like Aiven's UI. |
| **Advanced Mode** | Dashboard UX for DevOps. View/edit YAML directly, diff view, change history, rollback. Like Lens for K8s. |
| **svc-provisioner** | "The Brain" — core service that orchestrates full provisioning pipeline (nodes → storage → S3 → CNPG → PG). |
| **svc-infra** | Service managing AWS resources in customer accounts (EC2 node groups, EBS, S3, IAM). |
| **svc-agent-relay** | gRPC server that multiplexes connections from all customer agents. |
| **svc-yamleditor** | Service powering Advanced Mode: YAML generation, validation, diff, change history. |
| **DBA Intelligence** | Kiven's automated database expertise: performance tuning, query optimization, backup verification, capacity planning, security auditing, incident diagnostics. |
| **Backup Verification** | Automated weekly restore test: spin up temporary CNPG cluster from latest backup, validate, tear down. Proves backups are restorable. |
| **Prerequisites Engine** | Validates customer's K8s environment before provisioning (CNPG operator, storage classes, resources, cert-manager, etc.). |
| **Customer Infrastructure** | AWS resources in the customer's account managed by Kiven: node groups, EBS volumes, S3 buckets, IAM roles. |
| **Cross-Account IAM** | AWS IAM role in customer's account that trusts Kiven's account. Kiven assumes this role to manage customer resources. |

---

# 2. CloudNativePG (CNPG) Terms

| Term | Definition |
|------|------------|
| **CloudNativePG (CNPG)** | CNCF Kubernetes operator for PostgreSQL. Manages cluster lifecycle, HA, backups, failover. |
| **CNPG Cluster** | Custom Resource (CR) defining a PostgreSQL cluster: instances, storage, config, backups. |
| **CNPG Pooler** | Custom Resource for PgBouncer connection pooling, managed by CNPG operator. |
| **CNPG ScheduledBackup** | Custom Resource defining automated backup schedule (frequency, retention, S3 target). |
| **Barman** | Backup tool used by CNPG for physical backups and WAL archiving to object storage (S3). |
| **PITR (Point-in-Time Recovery)** | Ability to restore a database to any specific moment using base backup + WAL replay. |
| **WAL (Write-Ahead Log)** | PostgreSQL's transaction log. Every change is written to WAL before data files. Used for replication and PITR. |
| **Switchover** | Planned promotion of a replica to primary (graceful, zero data loss). |
| **Failover** | Automatic promotion of a replica when primary fails (may lose last few transactions depending on replication mode). |
| **Replication Lag** | Time delay between primary writing data and replica receiving it. |
| **PVC (Persistent Volume Claim)** | Kubernetes resource requesting persistent storage (maps to EBS volume). |
| **PVC Reclaim Policy** | What happens to the EBS volume when the PVC is deleted. `Retain` = keep the volume (critical for Power Off/On). |

---

# 3. PostgreSQL Terms

| Term | Definition |
|------|------------|
| **postgresql.conf** | Main PostgreSQL configuration file. Controls memory, connections, WAL, checkpoints, etc. |
| **pg_hba.conf** | PostgreSQL Host-Based Authentication config. Controls who can connect and how. |
| **shared_buffers** | RAM allocated for caching data pages. Typically 25% of total RAM. |
| **work_mem** | RAM per query operation for sorting/hashing. Too low = spills to disk. |
| **effective_cache_size** | Hint to query planner about available cache. Typically 75% of RAM. |
| **max_connections** | Maximum concurrent connections. Should be sized with connection pooling. |
| **PgBouncer** | PostgreSQL connection pooler. Reduces connection overhead. Modes: session, transaction, statement. |
| **pg_stat_statements** | Extension tracking execution statistics of all SQL queries. |
| **pg_stat_activity** | System view showing currently active queries and connections. |
| **pg_stat_bgwriter** | System view for background writer and checkpoint statistics. |
| **pg_stat_user_tables** | System view for table-level statistics (seq scans, idx scans, dead tuples). |
| **Autovacuum** | Background process that reclaims dead tuples and updates statistics. |
| **Bloat** | Wasted space from dead tuples that autovacuum hasn't reclaimed. |
| **XID Wraparound** | PostgreSQL transaction ID limit (~2 billion). If reached, database freezes. Autovacuum prevents this. |
| **EXPLAIN / EXPLAIN ANALYZE** | Commands showing query execution plan (estimated vs actual). |
| **Sequential Scan** | Full table scan. Often indicates missing index. |
| **Index Scan** | Targeted lookup using an index. Generally faster than seq scan. |
| **Extensions** | PostgreSQL plugins: pg_vector (AI embeddings), PostGIS (geospatial), TimescaleDB (time-series), etc. |

---

# 4. AWS / Cloud Terms

| Term | Definition |
|------|------------|
| **EKS (Elastic Kubernetes Service)** | AWS managed Kubernetes service. |
| **EBS (Elastic Block Store)** | AWS block storage for EC2. Volumes attached to K8s nodes for database data. |
| **gp3** | EBS volume type. General purpose SSD with configurable IOPS and throughput. Default for Kiven. |
| **S3 (Simple Storage Service)** | AWS object storage. Used for CNPG backups (Barman) and WAL archiving. |
| **IRSA (IAM Roles for Service Accounts)** | AWS feature mapping K8s ServiceAccounts to IAM roles. CNPG uses IRSA to write backups to S3. |
| **AssumeRole** | AWS IAM action to temporarily take on another role's permissions. Kiven assumes customer's `KivenAccessRole`. |
| **Cross-Account Access** | Pattern where one AWS account accesses resources in another account via IAM role trust. |
| **CloudFormation** | AWS IaC service. Kiven provides a CF template for customers to create the access role. |
| **KMS (Key Management Service)** | AWS encryption key management. Used for EBS and S3 encryption. |
| **Managed Node Group** | EKS feature for managed EC2 instances as K8s worker nodes. Kiven creates dedicated node groups for databases. |
| **Taints** | K8s mechanism to repel pods from nodes. Kiven taints DB nodes so only DB pods run there. |
| **Tolerations** | K8s mechanism allowing pods to schedule on tainted nodes. CNPG pods tolerate the database taint. |
| **Multi-AZ** | Deploying across multiple Availability Zones for high availability. Kiven spreads primary/replicas across AZs. |

---

# 5. Kubernetes & Operator Terms

| Term | Definition |
|------|------------|
| **CRD (Custom Resource Definition)** | Extends K8s API with custom resources. CNPG adds Cluster, Backup, Pooler CRDs. |
| **CR (Custom Resource)** | Instance of a CRD. A CNPG `Cluster` CR defines one PostgreSQL cluster. |
| **Operator** | K8s controller that manages complex applications via CRDs. CNPG operator manages PostgreSQL. |
| **Controller** | Control loop watching K8s resources and reconciling actual vs desired state. |
| **Reconciliation Loop** | Continuous process comparing desired state (YAML) with actual state and making corrections. |
| **client-go** | Official Go client library for Kubernetes API. Used by Kiven agent. |
| **controller-runtime** | Go library for building K8s controllers/operators. Used by Kiven agent. |
| **Informer** | K8s pattern for watching resource changes efficiently. Agent uses informers for CNPG CRDs. |
| **Namespace** | K8s logical isolation. Kiven uses `kiven-system` (agent + operator) and `kiven-databases` (PG clusters). |
| **NetworkPolicy** | K8s L3/L4 firewall rules. Kiven creates policies so only authorized app pods reach the database. |
| **StorageClass** | K8s abstraction for dynamic storage provisioning. Kiven creates optimized storage classes for DB workloads. |
| **Helm** | K8s package manager. Agent and CNPG operator are installed via Helm charts. |

---

# 6. Communication & Protocol Terms

| Term | Definition |
|------|------------|
| **gRPC** | High-performance RPC framework by Google. Used for agent ↔ Kiven SaaS communication. |
| **mTLS (Mutual TLS)** | Both client and server verify each other's certificates. Used for agent ↔ SaaS security. |
| **Protobuf** | Protocol Buffers — binary serialization format for gRPC messages. |
| **Bidirectional Streaming** | gRPC feature where both sides can send messages continuously. Agent streams metrics, SaaS streams commands. |
| **Outbound-Only** | Agent initiates the connection to Kiven SaaS. No inbound ports needed on customer's firewall. |

---

# 7. Architecture & Software Terms

| Term | Definition |
|------|------------|
| **Provider Interface** | Go interface that each data service (CNPG, Strimzi, Redis) implements. Enables multi-operator support. |
| **Plugin Architecture** | Design pattern where functionality is added via plugins without modifying core code. |
| **GitOps** | Managing infrastructure and apps using Git as single source of truth. ArgoCD pulls from Git. |
| **Infrastructure as Code (IaC)** | Managing infra through code (Terraform, CloudFormation) rather than manual processes. |
| **Trunk-Based Development** | All developers merge to main branch. Short-lived feature branches. |
| **C4 Model** | Architecture documentation: Context, Container, Component, Code diagrams. |
| **Defense in Depth** | Multiple security layers so one breach doesn't compromise everything. |
| **Zero Trust** | Never trust, always verify. Every request is authenticated and authorized. |
| **RBAC (Role-Based Access Control)** | Permissions based on roles (Admin, Operator, Viewer). |
| **OIDC (OpenID Connect)** | Identity protocol for SSO. Login with Google, GitHub, SAML. |
| **Idempotency** | Operation producing the same result no matter how many times executed. Critical for agent commands. |

---

# 8. Observability & Reliability Terms

| Term | Definition |
|------|------------|
| **RPO (Recovery Point Objective)** | Maximum acceptable data loss in time. RPO 1h = can lose up to 1 hour of data. |
| **RTO (Recovery Time Objective)** | Maximum acceptable downtime. RTO 15min = must recover within 15 minutes. |
| **SLI (Service Level Indicator)** | Metric measuring service behavior (e.g., availability, latency). |
| **SLO (Service Level Objective)** | Target for an SLI (e.g., 99.9% availability). |
| **SLA (Service Level Agreement)** | Contractual commitment to SLO with consequences for breach. |
| **Error Budget** | Allowable unreliability: 100% - SLO. 99.9% SLO = 43 min/month error budget. |
| **Prometheus** | Time-series database for metrics. Collects from Kiven services and agents. |
| **Loki** | Log aggregation system by Grafana. Stores and queries logs. |
| **Tempo** | Distributed tracing system by Grafana. Traces requests across services. |
| **OpenTelemetry (OTel)** | Standard for telemetry (metrics, logs, traces) collection and export. |
| **Chaos Engineering** | Deliberately injecting failures to test system resilience. |

---

# 9. Business & Compliance Terms

| Term | Definition |
|------|------------|
| **GDPR** | EU General Data Protection Regulation. Requires data residency, consent, right to erasure. |
| **SOC2** | Security framework requiring audit controls, RBAC, monitoring, incident response. |
| **Data Sovereignty** | Data stored and processed within specific geographic boundaries. Kiven's model ensures this — data stays in customer's VPC. |
| **Vendor Lock-In** | Dependency on a specific vendor. Kiven reduces lock-in: customer owns their K8s infra, CNPG is open-source. |
| **DBaaS (Database-as-a-Service)** | Fully managed database. Aiven and Kiven are both DBaaS, but with different infrastructure models. |
| **BYOC (Bring Your Own Cloud)** | Model where the managed service runs on the customer's cloud account. Kiven's core model. |
| **Stripe** | Payment platform for SaaS billing. Kiven uses Stripe for subscription management. |

---

# 10. How to Use These Terms

## In PR Reviews
- *"This increases blast radius for customer data"*
- *"We need idempotency on this agent command"*
- *"Check the PVC reclaim policy — must be Retain for power off"*

## In Customer Conversations
- *"Your data never leaves your VPC"*
- *"You can power off dev databases on weekends to save 70%"*
- *"Our DBA intelligence will auto-tune your postgresql.conf"*

## In Architecture Decisions
- *"We need cross-account IAM for svc-infra to manage customer node groups"*
- *"The provider interface must be stable before we add Strimzi support"*

---

*Maintained by: Platform Team*
*Last updated: February 2026*
