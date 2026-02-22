# Customer Infrastructure Management
## *How Kiven Manages AWS Resources in Customer Accounts*

---

> **Back to**: [Architecture Overview](../EntrepriseArchitecture.md)

---

# Overview

Kiven manages **four types of AWS resources** in the customer's account:

1. **EKS Node Groups** — Dedicated compute for databases
2. **EBS Volumes** — Persistent storage for PostgreSQL data
3. **S3 Buckets** — Backup storage (Barman + WAL archiving)
4. **IAM Roles** — IRSA for CNPG to access S3

All managed via `svc-infra`, which assumes the customer's `KivenAccessRole` IAM role.

---

# Access Model

## Cross-Account IAM

```
┌─── Kiven AWS Account ──────────┐     ┌─── Customer AWS Account ────────────┐
│                                 │     │                                      │
│  svc-infra                      │     │  IAM Role: KivenAccessRole           │
│  ├── IRSA: svc-infra-role       │────▶│  ├── Trust: Kiven account ID         │
│  └── AssumeRole call            │     │  ├── Policy: KivenAccessPolicy       │
│                                 │     │  └── ExternalId: unique per customer │
│                                 │     │                                      │
└─────────────────────────────────┘     └──────────────────────────────────────┘
```

## IAM Policy (KivenAccessPolicy)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "EKSAccess",
      "Effect": "Allow",
      "Action": [
        "eks:DescribeCluster",
        "eks:ListNodegroups",
        "eks:DescribeNodegroup",
        "eks:CreateNodegroup",
        "eks:UpdateNodegroupConfig",
        "eks:DeleteNodegroup"
      ],
      "Resource": "arn:aws:eks:*:*:cluster/*"
    },
    {
      "Sid": "EC2ForNodeGroups",
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeInstances",
        "ec2:DescribeVolumes",
        "ec2:DescribeSubnets",
        "ec2:DescribeSecurityGroups",
        "ec2:CreateLaunchTemplate",
        "ec2:DeleteLaunchTemplate",
        "ec2:RunInstances",
        "ec2:TerminateInstances"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:RequestTag/managed-by": "kiven"
        }
      }
    },
    {
      "Sid": "S3BackupBucket",
      "Effect": "Allow",
      "Action": [
        "s3:CreateBucket",
        "s3:PutBucketEncryption",
        "s3:PutBucketLifecycleConfiguration",
        "s3:PutBucketVersioning",
        "s3:PutBucketPolicy",
        "s3:GetBucketLocation",
        "s3:ListBucket"
      ],
      "Resource": "arn:aws:s3:::kiven-backups-*"
    },
    {
      "Sid": "IAMForIRSA",
      "Effect": "Allow",
      "Action": [
        "iam:CreateRole",
        "iam:DeleteRole",
        "iam:AttachRolePolicy",
        "iam:DetachRolePolicy",
        "iam:PutRolePolicy",
        "iam:DeleteRolePolicy",
        "iam:GetRole",
        "iam:TagRole"
      ],
      "Resource": "arn:aws:iam::*:role/kiven-*"
    },
    {
      "Sid": "KMSForEncryption",
      "Effect": "Allow",
      "Action": [
        "kms:DescribeKey",
        "kms:CreateGrant",
        "kms:Encrypt",
        "kms:Decrypt",
        "kms:GenerateDataKey"
      ],
      "Resource": "*"
    }
  ]
}
```

**Key security constraints:**
- EC2 actions limited to resources tagged `managed-by: kiven`
- S3 actions limited to `kiven-backups-*` bucket prefix
- IAM actions limited to `kiven-*` role prefix
- ExternalId required to prevent confused deputy attacks

---

# Resource Management

## 1. Node Groups

### Create (on provisioning)

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| Name | `kiven-db-{cluster-id}` | Unique per database cluster |
| Instance type | Per service plan (t3.small → r6g.xlarge) | Memory-optimized for PG |
| Desired/min/max | Per plan (1-3 nodes) | HA: primary + replicas |
| Subnets | Multi-AZ (customer's private subnets) | Spread across AZs |
| Taints | `kiven.io/role=database:NoSchedule` | Only DB pods run here |
| Labels | `kiven.io/managed=true`, `kiven.io/cluster-id={id}` | Identification |
| Tags | `managed-by: kiven`, `kiven-cluster-id: {id}` | AWS-level tracking |
| AMI | EKS-optimized AL2023 | Standard, secure |

### Scale to Zero (Power Off)

```
svc-infra → UpdateNodegroupConfig:
  scalingConfig:
    minSize: 0
    desiredSize: 0
    maxSize: 0   (or original max)
