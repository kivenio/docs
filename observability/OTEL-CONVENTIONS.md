# OpenTelemetry Conventions

> **Back to**: [Observability Guide](./OBSERVABILITY-GUIDE.md) | [Architecture Overview](../EntrepriseArchitecture.md)

## Architecture Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Collector deployment | Agent + Gateway (two-tier) | Resilient, scalable, centralized config for heavy lifting |
| SDK approach | `kiven-go-sdk/telemetry` wrapping OTel Go SDK | Conventions baked in, zero-config for services |
| Exporter pattern | Exporter helper with persistent queue | New OTel pattern: no separate batch processor, survives restarts |
| Propagation | W3C TraceContext + Baggage | Industry standard, cross-service compatible |
| Metrics backend | Prometheus (scrape) + OTel Collector (receive) | Prometheus for K8s ecosystem, OTel for application metrics |
| Traces backend | Tempo | Grafana-native, S3 storage, cost effective |
| Logs backend | Loki | Grafana-native, label-based, low cost |

---

## Collector Deployment: Agent + Gateway Two-Tier

```
Services (pods)
  │
  │ OTLP gRPC (localhost:4317)
  ▼
OTel Collector Agent (DaemonSet, 1 per node)
  │  Lightweight: receive → forward
  │  No processing, no sampling
  │
  │ OTLP gRPC (cluster-internal)
  ▼
OTel Collector Gateway (Deployment, 2-3 replicas)
  │  Heavy lifting: batch, filter, sample, scrub PII
  │  Exporter helper with persistent queue
  │
  ├──► Tempo (traces)
  ├──► Prometheus remote write (metrics)
  └──► Loki (logs)
```

### Why Two-Tier

- **Agent (DaemonSet)**: minimal config, low memory (~50MB), just forwards. If a node dies, only that node's in-flight data is lost.
- **Gateway (Deployment)**: centralized processing, tail sampling decisions, PII scrubbing, persistent queue. Horizontally scalable.
- **Alternative considered**: Sidecar per pod -- rejected because 15+ services means 15+ Collector instances consuming memory. DaemonSet shares one Collector per node.

### Exporter Helper: Persistent Queue (New Pattern)

Since OTel Collector v0.110+, the recommended pattern is exporter-level batching with persistent storage. This replaces the old separate `batch` processor.

```yaml
# Gateway Collector config
exporters:
  otlp/tempo:
    endpoint: tempo:4317
    tls:
      insecure: true
    sending_queue:
      enabled: true
      storage: file_storage/traces       # persistent: survives collector restart
      queue_size: 5000
      num_consumers: 10
    retry_on_failure:
      enabled: true
      initial_interval: 5s
      max_interval: 30s
      max_elapsed_time: 300s
    batcher:
      enabled: true
      min_size: 500
      max_size: 2000
      timeout: 5s

extensions:
  file_storage/traces:
    directory: /var/lib/otel/traces       # PVC-backed for persistence
    timeout: 10s
    compaction:
      on_start: true
      directory: /tmp/otel-compaction
```

**Why this matters**: If the Gateway restarts (upgrade, OOM, node drain), queued spans are not lost. They're persisted to disk and replayed on startup.

---

## Span Naming Convention

All spans follow this pattern:

```
kiven.<service>.<layer>.<operation>
```

### Layers

| Layer | Description | Examples |
|-------|-------------|----------|
| `handler` | HTTP/gRPC request handlers | `kiven.svc-api.handler.CreateService` |
| `repo` | Database repository methods | `kiven.svc-api.repo.InsertService` |
| `provider` | Provider interface calls | `kiven.provider-cnpg.provider.GenerateClusterYAML` |
| `infra` | AWS/cloud infrastructure calls | `kiven.svc-infra.infra.CreateNodeGroup` |
| `agent` | Agent-side operations | `kiven.agent.agent.ApplyYAML` |
| `grpc` | gRPC calls (auto-generated) | `/kiven.agent.v1.AgentRelay/Heartbeat` |

### Auto-Generated Spans

The `kiven-go-sdk/telemetry` package generates spans automatically:

| Source | Span Name Format | Example |
|--------|-------------------|---------|
| HTTP middleware | `HTTP {method} {path}` | `HTTP GET /v1/services` |
| gRPC server interceptor | `/{package}.{Service}/{Method}` | `/kiven.agent.v1.AgentRelay/Heartbeat` |
| gRPC client interceptor | `/{package}.{Service}/{Method}` | `/kiven.agent.v1.AgentRelay/SendCommand` |
| Manual (Trace helper) | Developer-defined | `repo.GetService`, `aws.CreateNodeGroup` |

