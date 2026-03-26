# 🛠️ **Platform Engineering**
## *LOCAL-PLUS Contracts, Golden Path & Operations*

---

> **Retour vers** : [Architecture Overview](../EntrepriseArchitecture.md)

---

# 📋 **Table of Contents**

1. [Platform Contracts](#platform-contracts)
2. [CI/CD & Delivery](#cicd--delivery)
3. [Golden Path (New Service Checklist)](#golden-path-new-service-checklist)
4. [On-Call Structure](#on-call-structure)
5. [Incident Management](#incident-management)
6. [Service Templates](#service-templates)

---

# 📜 **Platform Contracts**

## Guarantees

| Contrat | Garantie Platform | Responsabilité Service |
|---------|-------------------|------------------------|
| **Deployment** | Git push → Prod < 15min | Manifests K8s valides |
| **Secrets** | Vault dynamic, rotation auto | Utiliser External-Secrets |
| **Observability** | Auto-collection traces/metrics/logs | Instrumentation OTel |
| **Networking** | mTLS enforced, Gateway API | Déclarer routes dans HTTPRoute |
| **Scaling** | HPA disponible | Configurer requests/limits |
| **Security** | Policies enforced | Passer les policies |

---

# 🚀 **CI/CD & Delivery**

## GitOps avec Flux

| Concept | Implementation |
|---------|----------------|
| **Source of Truth** | Git repositories |
| **Delivery Model** | Pull-based (Flux reconciles from Git) |
| **Environments** | Kustomize overlays (dev/staging/prod) |
| **Promotion** | PR from dev → staging → prod overlays |

## GitHub Actions — Reusable Workflows

> Les workflows partagés standardisent les pipelines CI/CD.

| Type | Localisation | Usage |
|------|--------------|-------|
| **Reusable workflows** | `.github/workflows/` | Build, test, deploy partagés |
| **Composite actions** | `.github/actions/` | Steps communs réutilisables |

## Workflows Standards

| Workflow | Description | Repos cibles |
|----------|-------------|--------------|
| `ci-python.yml` | Lint, test, build | `svc-*`, `sdk-python` |
| `ci-terraform.yml` | Format, lint, plan, apply | `platform-*`, `bootstrap` |
| `cd-flux.yml` | Trigger Flux reconcile | Tous |
| `security-scan.yml` | Trivy, Checkov, tfsec | Tous |

## Pipeline Stages

```
┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐
│  Lint   │───►│  Test   │───►│  Build  │───►│  Scan   │───►│  Push   │
└─────────┘    └─────────┘    └─────────┘    └─────────┘    └─────────┘
     │              │              │              │              │
     │              │              │              │              ▼
     │              │              │              │      ┌─────────────┐
     │              │              │              │      │ Flux Reconcile │
     │              │              │              │      └─────────────┘
     ▼              ▼              ▼              ▼
   Fail fast    Coverage      Image tag     CVE check
```

## Deployment SLA

| Metric | Target | Measurement |
|--------|--------|-------------|
| **Git to Dev** | < 5 min | Commit to Flux reconcile |
| **Git to Staging** | < 10 min | Commit to Flux reconcile (manual approval) |
| **Git to Prod** | < 15 min | Commit to Flux reconcile (manual approval) |
| **Rollback** | < 2 min | Flux rollback |

## Observability Requirements

| Signal | Requirement | Enforcement |
|--------|-------------|-------------|
| **Metrics** | `/metrics` endpoint exposed | Kyverno policy |
| **Logs** | Structured JSON, no PII | OTel scrubbing |
| **Traces** | OTel SDK instrumentation | Service template |
| **Health** | `/health/live` + `/health/ready` | Kyverno policy |

## Security Baseline

| Requirement | Enforcement | Exception Process |
|-------------|-------------|-------------------|
| Non-root containers | Kyverno policy | ADR + Platform approval |
| Read-only filesystem | Kyverno policy | ADR + Platform approval |
| Resource limits | Kyverno policy | None |
| Image signature | Kyverno policy | None |
| mTLS | Cilium automatic | None |

---

# 🛤️ **Golden Path (New Service Checklist)**

## Prerequisites

- [ ] GitHub repo created from template
- [ ] Team assigned in GitHub
- [ ] CODEOWNERS configured

## Step-by-Step Checklist

| Étape | Action | Validation | Owner |
|-------|--------|------------|-------|
| 1 | Créer repo depuis template | Structure conforme | Dev |
| 2 | Définir protos dans `contracts-proto` | `buf lint` pass | Dev |
| 3 | Implémenter service | Unit tests > 80% coverage | Dev |
| 4 | Configurer K8s manifests | Kyverno policies pass | Dev |
| 5 | Configurer External-Secret | Secrets résolus | Dev + Platform |
| 6 | Ajouter ServiceMonitor | Metrics visibles Grafana | Dev |
| 7 | Créer HTTPRoute | Trafic routable | Dev |
| 8 | Configurer alerts | Runbook links | Dev + Platform |
| 9 | PR review | Merge → Auto-deploy dev | Dev + Reviewer |
| 10 | Staging validation | E2E tests pass | QA |
| 11 | Prod deployment | Manual approval | Tech Lead |

## Post-Deployment

- [ ] Dashboard créé dans Grafana
- [ ] Runbook documenté
- [ ] On-call routing configuré
- [ ] Load test baseline établi

---

# 📞 **On-Call Structure**

## Team Rotation (5 personnes)

| Rôle | Responsabilité | Rotation | Escalation |
|------|---------------|----------|------------|
| **Primary** | First responder, triage | Weekly | → Secondary (15min) |
| **Secondary** | Escalation, expertise | Weekly | → Incident Commander |
| **Incident Commander** | Coordination si P1 | On-demand | → Management |

## Rotation Schedule

| Week | Primary | Secondary |
|------|---------|-----------|
| 1 | Alice | Bob |
| 2 | Bob | Charlie |
| 3 | Charlie | Diana |
| 4 | Diana | Eve |
| 5 | Eve | Alice |

## On-Call Expectations

| Aspect | Requirement |
|--------|-------------|
| **Response Time (P1)** | < 5 min acknowledge |
| **Response Time (P2)** | < 15 min acknowledge |
| **Response Time (P3)** | < 1 hour acknowledge |
| **Availability** | Reachable 24/7 during rotation |
| **Handoff** | 30 min sync at rotation change |

## Compensation

| Activity | Compensation |
|----------|--------------|
| On-call week | Flat bonus |
| Night incident (22:00-08:00) | Time-off + bonus |
| Weekend incident | 1.5x time-off |

---

# 🚨 **Incident Management**

## Severity Levels

| Severity | Definition | Response | Communication |
|----------|------------|----------|---------------|
| **P1 — Critical** | Service down, data loss risk | Immediate, all hands | Slack + PagerDuty + Status page |
| **P2 — High** | Degraded service, high error rate | Within 30 min | Slack + PagerDuty |
| **P3 — Medium** | Performance degradation | Business hours | Slack |
| **P4 — Low** | Minor issues, no user impact | Next sprint | Ticket |

## Incident Workflow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         INCIDENT WORKFLOW                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  1. DETECT                                                                   │
│     • Alert fires (Prometheus/Grafana)                                      │
│     • User report                                                           │
│     • Synthetic monitoring                                                  │
│                                                                              │
│  2. TRIAGE (Primary on-call)                                                │
│     • Acknowledge alert                                                     │
│     • Assess severity                                                       │
│     • Start incident channel (#inc-YYYYMMDD-short-name)                    │
│                                                                              │
│  3. MITIGATE                                                                │
│     • Apply runbook                                                         │
│     • Rollback if needed                                                    │
│     • Escalate if stuck > 15 min                                           │
│                                                                              │
│  4. RESOLVE                                                                 │
│     • Confirm service restored                                              │
│     • Update status page                                                    │
│     • Close alert                                                           │
│                                                                              │
│  5. POST-MORTEM (within 48h for P1/P2)                                     │
│     • Blameless analysis                                                    │
│     • Root cause identification                                             │
│     • Action items with owners                                              │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Runbook Structure

| Section | Contenu |
|---------|---------|
| **Overview** | Description de l'alerte |
| **Impact** | User impact (High/Medium/Low), Business impact |
| **Prerequisites** | Accès et permissions nécessaires |
| **Diagnosis Steps** | Étapes de diagnostic (dashboards, logs, métriques) |
| **Resolution Steps** | Actions de résolution |
| **Escalation** | Contacts et délais |
| **Related** | Dashboards, alertes liées, incidents passés |

## Post-Mortem Structure

| Section | Contenu |
|---------|---------|
| **Summary** | Résumé en un paragraphe |
| **Timeline** | Chronologie des événements (UTC) |
| **Root Cause** | Cause racine détaillée |
| **Impact** | Users affectés, impact revenue, SLO burn |
| **What Went Well** | Ce qui a bien fonctionné |
| **What Went Wrong** | Ce qui a mal fonctionné |
| **Action Items** | Actions avec owner et due date |
| **Lessons Learned** | Leçons apprises |

---

# 📦 **Service Templates**

## Python Service Template — Structure

| Répertoire | Contenu |
|------------|---------|
| `src/app/` | Code applicatif FastAPI |
| `src/app/api/` | Routes et dependencies |
| `src/app/domain/` | Entities et services métier |
| `src/app/infrastructure/` | Database, Kafka, Cache |
| `tests/` | Unit et integration tests |
| `k8s/base/` | Manifests Kubernetes de base |
| `k8s/overlays/` | Overlays par environnement (dev, staging, prod) |
| `migrations/` | Alembic migrations |

## Kubernetes Manifests Inclus

| Fichier | Rôle |
|---------|------|
| `deployment.yaml` | Définition du Deployment |
| `service.yaml` | Exposition interne (ClusterIP) |
| `configmap.yaml` | Configuration non-sensible |
| `hpa.yaml` | Horizontal Pod Autoscaler |
| `pdb.yaml` | Pod Disruption Budget |
| `servicemonitor.yaml` | Prometheus scraping |
| `kustomization.yaml` | Kustomize base |

## Deployment Configuration

| Setting | Valeur | Raison |
|---------|--------|--------|
| **Replicas** | 2 (min) | Haute disponibilité |
| **Security Context** | runAsNonRoot: true, runAsUser: 1000 | Sécurité |
| **Filesystem** | readOnlyRootFilesystem: true | Sécurité |
| **Capabilities** | drop: ALL | Principle of least privilege |
| **Resources requests** | CPU: 100m, Memory: 256Mi | Scheduling |
| **Resources limits** | CPU: 500m, Memory: 512Mi | Protection contre les fuites |

## Health Probes

| Probe | Path | Delay | Period |
|-------|------|-------|--------|
| **Liveness** | `/health/live` | 10s | 10s |
| **Readiness** | `/health/ready` | 5s | 5s |

## Horizontal Pod Autoscaler

| Métrique | Target | Min/Max Replicas |
|----------|--------|------------------|
| **CPU** | 70% average utilization | 2 / 10 |
| **Memory** | 80% average utilization | 2 / 10 |

## ServiceMonitor Configuration

| Setting | Valeur |
|---------|--------|
| **Port** | http (8080) |
| **Path** | /metrics |
| **Interval** | 30s |
| **Selector** | matchLabels: app: ${SERVICE_NAME} |

---

## SLI/SLO/Error Budgets

| Service | SLI | SLO | Error Budget | Burn Rate Alert |
|---------|-----|-----|--------------|-----------------|
| **svc-ledger** | Availability | 99.9% | 43 min/mois | 14.4x = 1h alert |
| **svc-ledger** | Latency P99 | < 200ms | N/A | P99 > 200ms for 5min |
| **svc-wallet** | Availability | 99.9% | 43 min/mois | 14.4x = 1h alert |
| **Platform (Flux, Prometheus)** | Availability | 99.5% | 3.6h/mois | 6x = 2h alert |

## Error Budget Policy

| Budget Consumed | Action |
|-----------------|--------|
| < 50% | Normal development velocity |
| 50-75% | Increased testing, careful deployments |
| 75-90% | Feature freeze, reliability focus |
| > 90% | Emergency mode, only critical fixes |

---

*Document maintenu par : Platform Team*  
*Dernière mise à jour : Janvier 2026*