```

Nodes are terminated. EBS volumes detach but are RETAINED (PVC reclaim policy = Retain).

### Scale Up (Power On)

```
svc-infra → UpdateNodegroupConfig:
  scalingConfig:
    minSize: {plan.instances}
    desiredSize: {plan.instances}
    maxSize: {plan.instances * 2}
```

New nodes join. CNPG pods re-created, PVCs reattached to new nodes.

### Delete (on cluster deletion)

Node group fully deleted only when customer deletes the database **and** confirms data destruction.

## 2. EBS Volumes

Managed indirectly via Kubernetes StorageClass + PVCs. Kiven creates the StorageClass:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: kiven-db-gp3
parameters:
  type: gp3
  iops: "3000"          # adjusted per plan
  throughput: "125"      # adjusted per plan
  encrypted: "true"
  kmsKeyId: <customer-kms-key>
provisioner: ebs.csi.aws.com
reclaimPolicy: Retain    # CRITICAL: keep volumes on PVC delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

**Monitoring:**
- Disk usage alerts at 70%, 80%, 90%
- IOPS utilization alerts
- Auto-resize recommendation via DBA intelligence

## 3. S3 Buckets

One bucket per customer (shared across their database clusters):

| Config | Value | Rationale |
|--------|-------|-----------|
| Name | `kiven-backups-{customer-id}` | Unique per customer |
| Region | Same as customer's EKS | Data locality |
| Encryption | SSE-KMS (customer's key) | Customer controls encryption |
| Versioning | Enabled | Protect against accidental delete |
| Lifecycle | Transition to IA after 30d, Glacier after 90d, delete after 365d | Cost optimization |
| Bucket policy | Only IRSA role can access | Least privilege |

## 4. IAM Roles (IRSA)

CNPG needs S3 access for backups. Kiven creates an IRSA role:

| Parameter | Value |
|-----------|-------|
| Role name | `kiven-cnpg-backup-{cluster-id}` |
| Trust | EKS OIDC provider + kiven-databases ServiceAccount |
| Policy | s3:PutObject, s3:GetObject, s3:DeleteObject on backup bucket |

---

# Tagging Strategy

All Kiven-managed resources are tagged:

| Tag Key | Value | Purpose |
|---------|-------|---------|
| `managed-by` | `kiven` | Identify Kiven resources |
| `kiven-cluster-id` | `{cluster-id}` | Link to specific database |
| `kiven-customer-id` | `{customer-id}` | Link to customer org |
| `kiven-plan` | `hobbyist/startup/business/premium/custom` | Service plan |
| `kiven-environment` | `production/staging/development` | Environment label |

These tags enable:
- Cost tracking per database cluster
- Resource cleanup on cluster deletion
- IAM policy conditions (only manage tagged resources)

---

# Audit & Compliance

Every AWS API call made by `svc-infra` is:
1. **Logged in CloudTrail** (customer's account) — they can see exactly what Kiven does
2. **Logged in Kiven audit** (`svc-audit`) — immutable record on our side
3. **Attributed** — which Kiven user/service triggered the action

---

# Cost Tracking

Kiven tracks estimated costs per cluster by monitoring:

| Resource | Cost Calculation |
|----------|-----------------|
| **EC2 nodes** | Instance type × hours running (from power on/off events) |
| **EBS storage** | Volume size × hours provisioned + IOPS cost |
| **S3 storage** | Bucket size × storage class pricing |
| **Data transfer** | Estimated from backup size + WAL volume |

Displayed in customer dashboard: "Estimated AWS cost: $X/month for this cluster"

---

*Maintained by: Platform Team*
*Last updated: February 2026*
