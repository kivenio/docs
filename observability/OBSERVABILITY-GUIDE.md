# Observability Guide
## Kiven Platform -- Monitoring, Logging, Tracing & APM

---

> **Back to**: [Architecture Overview](../EntrepriseArchitecture.md) | **See also**: [OTel Conventions](./OTEL-CONVENTIONS.md)

---

## Table of Contents

1. [Stack Overview](#stack-overview)
2. [Telemetry Pipeline](#telemetry-pipeline)
3. [Metrics (Prometheus)](#metrics-prometheus)
4. [Logs (Loki)](#logs-loki)
5. [Traces (Tempo)](#traces-tempo)
6. [APM (Application Performance Monitoring)](#apm-application-performance-monitoring)
7. [Cardinality Management](#cardinality-management)
8. [SLI/SLO/Error Budgets](#slisloerror-budgets)
9. [Alerting Strategy](#alerting-strategy)
10. [Dashboards & Visualizations](#dashboards--visualizations)

> For OTel-specific conventions (span naming, attributes, Collector deployment, exporter helper, SDK usage), see [OTEL-CONVENTIONS.md](./OTEL-CONVENTIONS.md).

---

## Stack Overview

### Self-Hosted Stack

| Composant | Outil | Coût | Retention |
|-----------|-------|------|-----------|
| **Metrics** | Prometheus | 0€ (self-hosted) | 15 jours local |
| **Metrics long-term** | Prometheus avec Remote Write → S3 | ~5€/mois S3 | 1 an |
| **Logs** | Loki | 0€ (self-hosted) | 30 jours (GDPR) |
| **Traces** | Tempo | 0€ (self-hosted) | 7 jours |
| **Dashboards** | Grafana | 0€ (self-hosted) | N/A |
| **Fallback logs** | CloudWatch Logs | Tier gratuit 5GB | 7 jours |

**Coût estimé : < 50€/mois** (principalement stockage S3)

### Note sur le stockage long-terme

Pour conserver les métriques au-delà de 15 jours :

| Option | Description | Complexité |
|--------|-------------|------------|
| **Remote Write vers S3** | Prometheus écrit directement vers un backend compatible S3 | Simple |
| **Grafana Mimir** | Solution CNCF pour le stockage long-terme, scalable | Moyen |
| **Victoria Metrics** | Alternative performante, compatible Prometheus | Moyen |

> **Choix Local-Plus :** Remote Write vers S3 via Grafana Mimir (ou Victoria Metrics) — pas besoin de composants additionnels complexes.

---

## Telemetry Pipeline

```
┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐     ┌──────────────┐
│  Kiven Services  │     │  OTel Agent      │     │  OTel Gateway    │     │  Backends    │
│                  │     │  (DaemonSet)     │     │  (Deployment)    │     │              │
│  • Go svc-*      │────►│  • Receive OTLP  │────►│  • Batch         │────►│  Prometheus  │
│  • kiven-agent   │     │  • Forward       │     │  • Tail sample   │     │  Loki        │
│  • dashboard     │     │  • No processing │     │  • Scrub PII     │     │  Tempo       │
│                  │     │                  │     │  • Persistent Q  │     │  Grafana     │
│  Instrumented    │     │  Lightweight     │     │  • Export         │     │              │
│  via kiven-go-   │     │  ~50MB per node  │     │  • 2-3 replicas  │     │              │
│  sdk/telemetry   │     │                  │     │                  │     │              │
└──────────────────┘     └──────────────────┘     └──────────────────┘     └──────────────┘
                                                         │
                                                         │ GDPR Scrubbing
                                                         ▼
                                                  ┌──────────────────┐
                                                  │ Removed:         │
                                                  │ • user_id        │
                                                  │ • user.email     │
                                                  │ • client_ip      │
                                                  │ • SQL params     │
                                                  └──────────────────┘
```

> See [OTEL-CONVENTIONS.md](./OTEL-CONVENTIONS.md) for Collector config details, exporter helper pattern, and persistent queue setup.

## OTel Collector — Rôle

| Composant | Rôle | Exemples |
|-----------|------|----------|
| **Receivers** | Réceptionne les données de télémétrie | OTLP (gRPC/HTTP), Prometheus scrape |
| **Processors** | Transforme, filtre, enrichit les données | Batch, Memory limiter, Attribute deletion (PII), Sampling |
| **Exporters** | Envoie vers les backends | Prometheus, Loki, Tempo |

## GDPR Compliance — Données supprimées

| Donnée | Action | Raison |
|--------|--------|--------|
| `user.id` | Supprimé | PII |
| `user.email` | Supprimé | PII |
| `http.client_ip` | Hashé | Anonymisation |
| `*_bucket` haute cardinalité | Filtré | Performance |

---

# 📈 **Metrics (Prometheus)**

## Comment Prometheus collecte les métriques

Prometheus utilise un modèle **pull** : il va chercher les métriques sur chaque cible.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                  PROMETHEUS — MODÈLE DE COLLECTE                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│                        PROMETHEUS                                            │
│                       (scraping)                                             │
│                            │                                                 │
│         ┌──────────────────┼──────────────────┐                             │
│         │                  │                  │                             │
│         ▼                  ▼                  ▼                             │
│   ┌──────────┐      ┌──────────┐      ┌──────────┐                         │
│   │ Pod A    │      │ Pod B    │      │ Pod C    │                         │
│   │          │      │          │      │          │                         │
│   │ :8080    │      │ :8080    │      │ :9090    │                         │
│   │ /metrics │      │ /metrics │      │ /metrics │                         │
│   └──────────┘      └──────────┘      └──────────┘                         │
│                                                                              │
│   Prometheus fait GET http://pod:port/metrics toutes les 30s               │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Découverte des cibles — ServiceMonitor (Prometheus Operator)

Le **Prometheus Operator** utilise des **Custom Resources** pour configurer automatiquement les cibles de scraping.

| Ressource | Ce qu'elle fait |
|-----------|-----------------|
| **ServiceMonitor** | Sélectionne les Services via labels, Prometheus scrape les pods derrière |
| **PodMonitor** | Sélectionne les Pods directement via labels |

**Flux :**

1. Developer deploys service with a label (e.g., `app: svc-api`)
2. A ServiceMonitor selects this label
3. Prometheus Operator configure automatiquement Prometheus
4. Prometheus scrape `/metrics` sur le port spécifié

**Avantages :**
- GitOps-friendly — fichier séparé, versionné, reviewable
- Séparation des concerns — monitoring découplé du déploiement
- Flexibilité — intervalles, relabeling, TLS, authentification

### Endpoints

| Service | Port | Path | Description |
|---------|------|------|-------------|
| **svc-api** | 8080 | `/metrics` | Via `promhttp` handler |
| **svc-agent-relay** | 9090 | `/metrics` | gRPC service metrics |
| **svc-provisioner** | 8082 | `/metrics` | Provisioning pipeline metrics |
| **kiven-agent** | 9090 | `/metrics` | Agent-side CNPG + PG metrics |
| **Grafana** | 3000 | `/metrics` | Internal metrics |
| **Flux** | 8080 | `/metrics` | Reconciliation metrics |
| **Node Exporter** | 9100 | `/metrics` | System metrics (CPU, RAM, disk) |

---

# 📝 **Logs (Loki)**

## Configuration

| Paramètre | Valeur | Raison |
|-----------|--------|--------|
| **Retention** | 30 jours | GDPR compliance |
| **Max query series** | 5000 | Protection performance |
| **Max entries per query** | 10000 | Protection performance |
| **Storage backend** | S3 | Coût faible, durabilité |

## Log Labels (Low Cardinality)

| Label | Exemple | Cardinalité |
|-------|---------|-------------|
| `namespace` | svc-ledger | Low |
| `pod` | svc-ledger-abc123 | Medium |
| `container` | svc-ledger | Low |
| `level` | info, error, warn | Very Low |
| `stream` | stdout, stderr | Very Low |

**⚠️ Never use as labels:** `user_id`, `request_id`, `trace_id`

---

# 🔍 **Traces (Tempo)**

## Configuration

| Paramètre | Valeur | Raison |
|-----------|--------|--------|
| **Retention** | 7 jours | Coût / utilité |
| **Backend** | S3 | Durabilité |
| **Protocol** | OTLP (gRPC + HTTP) | Standard OTel |

## Trace-to-Logs Correlation

```
┌─────────────────┐     trace_id     ┌─────────────────┐
│     TRACES      │◄────────────────►│      LOGS       │
│     (Tempo)     │                  │     (Loki)      │
└────────┬────────┘                  └────────┬────────┘
         │                                    │
         │ Exemplars (trace_id in metrics)    │
         │                                    │
         ▼                                    ▼
┌─────────────────────────────────────────────────────────┐
│                    GRAFANA                               │
│  • Click trace → See logs for that request              │
│  • Click metric spike → Jump to exemplar trace          │
│  • Click error log → Navigate to full trace             │
└─────────────────────────────────────────────────────────┘
```

---

# 🎯 **APM (Application Performance Monitoring)**

## Stack APM

| Composant | Outil | Usage |
|-----------|-------|-------|
| **Distributed Tracing** | Tempo + OTel | Request flow, latency breakdown |
| **Profiling** | Pyroscope (Grafana) | CPU/Memory profiling continu |
| **Error Tracking** | Sentry (self-hosted) | Exception tracking, stack traces |
| **Database APM** | pg_stat_statements | Query performance |
| **Real User Monitoring** | Grafana Faro | Frontend performance (si applicable) |

## Sampling Strategy

| Environment | Head Sampling | Tail Sampling | Rationale |
|-------------|---------------|---------------|-----------|
| **Dev** | 100% | N/A | Full visibility pour debug |
| **Staging** | 50% | Errors: 100% | Balance cost/visibility |
| **Prod** | 10% | Errors: 100%, Slow: 100% (>500ms) | Cost optimization |

### Tail Sampling — Règles

| Règle | Condition | Pourquoi |
|-------|-----------|----------|
| **error-policy** | Status = ERROR | Toujours conserver les erreurs |
| **slow-policy** | Latency > 500ms | Détecter les lenteurs |
| **probabilistic-policy** | 10% aléatoire | Échantillonnage de base |

---

# 📉 **Cardinality Management**

## Label Rules

| Label | Action | Rationale |
|-------|--------|-----------|
| `user_id` | DROP | High cardinality, use traces |
| `request_id` | DROP | Use trace_id instead |
| `http.url` | DROP | URLs uniques = explosion |
| `http.route` | KEEP | Templated, low cardinality |
| `service.name` | KEEP | Essential |
| `http.method` | KEEP | Low cardinality |
| `http.status_code` | KEEP | Low cardinality |

## Cardinality Limits

| Metric Type | Max Labels | Max Series |
|-------------|------------|------------|
| Counter | 5 | 1000 |
| Histogram | 4 | 500 |
| Gauge | 5 | 1000 |

---

# 🎯 **SLI/SLO/Error Budgets**

## Service SLOs

| Service | SLI | SLO | Error Budget | Burn Rate Alert |
|---------|-----|-----|--------------|-----------------|
| **svc-api** | Availability | 99.9% | 43 min/month | 14.4x = 1h alert |
| **svc-api** | Latency P99 | < 200ms | N/A | P99 > 200ms for 5min |
| **svc-provisioner** | Availability | 99.9% | 43 min/month | 14.4x = 1h alert |
| **svc-agent-relay** | Availability | 99.95% | 22 min/month | 14.4x = 30min alert |
| **Customer PostgreSQL** | Availability | 99.99% | 4.3 min/month | 6x = 15min alert |
| **Platform (infra)** | Availability | 99.5% | 3.6h/month | 6x = 2h alert |

## SLO Formulas

| Métrique | Formule | Signification |
|----------|---------|---------------|
| **Availability** | `1 - (erreurs / total)` | % de requêtes sans erreur 5xx |
| **Error Budget Remaining** | `1 - ((1 - availability) / (1 - SLO))` | % du budget restant |
| **Burn Rate** | `error_rate / allowed_error_rate` | Vitesse de consommation du budget |

---

# 🚨 **Alerting Strategy**

## Severity Levels

| Severity | Exemple | Notification | On-call |
|----------|---------|--------------|---------|
| **P1 — Critical** | svc-ledger down | PagerDuty immediate | Wake up |
| **P2 — High** | Error rate > 5% | Slack + PagerDuty 15min | Within 30min |
| **P3 — Medium** | Latency P99 > 500ms | Slack | Business hours |
| **P4 — Low** | Disk usage > 80% | Slack | Next day |

## Alertes principales

| Alerte | Condition | Sévérité | Action |
|--------|-----------|----------|--------|
| **ServiceDown** | `up == 0` pendant 1min | P1 | Runbook: restart, check logs |
| **HighErrorRate** | Error rate > 5% pendant 5min | P2 | Investigate traces + Sentry |
| **LatencyDegradation** | P99 > 2x baseline pendant 10min | P2 | Check slow spans in Tempo |
| **DiskAlmostFull** | Disk > 80% | P4 | Extend volume or cleanup |

---

# 📊 **Dashboards & Visualizations**

## Types de visualisations Grafana par type de métrique

### Counter (Compteur)

> **Définition :** Valeur qui ne peut qu'augmenter (ou reset à 0 au restart).

| Visualisation | Query | Quand utiliser |
|---------------|-------|----------------|
| **Stat (nombre)** | `sum(http_requests_total)` | Total absolu |
| **Time Series (rate)** | `rate(http_requests_total[5m])` | Débit par seconde (RPS) |
| **Bar Gauge** | `sum by (status_code) (rate(http_requests_total[5m]))` | Comparaison entre labels |

```
Exemple visuel — Counter en Time Series (rate)

  RPS
  30 │          ╭───╮
     │    ╭────╯   │
  20 │───╯         │
     │             ╰────╮
  10 │                  ╰─────
     └─────────────────────────▶ temps
        10:00   10:05   10:10
```

### Gauge (Jauge)

> **Définition :** Valeur instantanée qui peut monter ou descendre (température, connexions actives, CPU%).

| Visualisation | Query | Quand utiliser |
|---------------|-------|----------------|
| **Gauge (cadran)** | `pg_stat_activity_count` | Valeur courante visuelle |
| **Stat** | `node_memory_MemAvailable_bytes / 1e9` | Valeur simple avec unité |
| **Time Series** | `process_resident_memory_bytes` | Évolution dans le temps |
| **Heatmap** | `avg by (pod) (container_memory_usage_bytes)` | Comparaison multi-pods |

```
Exemple visuel — Gauge en cadran

        ┌─────────────────┐
        │     CPU %       │
        │                 │
        │   ┌───────┐     │
        │   │  67%  │     │
        │   │  ███  │     │
        │   └───────┘     │
        │ 0%         100% │
        └─────────────────┘
```

### Histogram (Histogramme)

> **Définition :** Distribution de valeurs dans des "buckets" (ex: latence). Permet de calculer des percentiles.

| Visualisation | Query | Quand utiliser |
|---------------|-------|----------------|
| **Time Series (P50/P95/P99)** | `histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))` | Latency trends |
| **Heatmap** | `sum by (le) (rate(http_request_duration_seconds_bucket[5m]))` | Distribution visuelle |
| **Stat** | `histogram_quantile(0.95, ...)` | Valeur P95 courante |

```
Exemple visuel — Histogram en Heatmap (latence)

  Latency
  1s    │░░▓▓░░
  500ms │▓▓▓▓▓▓████████
  200ms │████████████████████████
  100ms │██████████████████████████████
  50ms  │████████████████████████████████████
        └────────────────────────────────────▶ temps
           10:00        10:30        11:00
        
  ░ = peu de requêtes   ▓ = moyen   █ = beaucoup
```

### Summary

> **Définition :** Comme Histogram mais les percentiles sont calculés côté client (moins flexible).

| Visualisation | Query | Quand utiliser |
|---------------|-------|----------------|
| **Time Series** | `go_gc_duration_seconds{quantile="0.99"}` | Pré-calculé |
| **Stat** | `go_gc_duration_seconds{quantile="0.5"}` | Médiane |

> **Note :** Préférer Histogram. Summary est principalement utilisé par les exporters Go legacy.

---

## Dashboards recommandés par audience

### Dashboard 1 : Service Overview (On-call)

| Panel | Type | Métrique | Visualisation |
|-------|------|----------|---------------|
| **Request Rate** | Counter | `rate(http_requests_total[5m])` | Time Series |
| **Error Rate %** | Counter | `rate(errors[5m]) / rate(total[5m]) * 100` | Time Series + Threshold |
| **Latency P50/P95/P99** | Histogram | `histogram_quantile(...)` | Time Series (3 lignes) |
| **Active Requests** | Gauge | `http_requests_in_flight` | Stat |

### Dashboard 2 : Infrastructure (Platform Team)

| Panel | Type | Métrique | Visualisation |
|-------|------|----------|---------------|
| **CPU Usage %** | Gauge | `container_cpu_usage_seconds_total` | Gauge cadran |
| **Memory Usage** | Gauge | `container_memory_usage_bytes` | Bar Gauge |
| **Network I/O** | Counter | `rate(container_network_receive_bytes_total[5m])` | Time Series |
| **Disk Usage %** | Gauge | `node_filesystem_avail_bytes / node_filesystem_size_bytes` | Gauge |
| **Pod Count** | Gauge | `kube_pod_status_phase{phase="Running"}` | Stat |

### Dashboard 3 : Database (Backend Devs)

| Panel | Type | Métrique | Visualisation |
|-------|------|----------|---------------|
| **Active Connections** | Gauge | `pg_stat_activity_count` | Gauge cadran |
| **Query Duration P95** | Histogram | `pg_stat_statements_mean_time_seconds` | Time Series |
| **Transactions/sec** | Counter | `rate(pg_stat_database_xact_commit[5m])` | Time Series |
| **Replication Lag** | Gauge | `pg_replication_lag_seconds` | Stat avec threshold |
| **Cache Hit Ratio** | Gauge | `pg_stat_database_blks_hit / (blks_hit + blks_read)` | Stat % |

### Dashboard 4 : Kiven Business Metrics (Product)

| Panel | Type | Metric | Visualization |
|-------|------|--------|---------------|
| **Services Created** | Counter | `sum(rate(kiven_services_created_total[1h]))` | Stat (big number) |
| **Active Databases** | Gauge | `kiven_services_active_count` | Stat |
| **Provisioning Time P95** | Histogram | `histogram_quantile(0.95, rate(kiven_provisioning_duration_bucket[1h]))` | Time Series |
| **Connected Agents** | Gauge | `kiven_agent_connected` | Stat |
| **Backup Success Rate** | Counter | `rate(kiven_backup_success_total[1h]) / rate(kiven_backup_total[1h])` | Gauge % |
| **Business Errors** | Counter | `sum by (error_type) (rate(kiven_errors_total[5m]))` | Bar chart |

---

## Récapitulatif : Quel type pour quelle métrique ?

| Métrique | Type Prometheus | Visualisation Grafana |
|----------|-----------------|----------------------|
| Nombre de requêtes | Counter | Time Series (rate) |
| Erreurs totales | Counter | Time Series (rate) + Stat |
| Latence | Histogram | Time Series (quantile) + Heatmap |
| Connexions actives | Gauge | Gauge cadran ou Stat |
| Mémoire utilisée | Gauge | Time Series ou Bar Gauge |
| CPU % | Gauge | Gauge cadran |
| Durée GC | Summary | Time Series |
| Taille de queue | Gauge | Stat avec threshold |

---

*Maintained by: @kivenio/platform*
*Last updated: February 2026*
*See also: [OTEL-CONVENTIONS.md](./OTEL-CONVENTIONS.md) for instrumentation details*
