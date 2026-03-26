# Kiven — Development Strategy & Local Dev Guide

> **Back to**: [Architecture Overview](../EntrepriseArchitecture.md)

---

# Development Environments

## Overview

```
Level 1: LOCAL ($0/mo)          Level 2: SANDBOX (~$400/mo)       Level 3: STAGING/PROD
─────────────────────           ──────────────────────────         ──────────────────────
kiven-dev repo                  AWS Account: kiven-sandbox        AWS Accounts:
  Docker Compose (shared)       ├── EKS "kiven-dev"               kiven-staging
  kind cluster (shared)         │   └── Kiven services            kiven-prod
  Tilt (orchestrates all)       └── EKS "test-client"
                                    └── Agent + CNPG              Real Aiven (PG, Kafka)
Each svc-* repo:                                                  Real customers
  Go code + Dockerfile          Full AWS integration:
  task init (mise + tools)      node groups, EBS, S3, IAM
  Own CI (reusable GH wf)
                                Flux deployment
```

---

# Architecture: Polyrepo + Dev Orchestrator

## Why Polyrepo

- Each service has its **own repo**, its **own CI** (reusable GitHub workflows), its **own release cycle**
- Shared infrastructure (Docker Compose, kind, Tilt) lives in **one `kiven-dev` repo**
- Shared Go code lives in **`kiven-go-sdk`** (imported as a Go module)
- No port conflicts, no duplicate infra

## Repo Layout

```
kivenio/                              ← GitHub Organization
│
├── kiven-dev/                        ← DEV ORCHESTRATOR (this section)
│   ├── docker-compose.yml            ← ONE PostgreSQL, Redpanda, Valkey, MinIO
│   ├── kind/
│   │   ├── cluster.yaml              ← ONE kind cluster
│   │   └── cnpg-test-cluster.yaml    ← Test PG cluster (simulates customer DB)
│   ├── Tiltfile                      ← Orchestrates ALL services for local dev
│   ├── init-db.sql                   ← Product DB schema + seed data
│   ├── .mise.toml                    ← Shared tool versions (Go, Node, kubectl, helm...)
│   └── Taskfile.yml                  ← task dev, task infra:up, task kind:create
│
├── kiven-go-sdk/                     ← SHARED GO CODE (imported as module)
│   ├── provider/                     ← Provider interface (provider.go, registry.go)
│   ├── grpcapi/                      ← gRPC types (generated from proto)
│   ├── models/                       ← Shared domain models
│   └── go.mod                        ← github.com/kivenio/kiven-go-sdk
│
├── contracts-proto/                  ← PROTOBUF DEFINITIONS
│   ├── agent/v1/agent.proto          ← Agent ↔ SaaS protocol
│   ├── api/v1/services.proto         ← REST API types
│   └── buf.yaml
│
├── svc-api/                          ← SERVICE REPO (one of many)
│   ├── cmd/main.go
│   ├── internal/                     ← Service-specific logic
│   ├── Dockerfile
│   ├── .mise.toml                    ← Tool versions for THIS service
│   ├── Taskfile.yml                  ← task init, task run, task test, task build
│   ├── .github/workflows/ci.yml     ← Uses reusable workflow
│   └── go.mod                        ← imports github.com/kivenio/kiven-go-sdk
│
├── svc-provisioner/                  ← Same structure as svc-api
├── svc-agent-relay/                  ← Same structure
├── svc-clusters/                     ← Same structure
├── svc-backups/                      ← Same structure
├── svc-monitoring/                   ← Same structure
├── svc-users/                        ← Same structure
├── kiven-agent/                      ← Same structure (deployed in customer K8s)
├── provider-cnpg/                    ← Same structure (Go library)
├── dashboard/                        ← Next.js frontend
│   ├── src/
│   ├── .mise.toml
│   ├── Taskfile.yml
│   └── package.json
│
└── platform-github-management/       ← Repo management + reusable workflows
    └── .github/workflows/
        └── reusable-go-ci.yml        ← Reusable CI for all Go services
```