### Manual Span Creation

Use the helpers from `kiven-go-sdk/telemetry`:

```go
// Simple span
ctx, span := telemetry.Trace(ctx, "repo.GetService")
defer span.End()

// Span with automatic error handling
err := telemetry.TraceFunc(ctx, "svc.CreateService", func(ctx context.Context) error {
    return repo.Insert(ctx, svc)
})

// Add domain attributes to current span
telemetry.SetSpanAttributes(ctx,
    attribute.String("kiven.service_id", serviceID),
    attribute.String("kiven.plan", "business"),
)

// Get trace ID for log correlation
traceID := telemetry.TraceID(ctx)
```

---

## Standard Attributes

Every span SHOULD include these attributes where applicable:

### Kiven Domain Attributes

| Attribute | Type | Description | Example |
|-----------|------|-------------|---------|
| `kiven.org_id` | string | Organization ID | `org-abc123` |
| `kiven.project_id` | string | Project ID | `proj-xyz789` |
| `kiven.service_id` | string | Managed database service ID | `svc-pg-001` |
| `kiven.cluster_id` | string | Customer EKS cluster ID | `cluster-eu-west-1` |
| `kiven.plan` | string | Service plan | `business` |
| `kiven.env` | string | Environment | `production` |
| `kiven.agent_id` | string | Agent instance ID | `agent-abc123` |

### OTel Semantic Convention Attributes (auto-set by middleware)

| Attribute | Set By | Example |
|-----------|--------|---------|
| `http.request.method` | HTTP middleware | `GET` |
| `url.path` | HTTP middleware | `/v1/services` |
| `http.response.status_code` | HTTP middleware | `200` |
| `rpc.system` | gRPC interceptor | `grpc` |
| `rpc.service` | gRPC interceptor | `kiven.agent.v1.AgentRelay` |
| `rpc.method` | gRPC interceptor | `Heartbeat` |
| `rpc.grpc.status_code` | gRPC interceptor | `0` (OK) |
| `service.name` | Provider resource | `svc-api` |
| `deployment.environment` | Provider resource | `production` |

---

## Metric Naming

Follow OTel semantic conventions. Custom Kiven metrics use the `kiven.` prefix.

### Standard Metrics (auto-collected)

| Metric | Type | Source |
|--------|------|--------|
| `http.server.duration` | Histogram | HTTP middleware |
| `http.server.request.size` | Histogram | HTTP middleware |
| `rpc.server.duration` | Histogram | gRPC interceptor |
| `db.client.operation.duration` | Histogram | pgx tracing (Phase 1 gap) |

### Kiven Business Metrics (service-specific)

| Metric | Type | Service | Description |
|--------|------|---------|-------------|
| `kiven.provisioning.duration` | Histogram | svc-provisioner | Time to provision a database |
| `kiven.provisioning.active` | Gauge | svc-provisioner | In-flight provisioning jobs |
| `kiven.agent.connected` | Gauge | svc-agent-relay | Connected agents count |
| `kiven.agent.heartbeat.lag` | Histogram | svc-agent-relay | Time since last heartbeat |
| `kiven.backup.duration` | Histogram | svc-backups | Backup execution time |
| `kiven.backup.size` | Gauge | svc-backups | Last backup size in bytes |

---

## Sampling Strategy

Configured via `kiven-go-sdk/telemetry` Config:

| Environment | Head Sampling Rate | Tail Sampling (Gateway) | Rationale |
|-------------|-------------------|-------------------------|-----------|
| `local` | 100% | N/A (stdout exporter) | Full visibility for development |
| `staging` | 100% | Keep all | Full visibility for QA |
| `production` | 10% | Errors: 100%, Slow (>500ms): 100% | Cost optimization |

### Head Sampling (SDK-side)

Set via `OTEL_SAMPLE_RATE` env var or `Config.SampleRate`. Uses `ParentBased(TraceIDRatioBased(rate))` so child spans always respect parent's sampling decision.

### Tail Sampling (Gateway Collector)

The Gateway applies tail sampling after receiving all spans of a trace:

