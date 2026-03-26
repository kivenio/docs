# Customer Onboarding
## *From Sign-Up to Running Database in 10 Minutes*

---

> **Back to**: [Architecture Overview](../EntrepriseArchitecture.md)

---

# Onboarding Flow

```
Step 1          Step 2              Step 3           Step 4           Step 5
Sign up   →   Deploy TF      →   Connect EKS   →  Create DB    →  Connected!
(1 min)       module (2 min)     (1 min)          (5-7 min)       
```

---

# Step 1: Sign Up (1 minute)

Customer creates account on kiven.io:
- Email + password or SSO (Google / GitHub)
- Create organization
- Invite team members (optional)

---

# Step 2: Deploy Terraform Module (2 minutes)

Customer deploys Kiven's Terraform module in their AWS account. This creates the `KivenAccessRole` IAM role.

### How It Works

1. Kiven dashboard shows: "Connect your AWS account"
2. Customer copies the Terraform module configuration (or uses the Terraform Registry)
3. Customer runs `terraform init` and `terraform apply`
4. Module creates:
   - IAM Role `KivenAccessRole` (trusts Kiven's AWS account)
   - IAM Policy `KivenAccessPolicy` (scoped permissions)
   - ExternalId parameter (unique per customer, prevents confused deputy)
5. Terraform outputs: Role ARN → customer copies back to Kiven dashboard

### Terraform Module (Summary)

```hcl
module "kiven_access" {
  source  = "kivenio/kiven/aws"
  version = "~> 1.0"

  external_id      = var.kiven_external_id  # Provided by Kiven dashboard
  kiven_account_id = "123456789012"          # Kiven's AWS account ID
}

output "role_arn" {
  description = "Paste this ARN in the Kiven dashboard"
  value       = module.kiven_access.role_arn
}
```

---

# Step 3: Connect EKS Cluster (1 minute)

Customer provides their EKS cluster details:

1. Select AWS region (auto-detected from IAM role)
2. Select EKS cluster (Kiven lists available clusters via AWS API)
3. Kiven validates:
   - Can assume role ✓
   - Can describe EKS cluster ✓
   - Can access K8s API ✓
4. Dashboard shows: "Cluster connected!"

### Cluster Discovery

Kiven automatically discovers:
- EKS version
- VPC / subnets / AZs
- Existing node groups
- Installed operators (CNPG, cert-manager)
- Available storage classes
- Resource capacity (CPU, memory)

---

# Step 4: Create Database (5-7 minutes)

Customer clicks "Create Database" and configures:

### Simple Mode (Form)

| Field | Options | Default |
|-------|---------|---------|
| **Name** | Free text | `my-database` |
| **PostgreSQL Version** | 15, 16, 17 | 17 |
| **Plan** | Hobbyist, Startup, Business, Premium, Custom | Startup |
| **Region / AZ** | From customer's EKS subnets | Multi-AZ (auto) |
| **Initial Database** | Database name | `app` |
| **Initial User** | Username | `app_user` |

### What Happens Behind the Scenes

```
1. Prerequisites Check (svc-provisioner)              [~10s]
   ├── Validate IAM permissions
   ├── Check EKS cluster health
   ├── Verify subnet availability across AZs
   └── Check resource capacity

2. Infrastructure Setup (svc-infra)                    [~2-3min]
   ├── Create dedicated node group (kiven-db-{id})
   ├── Create StorageClass (kiven-db-gp3)
   ├── Create S3 bucket (kiven-backups-{customer-id})
   ├── Create IRSA role (kiven-cnpg-backup-{id})
   └── Wait for nodes ready

3. CNPG Setup (agent)                                  [~1min]
   ├── Create namespace (kiven-databases)
   ├── Install CNPG operator (if not present)
   ├── Deploy Kiven agent
   └── Apply NetworkPolicies

4. Database Provisioning (agent)                       [~2-3min]
   ├── Apply CNPG Cluster YAML
   ├── Apply PgBouncer Pooler YAML
   ├── Apply ScheduledBackup YAML
   ├── Wait for primary ready
   ├── Wait for replicas synced
   └── Create initial database + user

5. Ready!                                              [total: ~5-7min]
   └── Return connection strings
```

### Dashboard Progress

```
┌──────────────────────────────────────────────────────────────┐
│  Creating database: my-database                              │
│                                                              │
│  [████████████████████████████░░░░░░░░░░] 72%               │
│                                                              │
│  ✅ Prerequisites validated                                  │
│  ✅ Node group created (2× r6g.medium)                      │
│  ✅ Storage and backups configured                           │
│  ✅ CNPG operator ready                                      │
│  ⏳ PostgreSQL starting...                                   │
│  ○  Replicas syncing                                        │
│  ○  Creating database and user                              │
│                                                              │
│  Estimated time remaining: ~2 minutes                        │
└──────────────────────────────────────────────────────────────┘
```

---

# Step 5: Connected!

Customer receives:

```
┌──────────────────────────────────────────────────────────────┐
│  ✅ Database ready!                                          │
│                                                              │
│  Connection Details:                                         │
│                                                              │
│  Host:     pg-my-database-rw.kiven-databases.svc             │
│  Port:     5432                                              │
│  Database: app                                               │
│  User:     app_user                                          │
│  Password: ••••••••••  [Reveal] [Copy]                       │
│                                                              │
│  Pooler (recommended):                                       │
│  Host:     pg-my-database-pooler.kiven-databases.svc         │
│  Port:     5432                                              │
│                                                              │
│  Connection String:                                          │
│  postgresql://app_user:***@pg-my-database-pooler             │
│    .kiven-databases.svc:5432/app?sslmode=require             │
│                                            [Copy]            │
│                                                              │
│  [Open Dashboard]  [View Metrics]  [Add User]                │
└──────────────────────────────────────────────────────────────┘
```

---

# Offboarding

When a customer deletes their database:

1. **Confirmation dialog**: "This will delete your database. Backups will be retained for 30 days."
2. CNPG cluster deleted
3. Node group deleted
4. EBS volumes retained for 7 days, then deleted (configurable)
5. S3 backups retained for 30 days (configurable)
6. IAM IRSA role deleted
7. Audit log entry created

When a customer removes Kiven entirely:
1. All databases must be deleted first (or exported)
2. Kiven agent uninstalled (`helm uninstall kiven-agent`)
3. Customer runs `terraform destroy` (removes IAM role)
4. Kiven retains customer metadata for 90 days (GDPR), then purges

---

*Maintained by: Product Team + Platform Team*
*Last updated: February 2026*