## How It Fits Together

```
┌─── kiven-dev (Dev Orchestrator) ──────────────────────────────────┐
│                                                                    │
│  Docker Compose (shared infra)    kind cluster (shared K8s)       │
│  ┌────────┐ ┌────────┐           ┌────────────────────────────┐   │
│  │Postgres│ │Redpanda│           │ CNPG Operator              │   │
│  │ :5432  │ │ :19092 │           │ kiven-agent (from repo)    │   │
│  ├────────┤ ├────────┤           │ test-pg cluster             │   │
│  │ Valkey │ │ MinIO  │           └────────────────────────────┘   │
│  │ :6379  │ │ :9000  │                                           │
│  └────────┘ └────────┘                                           │
│                                                                    │
│  Tilt (watches all service repos, builds & runs them)             │
│  ┌────────┐ ┌──────────────┐ ┌───────────────┐ ┌──────────┐     │
│  │svc-api │ │svc-agent-relay│ │svc-provisioner│ │svc-clusters│   │
│  │ :8080  │ │  :9090 gRPC  │ │  :8082        │ │  :8083    │    │
│  └────────┘ └──────────────┘ └───────────────┘ └──────────┘     │
│                                                                    │
│  Dashboard (from dashboard/ repo)                                 │
│  ┌──────────┐                                                     │
│  │ Next.js  │                                                     │
│  │  :3000   │                                                     │
│  └──────────┘                                                     │
└────────────────────────────────────────────────────────────────────┘
```

---

# Level 1: Local Development

## Prerequisites (via mise)

Each repo has a `.mise.toml`. The `kiven-dev` repo has the global one:

```toml
# kiven-dev/.mise.toml
[tools]
go = "1.23"
node = "22"
kubectl = "latest"
helm = "latest"
kind = "latest"
tilt = "latest"
buf = "latest"
task = "latest"
golangci-lint = "latest"
```

```bash
# Install mise (one time)
curl https://mise.run | sh

# Install all tools (in kiven-dev/)
mise install
```

## Quick Start

```bash
# 1. Clone the dev orchestrator
git clone git@github.com:kivenio/kiven-dev.git
cd kiven-dev

# 2. Install tools via mise
mise install

# 3. Clone the service repos you need (siblings of kiven-dev)
task repos:clone          # Clones all service repos next to kiven-dev/

# 4. Start shared infrastructure
task infra:up             # Docker Compose: PostgreSQL, Redpanda, Valkey, MinIO

# 5. Create kind cluster + CNPG
task kind:create          # kind cluster + CNPG operator + namespaces

# 6. Deploy test PostgreSQL cluster
task cnpg:deploy          # Creates a CNPG PostgreSQL cluster inside kind
                          # This simulates a customer's database.
                          # The agent watches this cluster and reports to svc-agent-relay.

# 7. Start all services with Tilt
tilt up                   # Watches all repos, builds, runs, shows logs
                          # Open http://localhost:10350 for Tilt dashboard

# OR start individual services manually:
task svc:api              # Runs svc-api from ../svc-api/
task svc:relay            # Runs svc-agent-relay from ../svc-agent-relay/
task agent                # Runs kiven-agent from ../kiven-agent/
task frontend             # Runs dashboard from ../dashboard/
```

## What Is the Test CNPG Cluster?

When you run `task cnpg:deploy`, Kiven-dev creates a **real PostgreSQL cluster** inside kind using the CloudNativePG operator. This is what happens:

```
1. CNPG Operator (already installed in kind) receives a Cluster YAML
2. Operator creates a PostgreSQL pod (test-pg-1) in namespace kiven-databases
3. PostgreSQL starts with:
   - Database: "app"
   - User: "app_user" (password in K8s Secret)
   - pg_stat_statements enabled
   - Logs slow queries > 200ms

This cluster simulates a REAL CUSTOMER DATABASE.
The Kiven agent watches it, collects metrics, and reports to svc-agent-relay.
When you test provisioning in the dashboard, THIS is the cluster you see.
```

