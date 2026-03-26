# ⚡ **Resilience & Disaster Recovery Guide**
## *LOCAL-PLUS Backup, Recovery & Business Continuity*

---

> **Retour vers** : [Architecture Overview](../EntrepriseArchitecture.md)  
> **Voir aussi** : [Testing Strategy](../testing/TESTING-STRATEGY.md) — Tests de performance et chaos

---

# 📋 **Table of Contents**

1. [Failure Modes](#failure-modes)
2. [Backup Strategy](#backup-strategy)
3. [Automated Recovery](#automated-recovery)
4. [Chaos Engineering](#chaos-engineering)
5. [Disaster Recovery](#disaster-recovery)
6. [Business Continuity](#business-continuity)

---

# 🔥 **Failure Modes**

## Failure Matrix

| Failure | Detection | Recovery | RTO | Impact |
|---------|-----------|----------|-----|--------|
| **Pod crash** | Liveness probe | K8s restart automatique | < 30s | None (replicas) |
| **Node failure** | Node NotReady | Pod reschedule automatique | < 2min | Minor latency spike |
| **AZ failure** | Multi-AZ detect | Traffic shift automatique | < 5min | Reduced capacity |
| **DB primary failure** | Aiven health | Failover automatique | < 5min | Brief connection errors |
| **Kafka broker failure** | Aiven health | Rebalance automatique | < 2min | Brief producer retries |
| **Cache failure** | Health check | Fallback to DB automatique | < 1min | Increased latency |
| **Full region failure** | Health checks | DR procedure | 4h (target) | Extended outage |

## Blast Radius Analysis

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         BLAST RADIUS ANALYSIS                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  SINGLE POD FAILURE                                                         │
│  └── Impact: None (other replicas serve traffic)                           │
│  └── Recovery: Automatique via Kubernetes                                  │
│                                                                              │
│  SINGLE NODE FAILURE                                                        │
│  └── Impact: 10-20% capacity loss temporarily                              │
│  └── Recovery: Automatique via pod anti-affinity + reschedule              │
│                                                                              │
│  SINGLE AZ FAILURE                                                          │
│  └── Impact: 33% capacity loss                                             │
│  └── Recovery: Automatique via Multi-AZ + ALB health checks                │
│                                                                              │
│  DATABASE PRIMARY FAILURE                                                   │
│  └── Impact: Write unavailability ~5 min                                   │
│  └── Recovery: Automatique via Aiven failover                              │
│                                                                              │
│  FULL REGION FAILURE                                                        │
│  └── Impact: Complete service unavailability                               │
│  └── Recovery: Semi-automatique (DR procedure)                             │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

# 💾 **Backup Strategy**

## Backup Matrix

| Data | Method | Frequency | Retention | Location | Encryption |
|------|--------|-----------|-----------|----------|------------|
| **PostgreSQL** | Aiven automated | Hourly | 7 jours | Aiven (cross-AZ) | AES-256 |
| **PostgreSQL PITR** | Aiven WAL | Continuous | 24h | Aiven | AES-256 |
| **Kafka** | Topic retention | N/A | 7 jours | Aiven | AES-256 |
| **Valkey** | RDB + AOF | Continuous | 24h | Aiven | AES-256 |
| **Terraform state** | S3 versioning | Every apply | 90 jours | S3 bucket | AES-256 |
| **Git repos** | GitHub | Every push | Infini | GitHub | At-rest |
| **Secrets (Vault)** | Integrated storage | Continuous | 30 jours | Vault HA | Transit |

## Backup Verification — Automatisée

| Check | Frequency | Automation | Alert si échec |
|-------|-----------|------------|----------------|
| PostgreSQL restore test | Weekly | Job K8s scheduled | P2 |
| Terraform state backup | Daily | CI pipeline | P3 |
| Vault backup verification | Weekly | Job K8s scheduled | P2 |
| Git clone verification | Monthly | GitHub Actions | P4 |

## Backup Monitoring

| Metric | Alert Threshold | Severity |
|--------|-----------------|----------|
| Last backup age | > 2 hours | P2 |
| Backup size anomaly | > 50% change | P3 |
| Backup job failure | Any failure | P2 |
| PITR lag | > 1 hour | P2 |

---

# 🤖 **Automated Recovery**

## Principe : Self-Healing Infrastructure

> **Objectif :** Minimiser l'intervention humaine. Le système doit se réparer automatiquement.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       AUTOMATED RECOVERY LAYERS                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  LAYER 1: APPLICATION (Kubernetes)                                          │
│  ├── Liveness probes → Restart automatique                                 │
│  ├── Readiness probes → Traffic routing                                    │
│  ├── HPA → Scale automatique                                               │
│  └── PDB → Protection pendant maintenance                                  │
│                                                                              │
│  LAYER 2: DATABASE (Aiven)                                                  │
│  ├── Health monitoring → Failover automatique                              │
│  ├── Connection pooling (PgBouncer) → Retry transparent                    │
│  └── Read replicas → Load distribution                                     │
│                                                                              │
│  LAYER 3: MESSAGING (Kafka)                                                 │
│  ├── Broker failure → Partition rebalance automatique                      │
│  ├── Consumer failure → Rebalance consumer group                           │
│  └── Producer retry → Idempotent delivery                                  │
│                                                                              │
│  LAYER 4: CACHE (Valkey)                                                    │
│  ├── Cache miss → Fallback DB automatique (cache-aside pattern)           │
│  ├── Node failure → Cluster failover                                       │
│  └── TTL expiration → Lazy refresh                                         │
│                                                                              │
│  LAYER 5: NETWORKING (Cloudflare + Cilium)                                  │
│  ├── Origin failure → Health check + failover                              │
│  ├── DDoS → Auto-mitigation                                                │
│  └── mTLS → Automatic certificate rotation                                 │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Recovery automatique par composant

| Composant | Failure | Recovery Mechanism | Temps | Intervention |
|-----------|---------|-------------------|-------|--------------|
| **Pod** | Crash | Kubernetes restart | < 30s | Aucune |
| **Pod** | OOM | Kubernetes restart + alert | < 30s | Investigation |
| **Deployment** | Bad deploy | Flux rollback auto (si configuré) | < 2min | Aucune |
| **DB Primary** | Failure | Aiven automatic failover | < 5min | Aucune |
| **DB Connection** | Pool exhausted | PgBouncer retry + scale | < 1min | Aucune |
| **Kafka Consumer** | Lag > threshold | KEDA auto-scale | < 2min | Aucune |
| **Cache** | Node down | Cluster failover + fallback DB | < 1min | Aucune |
| **Certificate** | Expiring | Cert-manager auto-renew | N/A | Aucune |

---

# 🔬 **Chaos Engineering**

## Philosophie

> **"Nous ne testons pas si le système tombe, mais si le système se relève."**

## Chaos Testing Framework

| Outil | Usage | Intégration |
|-------|-------|-------------|
| **Chaos Mesh** | Injection de pannes Kubernetes | CRD natif K8s |
| **Litmus** | Alternative open-source | Scenarios prédéfinis |
| **Gremlin** | Enterprise (si budget) | SaaS, plus de features |

## Experiments Automatisés

| Experiment | Target | Fréquence | Validation |
|------------|--------|-----------|------------|
| **Pod Kill** | Random pod in service | Daily (staging) | Service continues responding |
| **Network Latency** | Inter-service +100ms | Weekly | SLO latency maintained |
| **Node Drain** | Random node | Weekly | Pods rescheduled, no downtime |
| **DB Failover** | Force primary switch | Monthly | Connections recover < 5min |
| **Cache Flush** | Valkey flush | Weekly | Fallback to DB works |
| **AZ Failure Simulation** | Cordon all nodes in 1 AZ | Quarterly | Traffic shifts to other AZs |

## Chaos Test Pipeline

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      CHAOS TESTING PIPELINE                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  1. PRE-CHECK                                                               │
│     • Verify system healthy (all green)                                     │
│     • Baseline metrics recorded                                             │
│     • Alerting team notified (staging)                                      │
│                                                                              │
│  2. INJECT CHAOS                                                            │
│     • Apply Chaos Mesh experiment                                           │
│     • Duration: 5-15 minutes                                                │
│                                                                              │
│  3. OBSERVE                                                                 │
│     • Monitor SLIs (error rate, latency)                                    │
│     • Check recovery mechanisms activate                                    │
│     • Record recovery time                                                  │
│                                                                              │
│  4. VALIDATE                                                                │
│     • SLO maintained? ✅ / ❌                                               │
│     • Recovery time within RTO? ✅ / ❌                                     │
│     • No data loss? ✅ / ❌                                                 │
│                                                                              │
│  5. CLEANUP                                                                 │
│     • Remove chaos experiment                                               │
│     • Verify system back to baseline                                        │
│     • Generate report                                                       │
│                                                                              │
│  6. ACTION                                                                  │
│     • Si échec → Ticket pour fix                                           │
│     • Si succès → Increase chaos scope                                     │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Game Days

| Activité | Fréquence | Participants | Scope |
|----------|-----------|--------------|-------|
| **Chaos Friday** | Weekly | On-call | Staging, experiments simples |
| **Game Day** | Monthly | Full team | Staging, multi-failure scenarios |
| **DR Drill** | Quarterly | Team + Management | Staging, full DR simulation |
| **Production Chaos** | Annually | Team + SRE | Prod (maintenance window) |

---

# 🏥 **Disaster Recovery**

## DR Scenarios

| Scenario | Recovery | Automation Level |
|----------|----------|------------------|
| Single AZ failure | Automatique (multi-AZ) | 100% |
| Region failure | Semi-automatique (IaC + GitOps) | 80% |
| Data corruption | PITR restore | 60% |
| Ransomware | Immutable backups restore | 50% |

## DR Automation — Infrastructure as Code

> **Principe :** Toute l'infrastructure est reproductible via Terraform + Flux.

| Composant | Reproductibilité | Temps estimé |
|-----------|------------------|--------------|
| **EKS Cluster** | Terraform apply | ~30 min |
| **Platform tools** | Flux reconcile | ~15 min |
| **Applications** | Flux reconcile | ~10 min |
| **Database** | Aiven restore from backup | ~1-2h |
| **DNS cutover** | Cloudflare API / Terraform | ~5 min |

## DR Runbook — Region Failure

| Phase | Durée | Actions | Automation |
|-------|-------|---------|------------|
| **1. Detection** | 15 min | Confirmer failure, déclarer DR | Alerting automatique |
| **2. Infrastructure** | 1-2h | Terraform apply DR region | Semi-auto (approval required) |
| **3. Data** | 1-2h | Aiven restore, verify integrity | Semi-auto (Aiven console) |
| **4. Applications** | 30 min | Flux reconcile | Automatique |
| **5. Traffic** | 15 min | Cloudflare DNS update | Semi-auto (Terraform) |
| **6. Validation** | 30 min | E2E tests, verify SLIs | Automatique (CI) |

**RTO Total : 4 heures**

## DR Test Schedule

| Test | Fréquence | Scope | Duration |
|------|-----------|-------|----------|
| Backup restore | Weekly | PostgreSQL single table | 30 min |
| Failover test | Monthly | Database failover | 1 hour |
| DR drill | Quarterly | Full DR simulation (staging) | 4 hours |
| Full DR test | Annually | Production DR (maintenance) | 8 hours |

---

# 📊 **Business Continuity**

## RPO/RTO Summary

| Scenario | RPO | RTO | Data Loss Risk | Automation |
|----------|-----|-----|----------------|------------|
| Pod failure | 0 | < 30s | None | 100% auto |
| Node failure | 0 | < 2min | None | 100% auto |
| AZ failure | 0 | < 5min | None | 100% auto |
| DB failover | 0 (sync) | < 5min | None | 100% auto |
| Region failure | 1 hour | 4 hours | Up to 1 hour | 80% auto |

## Communication Plan

| Audience | Channel | Frequency | Owner |
|----------|---------|-----------|-------|
| Engineering | Slack #incidents | Real-time | On-call |
| Management | Email + Slack | Every 30 min | Incident Commander |
| Customers | Status page | Every 15 min | Communications |
| Partners | Email | Major updates | Account Management |

## Status Page

| Status | Definition |
|--------|------------|
| **Operational** | All systems normal |
| **Degraded** | Partial impact (increased latency) |
| **Partial Outage** | Some features unavailable |
| **Major Outage** | Service unavailable |

---

## Recovery Validation — Automatisée

> Ces checks sont exécutés automatiquement après chaque recovery.

### Health Check Pipeline

| Check | Method | Failure Action |
|-------|--------|----------------|
| All pods running | kubectl health check | Alert P1 |
| Metrics flowing | Prometheus query | Alert P2 |
| Logs flowing | Loki query | Alert P2 |
| Traces flowing | Tempo query | Alert P3 |
| E2E critical path | Automated tests | Alert P1 |
| Error rate normal | SLI check | Alert P2 |

### Database Validation

| Check | Method | Failure Action |
|-------|--------|----------------|
| Data integrity | Checksum validation | Alert P1 |
| Transaction count | Count comparison | Alert P2 |
| FK constraints | DB validation | Alert P2 |
| Read/Write test | Smoke test | Alert P1 |

---

> **Voir aussi :** [Testing Strategy](../testing/TESTING-STRATEGY.md) pour les tests de performance, load testing, et chaos engineering détaillés.

---

*Document maintenu par : Platform Team + SRE*  
*Dernière mise à jour : Janvier 2026*