```yaml
processors:
  tail_sampling:
    decision_wait: 10s
    policies:
      - name: errors
        type: status_code
        status_code: {status_codes: [ERROR]}
      - name: slow
        type: latency
        latency: {threshold_ms: 500}
      - name: probabilistic
        type: probabilistic
        probabilistic: {sampling_percentage: 10}
```

---

## SDK Usage: kiven-go-sdk/telemetry

### Service Bootstrap

```go
func main() {
    ctx := context.Background()

    // Auto-configures from env vars (OTEL_EXPORTER_TYPE, OTEL_SAMPLE_RATE, etc.)
    cfg, err := telemetry.NewConfigFromEnv("svc-api")
    if err != nil {
        log.Fatal(err)
    }

    tp, err := telemetry.NewProvider(ctx, cfg)
    if err != nil {
        log.Fatal(err)
    }
    defer tp.Shutdown(ctx)

    // HTTP server with tracing middleware
    r := chi.NewRouter()
    r.Use(telemetry.HTTPMiddleware("svc-api"))
    // ...
}
```

### gRPC Server with Tracing

```go
server := grpc.NewServer(
    grpc.UnaryInterceptor(telemetry.UnaryServerInterceptor()),
    grpc.StreamInterceptor(telemetry.StreamServerInterceptor()),
)
```

### gRPC Client with Tracing

```go
conn, _ := grpc.Dial(address,
    grpc.WithUnaryInterceptor(telemetry.UnaryClientInterceptor()),
    grpc.WithStreamInterceptor(telemetry.StreamClientInterceptor()),
)
```

### Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `OTEL_EXPORTER_TYPE` | `stdout` | `stdout`, `otlp`, or `none` |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | (none) | `host:port` of OTel Collector |
| `OTEL_EXPORTER_OTLP_INSECURE` | `false` | Skip TLS for local collectors |
| `OTEL_ENVIRONMENT` | `local` | `local`, `staging`, `production` |
| `OTEL_SAMPLE_RATE` | env-based | `0.0`-`1.0` (negative = auto) |
| `OTEL_SERVICE_VERSION` | (none) | Service version for resource |

---

## What's Implemented vs Gaps

### Implemented (kiven-go-sdk/telemetry)

| File | What It Does |
|------|-------------|
| `config.go` | Config struct, DefaultConfig, NewConfigFromEnv, exporter types, env-based sampling |
| `provider.go` | TracerProvider with resource, exporter, sampler, global registration, W3C propagation |
| `span.go` | Trace(), TraceFunc(), SetSpanError(), SetSpanAttributes(), TraceID() helpers |
| `httpmiddleware.go` | chi-compatible HTTP middleware with W3C extraction, semantic convention attrs, X-Trace-ID header |
| `grpc.go` | Full gRPC instrumentation: Unary/Stream Server/Client interceptors with metadata propagation |
| Tests | config_test.go, provider_test.go, span_test.go, httpmiddleware_test.go, grpc_test.go |

### Phase 1 Gaps (to implement)

| Gap | Description | Priority |
|-----|-------------|----------|
| **MeterProvider** | OTel MeterProvider setup (like TracerProvider but for metrics). Services need `meter.Int64Counter()`, `meter.Float64Histogram()` etc. | P0 |
| **slog bridge** | Bridge Go `log/slog` to OTel Logs so structured logs flow through the same pipeline as traces/metrics | P1 |
| **pgx tracing hook** | `pgx.QueryTracer` implementation that auto-creates spans for every SQL query with `db.statement`, `db.operation` attributes | P0 |
| **Kiven attribute constants** | Package-level constants for `kiven.org_id`, `kiven.service_id` etc. to avoid string typos | P1 |

---

## GDPR Compliance in OTel Pipeline

The Gateway Collector scrubs PII before exporting:

| Data | Action | Processor |
|------|--------|-----------|
| `user.id` | Drop | `attributes/delete` |
| `user.email` | Drop | `attributes/delete` |
| `http.client_ip` | Hash | `transform` |
| High cardinality metric labels | Drop | `filter` |
| SQL query parameters | Redact | `transform` (replace bind values with `?`) |

```yaml
processors:
  attributes/scrub:
    actions:
      - key: user.id
        action: delete
      - key: user.email
        action: delete
      - key: enduser.id
        action: delete
  transform/anonymize:
    trace_statements:
      - context: span
        statements:
          - replace_pattern(attributes["http.client_ip"], "^(.*)$", "REDACTED")
```