You can connect to it directly:
```bash
# Port-forward to the test PostgreSQL
kubectl port-forward -n kiven-databases svc/test-pg-rw 15432:5432

# Connect with psql
psql postgresql://app_user:<password>@localhost:15432/app

# Get the password
kubectl get secret -n kiven-databases test-pg-app -o jsonpath='{.data.password}' | base64 -d
```

## What You Can Test Locally

| Feature | Works Locally? | How |
|---------|---------------|-----|
| Agent ↔ CNPG | Yes | Agent watches test-pg in kind |
| Agent ↔ svc-agent-relay | Yes | gRPC on localhost:9090 |
| CNPG cluster provisioning | Yes | Agent applies YAML to kind |
| Backup to S3 | Yes | MinIO at localhost:9000 (S3-compatible) |
| PG metrics collection | Yes | Agent reads pg_stat_* from test-pg |
| User/database management | Yes | Agent runs SQL on test-pg |
| PgBouncer pooling | Yes | CNPG Pooler CRD on kind |
| Dashboard ↔ API | Yes | Next.js :3000 → svc-api :8080 |
| YAML editor (Advanced Mode) | Yes | svc-yamleditor generates YAML |
| DBA intelligence | Yes | svc-monitoring analyzes PG stats |
| Power off / Power on | Partial | Delete/recreate CNPG cluster. No node management. |
| **AWS node groups** | **No** | Needs real AWS (Level 2) |
| **AWS EBS/S3/IAM** | **No** | Needs real AWS or LocalStack |
| **Cross-account IAM** | **No** | Needs sandbox |
| **Multi-AZ** | **No** | kind is single-node |

## Stopping

```bash
tilt down                 # Stop all services
task infra:down           # Stop Docker Compose
task kind:delete          # Delete kind cluster
```

---

# Service Repo Structure

Every Go service repo follows the same structure:

```
kivenio/svc-api/                      ← Example service
├── cmd/
│   └── main.go                       ← Entry point
├── internal/
│   ├── handler/                      ← HTTP/gRPC handlers
│   ├── service/                      ← Business logic
│   └── repository/                   ← Database access
├── migrations/                       ← SQL migrations (if needed)
├── Dockerfile                        ← Multi-stage build
├── .mise.toml                        ← Tool versions for this service
├── Taskfile.yml                      ← Service-level tasks
├── go.mod                            ← imports github.com/kivenio/kiven-go-sdk
├── go.sum
├── .github/
│   └── workflows/
│       └── ci.yml                    ← Uses reusable workflow from platform-github-management
├── .golangci.yml                     ← Linter config
├── .gitignore
└── README.md
```

## Service Taskfile (per repo)

Each service has its own `Taskfile.yml`:

```yaml
# svc-api/Taskfile.yml
version: "3"

tasks:
  init:
    desc: "Initialize dev environment (mise + tools + dependencies)"
    cmds:
      - mise install
      - go mod download
      - echo "✅ Ready! Run 'task run' to start."

  run:
    desc: "Run the service locally"
    env:
      DATABASE_URL: "postgres://kiven:kiven-local-dev@localhost:5432/kiven?sslmode=disable"
      KAFKA_BROKERS: "localhost:19092"
      VALKEY_ADDR: "localhost:6379"
      PORT: "8080"
    cmds:
      - go run ./cmd/

  test:
    desc: "Run unit tests"
    cmds:
      - go test ./... -v -count=1 -race

  test:coverage:
    desc: "Run tests with coverage"
    cmds:
      - go test ./... -coverprofile=coverage.out -race
      - go tool cover -html=coverage.out -o coverage.html

  build:
    desc: "Build binary"
    cmds:
      - go build -o bin/svc-api ./cmd/

  lint:
    desc: "Run linter"
    cmds:
      - golangci-lint run ./...

  docker:build:
    desc: "Build Docker image"
    cmds:
      - docker build -t kivenio/svc-api:dev .

  proto:generate:
    desc: "Generate Go code from proto (if this service uses gRPC)"
    cmds:
      - buf generate
```

