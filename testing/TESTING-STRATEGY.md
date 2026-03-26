# 🧪 **Testing Strategy**
## *LOCAL-PLUS Quality Engineering*

---

> **Retour vers** : [Architecture Overview](../EntrepriseArchitecture.md)  
> **Voir aussi** : [DR Guide](../resilience/DR-GUIDE.md) — Chaos Engineering

---

# 📋 **Table of Contents**

1. [Test Pyramid Philosophy](#test-pyramid-philosophy)
2. [Platform Testing](#platform-testing)
3. [Application Testing](#application-testing)
4. [Integration & Contract Testing](#integration--contract-testing)
5. [Performance Testing](#performance-testing)
6. [Chaos Engineering](#chaos-engineering)
7. [Compliance Testing](#compliance-testing)
8. [TNR (Tests de Non-Régression)](#tnr-tests-de-non-régression)

---

# 🔺 **Test Pyramid Philosophy**

## Le concept

```
                           ╱╲
                          ╱  ╲
                         ╱ E2E╲                 ← Peu, coûteux, lents
                        ╱──────╲                   Validation business
                       ╱        ╲
                      ╱ Contract ╲              ← Vérifie les interfaces
                     ╱────────────╲                Entre services
                    ╱              ╲
                   ╱  Integration   ╲           ← DB, Kafka, Cache réels
                  ╱──────────────────╲             Testcontainers
                 ╱                    ╲
                ╱     Unit Tests       ╲        ← Beaucoup, rapides, isolés
               ╱────────────────────────╲          Logique métier
              ╱                          ╲
             ╱      Static Analysis       ╲     ← Linting, type checking
            ╱──────────────────────────────╲       Avant même d'exécuter
```

## Principes clés

| Principe | Description |
|----------|-------------|
| **Plus de tests en bas** | Unit tests = 70%, Integration = 20%, E2E = 10% |
| **Rapidité en bas** | Unit tests < 1s, Integration < 30s, E2E < 5min |
| **Isolation en bas** | Unit = mocks, Integration = containers, E2E = real env |
| **Coût croissant** | Plus on monte, plus c'est cher à maintenir |
| **Confiance croissante** | Plus on monte, plus on valide le "vrai" système |

## Application à Local-Plus

| Layer | Type de test | Cible | Fréquence |
|-------|--------------|-------|-----------|
| **Infrastructure** | Terraform tests, Policy checks | IaC modules | PR |
| **Platform** | Smoke tests, Policy audit | Kubernetes, Flux | Post-deploy |
| **Application** | Unit, Integration, Contract | Services Python/Go | PR |
| **System** | E2E, Performance, Chaos | Full stack | Nightly/Weekly |

---

# 🏗️ **Platform Testing**

## Test Pyramid pour l'Infrastructure

```
                        ╱╲
                       ╱  ╲
                      ╱ E2E╲                ← Déploiement réel (staging)
                     ╱──────╲                  Nightly
                    ╱        ╲
                   ╱Integration╲            ← Terratest (crée vraies ressources)
                  ╱────────────╲               Nightly, temps limité
                 ╱              ╲
                ╱   Unit Tests   ╲          ← terraform test (plan-based)
               ╱──────────────────╲            PR, rapide
              ╱                    ╲
             ╱   Static Analysis    ╲       ← tflint, tfsec, checkov
            ╱────────────────────────╲         Pre-commit, PR
```

## Terraform Testing

| Type | Outil | Quand | Ce que ça vérifie | Bloquant |
|------|-------|-------|-------------------|----------|
| **Format** | `terraform fmt` | Pre-commit | Code formatté | Oui |
| **Lint** | `tflint` | Pre-commit | Best practices HCL | Oui |
| **Security** | `tfsec`, `checkov` | PR | Vulnérabilités, misconfigs | Oui |
| **Compliance** | `terraform-compliance`, `conftest` | PR | Policies internes | Oui |
| **Unit** | `terraform test` (native) | PR | Logique des modules | Oui |
| **Integration** | `terratest` | Nightly | Ressources créées correctement | Non |
| **Drift** | `terraform plan` (scheduled) | Daily | Écart config vs réalité | Alerte |

## Policy as Code — Ce qu'on vérifie

| Policy | Description | Outil |
|--------|-------------|-------|
| **S3 encryption** | Tous les buckets doivent avoir encryption | OPA/Conftest |
| **Public access** | Aucune ressource publique sauf explicite | tfsec |
| **Tagging** | Tags obligatoires (env, owner, cost-center) | terraform-compliance |
| **Naming** | Convention de nommage respectée | Custom OPA |
| **Networking** | Pas d'IGW sur VPC privé | Checkov |

## Kubernetes Testing

| Type | Outil | Quand | Ce que ça vérifie |
|------|-------|-------|-------------------|
| **Manifest validation** | `kubectl --dry-run`, `kubeconform` | PR | YAML valide, schema correct |
| **Policy check** | Kyverno CLI | PR | Policies passent |
| **Helm lint** | `helm lint`, `helm template` | PR | Charts valides |
| **Smoke test** | Flux reconcile + health check | Post-deploy | App déployée et healthy |

---

# 📱 **Application Testing**

## Test Pyramid pour les Services

```
                        ╱╲
                       ╱  ╲
                      ╱ E2E╲                ← Playwright, staging
                     ╱──────╲                  Post-merge
                    ╱        ╲
                   ╱ Contract ╲             ← Pact, gRPC testing
                  ╱────────────╲               PR
                 ╱              ╲
                ╱  Integration   ╲          ← Testcontainers
               ╱──────────────────╲            PR
              ╱                    ╲
             ╱     Unit Tests       ╲       ← pytest, mocks
            ╱────────────────────────╲         Pre-commit, PR
           ╱                          ╲
          ╱      Static Analysis       ╲    ← ruff, mypy, bandit
         ╱──────────────────────────────╲      Pre-commit
```

## Unit Tests

| Aspect | Approche |
|--------|----------|
| **Cible** | Domain logic, Use cases, Utilities |
| **Isolation** | Mocks pour DB, Kafka, Cache, HTTP clients |
| **Coverage** | Minimum 80% sur le domain layer |
| **Vitesse** | < 1 seconde par test |
| **Framework** | pytest (Python), go test (Go) |

### Ce qu'on teste en Unit

| Composant | Tests |
|-----------|-------|
| **Domain entities** | Validation, business rules, state transitions |
| **Use cases** | Orchestration logic (avec mocks) |
| **Value objects** | Immutabilité, égalité |
| **Utilities** | Pure functions, helpers |

### Ce qu'on NE teste PAS en Unit

| Composant | Pourquoi |
|-----------|----------|
| **Repositories** | Nécessite vraie DB → Integration |
| **Kafka producers** | Nécessite vrai broker → Integration |
| **HTTP clients** | Interactions réelles → Contract |
| **Controllers/Routes** | Wiring → Integration ou E2E |

## Integration Tests

| Aspect | Approche |
|--------|----------|
| **Cible** | Repositories, Message producers, Cache clients |
| **Infrastructure** | Testcontainers (PostgreSQL, Kafka, Redis) |
| **Isolation** | Chaque test a sa propre DB/topic |
| **Vitesse** | < 30 secondes par test |
| **Framework** | pytest + testcontainers |

### Ce qu'on vérifie

| Composant | Vérifications |
|-----------|---------------|
| **PostgreSQL Repository** | CRUD fonctionne, transactions, contraintes FK |
| **Kafka Producer** | Messages publiés, sérialisation correcte |
| **Kafka Consumer** | Messages consommés, idempotence |
| **Cache Client** | Set/Get/Delete, TTL, invalidation |
| **Outbox Pattern** | Transaction + event atomiques |

---

# 🤝 **Integration & Contract Testing**

## Pourquoi Contract Testing ?

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    LE PROBLÈME SANS CONTRACT TESTING                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  svc-ledger                              svc-wallet                         │
│  ┌──────────┐                            ┌──────────┐                       │
│  │ Appelle  │─────── HTTP/gRPC ────────►│ Répond   │                       │
│  │ Wallet   │                            │          │                       │
│  └──────────┘                            └──────────┘                       │
│                                                                              │
│  ❌ Wallet change son API                                                   │
│  ❌ Ledger ne le sait pas                                                   │
│  ❌ Découvert en production = 💥                                            │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                    LA SOLUTION : CONTRACT TESTING                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  svc-ledger                              svc-wallet                         │
│  ┌──────────┐     ┌──────────────┐       ┌──────────┐                       │
│  │ Consumer │────►│   CONTRACT   │◄──────│ Provider │                       │
│  │ Tests    │     │  (Pact file) │       │ Tests    │                       │
│  └──────────┘     └──────────────┘       └──────────┘                       │
│                                                                              │
│  ✅ Ledger déclare ce qu'il attend                                          │
│  ✅ Wallet vérifie qu'il respecte le contrat                                │
│  ✅ CI bloque si contrat cassé                                              │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Contract Testing Approach

| Aspect | Approche |
|--------|----------|
| **Outil REST** | Pact |
| **Outil gRPC** | buf breaking, grpc-testing |
| **Consumer-driven** | Le consumer définit ses besoins |
| **Provider verification** | Le provider vérifie qu'il satisfait |
| **Broker** | Pact Broker (ou Pactflow) pour centraliser |

## Ce qu'on vérifie en Contract

| Type | Vérifications |
|------|---------------|
| **Request format** | Path, method, headers, body schema |
| **Response format** | Status code, headers, body schema |
| **Error cases** | 4xx/5xx responses, error messages |
| **Breaking changes** | Champs supprimés, types changés |

---

# ⚡ **Performance Testing**

## Types de tests de performance

| Type | Objectif | VUs | Durée | Fréquence |
|------|----------|-----|-------|-----------|
| **Smoke** | Vérifier que ça marche | 1-5 | 1 min | Post-deploy |
| **Load** | Charge normale | 50-100 | 10 min | Nightly |
| **Stress** | Trouver le breaking point | Ramping 500+ | 15 min | Weekly |
| **Soak** | Endurance, memory leaks | 50 | 4 hours | Weekly |
| **Spike** | Pics soudains | 10→200→10 | 5 min | Monthly |

## Outil : k6

| Aspect | Choix |
|--------|-------|
| **Outil** | k6 (Grafana) |
| **Scripting** | JavaScript |
| **Reporting** | Grafana Cloud ou self-hosted |
| **CI Integration** | GitHub Actions |

## Thresholds (Critères de succès)

| Métrique | Target | Alerte | Bloquant |
|----------|--------|--------|----------|
| **Latency P50** | < 50ms | > 100ms | Non |
| **Latency P95** | < 100ms | > 200ms | Oui |
| **Latency P99** | < 200ms | > 500ms | Oui |
| **Error Rate** | < 0.1% | > 1% | Oui |
| **Throughput** | > 500 TPS | < 400 TPS | Non |

## Scénarios de test par service

| Service | Scenario | VUs cible | Throughput cible |
|---------|----------|-----------|------------------|
| **svc-ledger** | Create transaction | 100 | 500 TPS |
| **svc-ledger** | Get balance | 200 | 1000 TPS |
| **svc-wallet** | Update balance | 100 | 500 TPS |
| **svc-merchant** | List transactions | 50 | 200 TPS |

## Performance Testing Pipeline

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    PERFORMANCE TESTING PIPELINE                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  1. SMOKE TEST (Post-deploy)                                                │
│     • 1-5 VUs, 1 minute                                                     │
│     • Vérifie que le service répond                                        │
│     • Gate pour continuer                                                   │
│                                                                              │
│  2. LOAD TEST (Nightly)                                                     │
│     • 50-100 VUs, 10 minutes                                               │
│     • Vérifie performance normale                                          │
│     • Compare avec baseline                                                │
│                                                                              │
│  3. STRESS TEST (Weekly)                                                    │
│     • Ramping jusqu'à failure                                              │
│     • Identifie le breaking point                                          │
│     • Documente les limites                                                │
│                                                                              │
│  4. SOAK TEST (Weekly)                                                      │
│     • 50 VUs, 4 heures                                                     │
│     • Détecte memory leaks                                                 │
│     • Vérifie stabilité long-terme                                         │
│                                                                              │
│  5. REPORT                                                                  │
│     • Dashboard Grafana                                                    │
│     • Trend analysis                                                       │
│     • Alertes si régression                                                │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

# 💥 **Chaos Engineering**

> **Détails complets** : voir [DR Guide — Chaos Engineering](../resilience/DR-GUIDE.md#chaos-engineering)

## Philosophie

| Principe | Description |
|----------|-------------|
| **Build confidence** | Prouver que le système résiste aux pannes |
| **Proactive** | Casser avant que ça casse en prod |
| **Controlled** | Experiments planifiés, scope limité |
| **Observable** | Mesurer l'impact, recovery time |

## Experiments par layer

| Layer | Experiment | Outil | Fréquence |
|-------|------------|-------|-----------|
| **Pod** | Kill random pod | Chaos Mesh | Daily (staging) |
| **Node** | Drain node | Chaos Mesh | Weekly |
| **Network** | Add latency 100ms | Chaos Mesh | Weekly |
| **Network** | Partition (isoler un service) | Chaos Mesh | Monthly |
| **Database** | Force failover | Aiven console | Monthly |
| **Cache** | Flush all | Chaos Mesh | Weekly |
| **AZ** | Cordon all nodes in 1 AZ | kubectl | Quarterly |

## Validation

| Experiment | Expected Behavior | Success Criteria |
|------------|-------------------|------------------|
| Pod kill | Traffic shifts to other pods | Error rate < 1%, recovery < 30s |
| Node drain | Pods rescheduled | No downtime |
| Network latency | Degraded but functional | SLO latency maintained |
| DB failover | Brief connection errors | Recovery < 5min |
| Cache flush | Fallback to DB | Increased latency, no errors |

---

# ✅ **Compliance Testing**

## Tests par standard

| Standard | Test | Ce qu'on vérifie | Outil |
|----------|------|------------------|-------|
| **GDPR** | PII in logs | Pas d'email, user_id, IP en clair | Log audit script |
| **GDPR** | Data retention | Logs < 30 jours | Loki config check |
| **GDPR** | Right to delete | API de suppression fonctionne | E2E test |
| **PCI-DSS** | Encryption in transit | mTLS enforced | Cilium policy audit |
| **PCI-DSS** | Encryption at rest | KMS enabled | AWS Config rules |
| **PCI-DSS** | No PAN storage | Pas de numéro de carte | Code scan + log audit |
| **SOC2** | Audit logs | CloudTrail + K8s audit | AWS Config |
| **SOC2** | Access control | RBAC enforced | Kyverno reports |
| **SOC2** | Change management | PR required, reviews | GitHub settings |

## Automatisation

| Check | Fréquence | Bloquant |
|-------|-----------|----------|
| Log audit (PII) | Nightly | Alerte P2 |
| Policy reports (Kyverno) | Continuous | Dashboard |
| AWS Config rules | Continuous | Alerte P2 |
| Encryption verification | Weekly | Alerte P1 si échec |

---

# 🔄 **TNR (Tests de Non-Régression)**

## Catégories

| Catégorie | Ce qu'on vérifie | Fréquence |
|-----------|------------------|-----------|
| **Critical Paths** | Flux métier essentiels | Nightly |
| **Golden Master** | Réponses API n'ont pas changé | Nightly |
| **Backward Compatibility** | Anciennes versions clients fonctionnent | Pre-release |
| **Data Migration** | Migrations n'ont pas cassé les données | Post-migration |

## Critical Paths

| Path | Étapes | SLA |
|------|--------|-----|
| **Earn flow** | Transaction → Balance update → Event → Notification | < 5s end-to-end |
| **Burn flow** | Transaction → Balance check → Deduction → Event | < 5s end-to-end |
| **Balance query** | Request → Cache/DB → Response | < 100ms |
| **Merchant onboarding** | Registration → Validation → Activation | < 30s |

## E2E Testing

| Aspect | Approche |
|--------|----------|
| **Outil** | Playwright |
| **Environment** | Staging (miroir de prod) |
| **Data** | Fixtures dédiées, cleanup après |
| **Fréquence** | Post-merge staging, pre-release prod |
| **Ownership** | QA Team |

## Pipeline TNR

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         TNR PIPELINE (Nightly)                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  00:00 ─► SETUP                                                             │
│           • Fresh staging environment                                       │
│           • Load test fixtures                                              │
│                                                                              │
│  00:15 ─► CRITICAL PATH TESTS                                               │
│           • Earn/Burn flows                                                 │
│           • All major user journeys                                         │
│                                                                              │
│  01:00 ─► PERFORMANCE TESTS                                                 │
│           • Load test (10 min)                                              │
│           • Compare with baseline                                           │
│                                                                              │
│  01:30 ─► COMPLIANCE TESTS                                                  │
│           • Log audit (PII check)                                           │
│           • Policy verification                                             │
│                                                                              │
│  02:00 ─► REPORT                                                            │
│           • Generate report                                                 │
│           • Alert if failures                                               │
│           • Update dashboard                                                │
│                                                                              │
│  02:30 ─► CLEANUP                                                           │
│           • Reset test data                                                 │
│           • Archive logs                                                    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Récapitulatif : Qui teste quoi ?

| Équipe | Responsabilité | Types de tests |
|--------|----------------|----------------|
| **Developers** | Unit, Integration | PR gate |
| **Platform** | Terraform, Kubernetes, Chaos | PR + Nightly |
| **QA** | E2E, TNR, Performance | Nightly + Pre-release |
| **Security** | Compliance, Policy audit | Continuous |

---

*Document maintenu par : QA Team + Platform Team*  
*Dernière mise à jour : Janvier 2026*