## Reusable GitHub Workflow

All service repos use the same CI workflow:

```yaml
# svc-api/.github/workflows/ci.yml
name: CI
on:
  push:
    branches: [main]
  pull_request:

jobs:
  ci:
    uses: kivenio/platform-github-management/.github/workflows/reusable-go-ci.yml@main
    with:
      go-version: "1.23"
    secrets: inherit
```

The reusable workflow (in `platform-github-management`) handles:
- Go build, test, lint
- Security scan (trivy)
- Docker build + push (on main)
- Deploy to sandbox (on main, if configured)

---

# Shared Go Code: kiven-go-sdk

Shared code that multiple services import:

```
kivenio/kiven-go-sdk/
├── provider/
│   ├── provider.go          ← Provider interface (30+ methods)
│   └── registry.go          ← Provider registry
├── grpcapi/
│   └── (generated from contracts-proto)
├── models/
│   ├── service.go           ← Service, Plan, Backup types
│   ├── cluster.go           ← ClusterSpec, ClusterStatus
│   └── user.go              ← DatabaseUser, UserSpec
├── config/
│   └── config.go            ← Shared config loading (env vars)
├── telemetry/
│   └── otel.go              ← OpenTelemetry setup
├── go.mod                   ← github.com/kivenio/kiven-go-sdk
└── go.sum
```

Each service imports it:
```go
// svc-api/go.mod
module github.com/kivenio/svc-api

require github.com/kivenio/kiven-go-sdk v0.1.0
```

---

# Tilt Configuration (kiven-dev)

Tilt watches all service repos and orchestrates local development:

```python
# kiven-dev/Tiltfile

# --- Shared infrastructure (already running via Docker Compose) ---
# PostgreSQL :5432, Redpanda :19092, Valkey :6379, MinIO :9000

# --- Go services ---
local_resource('svc-api',
    serve_cmd='cd ../svc-api && go run ./cmd/',
    deps=['../svc-api/cmd/', '../svc-api/internal/'],
    env={
        'PORT': '8080',
        'DATABASE_URL': 'postgres://kiven:kiven-local-dev@localhost:5432/kiven?sslmode=disable',
        'KAFKA_BROKERS': 'localhost:19092',
        'VALKEY_ADDR': 'localhost:6379',
    },
    labels=['backend'],
)

local_resource('svc-agent-relay',
    serve_cmd='cd ../svc-agent-relay && go run ./cmd/',
    deps=['../svc-agent-relay/cmd/', '../svc-agent-relay/internal/'],
    env={'GRPC_PORT': '9090'},
    labels=['backend'],
)

local_resource('svc-provisioner',
    serve_cmd='cd ../svc-provisioner && go run ./cmd/',
    deps=['../svc-provisioner/cmd/', '../svc-provisioner/internal/'],
    env={
        'PORT': '8082',
        'DATABASE_URL': 'postgres://kiven:kiven-local-dev@localhost:5432/kiven?sslmode=disable',
    },
    labels=['backend'],
)

local_resource('svc-clusters',
    serve_cmd='cd ../svc-clusters && go run ./cmd/',
    deps=['../svc-clusters/cmd/', '../svc-clusters/internal/'],
    env={
        'PORT': '8083',
        'DATABASE_URL': 'postgres://kiven:kiven-local-dev@localhost:5432/kiven?sslmode=disable',
    },
    labels=['backend'],
)

# --- Agent (runs against kind cluster) ---
local_resource('kiven-agent',
    serve_cmd='cd ../kiven-agent && go run ./cmd/',
    deps=['../kiven-agent/cmd/', '../kiven-agent/internal/'],
    env={
        'KUBECONFIG': os.environ.get('KUBECONFIG', os.path.expanduser('~/.kube/config')),
        'RELAY_ENDPOINT': 'localhost:9090',
    },
    labels=['agent'],
)

# --- Frontend ---
local_resource('dashboard',
    serve_cmd='cd ../dashboard && npm run dev',
    deps=['../dashboard/src/'],
    labels=['frontend'],
)
```

Tilt UI at `http://localhost:10350` shows all services, logs, status, restart buttons.

---

# Level 2: Sandbox (AWS)

## When to Use

Move to Level 2 when:
- Agent and core services work locally
- You need to test svc-infra (real AWS APIs)
- You need to test full provisioning pipeline (node groups → CNPG)
- You're preparing for first customer demo

## Architecture

```
AWS Account: kiven-sandbox (eu-west-1)
│
├── EKS "kiven-dev"
│   ├── Kiven services (deployed via Flux)
│   ├── Aiven VPC peering (product DB)
│   └── Platform stack (Prometheus, Loki, Flux)
│
├── EKS "test-client"
│   ├── Simulates a real customer cluster
│   ├── Kiven agent installed
│   ├── CNPG operator (installed by Kiven)
│   └── Full provisioning:
│       ├── Dedicated node group (created by svc-infra)
│       ├── CNPG cluster (created by agent)
│       ├── S3 backups (created by svc-infra)
│       └── Network policies, storage classes, IRSA
│
├── S3: kiven-backups-test-client
├── IAM: KivenAccessRole (simulates customer role)
└── IAM: IRSA roles for CNPG
```

## Cost Optimization

| Resource | Cost | Optimization |
|----------|------|-------------|
| EKS control plane x 2 | ~$146/mo | Can't avoid |
| EC2 nodes (kiven-dev, 2x t3.medium) | ~$60/mo | Power off nights/weekends |
| EC2 nodes (test-client, 2x t3.medium) | ~$60/mo | Power off when not testing |
| EBS volumes | ~$20/mo | Delete test data regularly |
| S3 | ~$5/mo | Lifecycle rules |
| **Total** | **~$300/mo** | **~$150/mo with power schedules** |

---

# Level 3: Staging & Production

Only needed when the product is ready for real customers.

| Environment | Account | EKS | Aiven | Purpose |
|-------------|---------|-----|-------|---------|
| Staging | kiven-staging | eks-staging | Staging plan | Pre-production validation |
| Production | kiven-prod | eks-prod | Business plan | Live product |

---

# Development Workflow

## Daily Workflow

```bash
cd kiven-dev

# Morning: start everything
task infra:up             # Docker Compose
task kind:create          # kind + CNPG (idempotent, skips if exists)
tilt up                   # All services + agent + dashboard

# Code in any svc-* repo → Tilt auto-reloads
# Dashboard at http://localhost:3000
# Tilt UI at http://localhost:10350

# End of day
tilt down
task infra:down
```

## Adding a New Service

```bash
# 1. Create repo from template
gh repo create kivenio/svc-my-service --template kivenio/platform-templates-service-go --private

# 2. Clone next to kiven-dev
cd .. && git clone git@github.com:kivenio/svc-my-service.git

# 3. Initialize
cd svc-my-service && task init

# 4. Add to Tiltfile in kiven-dev
# (add local_resource block)

# 5. Develop → test → PR → merge → CI runs automatically
```

## Adding a New Feature to Existing Service

```bash
cd svc-api                # Go to service repo
task init                 # Ensure tools are up to date (mise)
# ... code ...
task test                 # Run tests
task lint                 # Run linter
# Tilt auto-reloads if running
git add . && git commit && git push
# CI runs via reusable workflow
```

---

*Maintained by: Platform Team*
*Last updated: February 2026*
