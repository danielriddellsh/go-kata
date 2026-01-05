# Kata 25: Full-Stack Observability: The Complete Observability Stack

**Target Idioms:** Unified Observability (Metrics + Traces + Logs), Context Correlation, Production Readiness
**Difficulty:** 🔴 Advanced

## 🧠 The "Why"
The real power of observability emerges when **metrics, traces, and logs are correlated**. Developers often implement them as separate systems, making debugging painful:
- Metrics show high latency, but which trace?
- Trace shows slow span, but what were the logs?
- Logs show errors, but what's the request rate?

**Idiomatic observability** means: one request ID ties together all three signals, enabling you to jump from a Grafana spike → specific trace → relevant logs in seconds.

## 🎯 The Scenario
You're building a **real-time notification service** that processes millions of events per hour. When a production incident occurs:

1. **Alert fires:** Grafana shows error rate spike
2. **Trace analysis:** You need the slow trace ID from the metrics
3. **Log correlation:** You need logs for that specific trace
4. **Root cause:** What user/tenant triggered this? What was the input?

Your observability stack must answer these questions in < 2 minutes.

## 🛠 The Challenge
Build a complete observability stack that unifies metrics, traces, and logs with full correlation.

### 1. Functional Requirements

#### Application Layer
* [ ] Build a sample microservice with 3 endpoints:
  - `POST /notifications` - Send notification (calls downstream services)
  - `GET /health` - Health check
  - `GET /ready` - Readiness check
* [ ] Implement at least 2 downstream dependencies:
  - User service (HTTP call)
  - Database operation (simulated or real)

#### Metrics Layer (Prometheus)
* [ ] RED metrics for all HTTP endpoints
* [ ] Custom business metrics (e.g., `notifications_sent_total`)
* [ ] Exemplar support linking metrics to traces
* [ ] Service health metrics (DB connection pool, goroutine count)

#### Tracing Layer (OpenTelemetry + Jaeger/Tempo)
* [ ] End-to-end distributed tracing
* [ ] Automatic HTTP client/server instrumentation
* [ ] Custom spans for business logic
* [ ] Trace ID in all log messages
* [ ] Span events for critical operations

#### Logging Layer (slog + Loki)
* [ ] JSON structured logging
* [ ] Automatic request ID and trace ID injection
* [ ] Log levels: DEBUG, INFO, WARN, ERROR
* [ ] Loki labels for filtering (service, environment, level)
* [ ] Correlation with traces via trace ID

#### Visualization Layer (Grafana)
* [ ] Unified dashboard with:
  - Request rate, error rate, latency (RED)
  - Active traces panel with exemplars
  - Logs panel filtered by trace ID
  - Service health indicators
* [ ] Alert rules for SLO violations
* [ ] Trace-to-logs navigation
* [ ] Metrics-to-traces navigation

### 2. The "Idiomatic" Constraints (Pass/Fail Criteria)

* [ ] **Single Context Flow:** One `context.Context` flows through metrics, traces, and logs
* [ ] **Trace ID Propagation:** Same trace ID appears in metrics (exemplars), traces, and logs
* [ ] **Zero Code Duplication:** Use middleware/interceptors, not manual instrumentation everywhere
* [ ] **Graceful Degradation:** If tracing backend is down, app still works (logs warning)
* [ ] **Resource Cleanup:** All exporters shutdown gracefully with timeouts
* [ ] **Production Config:** Support environment-based configuration (dev vs. prod)
* [ ] **Cardinality Control:** No high-cardinality labels in metrics (user IDs → separate dimension)

### 3. Infrastructure Requirements

Create a complete Docker Compose stack with:

* [ ] Your Go application (with hot-reload for development)
* [ ] Prometheus (metrics storage + alerting)
* [ ] Jaeger or Grafana Tempo (trace backend)
* [ ] Grafana Loki (log aggregation)
* [ ] Grafana (unified visualization)
* [ ] Optional: AlertManager for alert routing

## 🧪 Self-Correction (Test Yourself)

### Test 1: The "Correlation Test"
```bash
# 1. Send a request
curl -X POST http://localhost:8080/notifications \
  -H "Content-Type: application/json" \
  -d '{"user_id": "123", "message": "Hello"}'

# 2. Copy the trace ID from response headers or logs
TRACE_ID="4bf92f3577b34da6a3ce929d0e0e4736"

# 3. Find it in all three systems:
# - Prometheus: Check exemplars in histogram metrics
# - Jaeger: Search for trace ID
# - Loki: Query logs by trace ID
```

**Pass Condition:** You should find the **same trace ID** in all three systems, showing the complete request flow.

### Test 2: The "Incident Simulation"
```bash
# Simulate high error rate
for i in {1..100}; do
  curl http://localhost:8080/notifications/fail
done
```

**Pass Condition:**
1. Prometheus alert should fire within 1 minute
2. Grafana dashboard shows error spike
3. Clicking a point on the graph shows example traces
4. Opening a trace shows all spans with errors
5. Clicking "Logs" in trace view shows correlated error logs

### Test 3: The "Performance Regression"
```bash
# Simulate slow downstream service
curl -X POST http://localhost:8080/config/slow-mode -d '{"latency_ms": 2000}'

# Send traffic
for i in {1..50}; do
  curl http://localhost:8080/notifications
done
```

**Pass Condition:**
1. p95 latency metric crosses threshold
2. Trace view identifies **which** span is slow (user-service vs. database)
3. Logs show "slow operation" warnings with timing details
4. You can drill down from metric → trace → logs in < 30 seconds

## 📚 Resources

### Core Libraries
* [OpenTelemetry Go](https://github.com/open-telemetry/opentelemetry-go)
* [Prometheus Go Client](https://github.com/prometheus/client_golang)
* [log/slog](https://pkg.go.dev/log/slog)

### Integration Guides
* [OpenTelemetry Exemplars](https://opentelemetry.io/docs/specs/otel/metrics/supplementary-guidelines/#exemplars)
* [Grafana Trace-to-Logs](https://grafana.com/docs/grafana/latest/explore/trace-integration/)
* [Loki Label Best Practices](https://grafana.com/docs/loki/latest/best-practices/)

### Architecture Patterns
* [Google SRE Observability Chapter](https://sre.google/sre-book/monitoring-distributed-systems/)
* [Three Pillars of Observability](https://www.oreilly.com/library/view/distributed-systems-observability/9781492033431/ch04.html)

## 💡 Key Integration Patterns

### 1. Trace ID Injection (Context → All Systems)
```go
// Extract trace ID from OpenTelemetry span
func traceIDFromContext(ctx context.Context) string {
    span := trace.SpanFromContext(ctx)
    if !span.SpanContext().IsValid() {
        return ""
    }
    return span.SpanContext().TraceID().String()
}

// Add to logger
logger := slog.With(slog.String("trace_id", traceIDFromContext(ctx)))

// Add to metrics (exemplar)
counter.Add(ctx, 1) // Context carries trace info for exemplars
```

### 2. Exemplar Support (Metrics → Traces)
```go
// Prometheus histogram with exemplars enabled
histogram := promauto.NewHistogram(prometheus.HistogramOpts{
    Name: "http_request_duration_seconds",
    Buckets: prometheus.DefBuckets,
    // Exemplars automatically extract trace ID from context!
})

// When observing, pass context
histogram.(prometheus.ExemplarObserver).ObserveWithExemplar(
    duration.Seconds(),
    prometheus.Labels{"trace_id": traceIDFromContext(ctx)},
)
```

### 3. Unified Middleware
```go
func ObservabilityMiddleware(
    logger *slog.Logger,
    tracer trace.Tracer,
    metrics *Metrics,
) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            ctx := r.Context()
            
            // 1. Start trace
            ctx, span := tracer.Start(ctx, "http-request")
            defer span.End()
            
            // 2. Enrich logger with trace ID
            reqLogger := logger.With(
                slog.String("trace_id", traceIDFromContext(ctx)),
                slog.String("request_id", generateRequestID()),
            )
            ctx = ContextWithLogger(ctx, reqLogger)
            
            // 3. Record metrics with exemplar
            start := time.Now()
            defer func() {
                duration := time.Since(start)
                metrics.RecordRequest(ctx, r.Method, r.URL.Path, duration)
            }()
            
            reqLogger.Info("request started")
            next.ServeHTTP(w, r.WithContext(ctx))
            reqLogger.Info("request completed")
        })
    }
}
```

## 🏗 Expected Directory Structure
```
.
├── docker-compose.yml
├── Dockerfile
├── go.mod
├── go.sum
├── main.go
├── internal/
│   ├── server/
│   │   └── handler.go
│   ├── observability/
│   │   ├── metrics.go
│   │   ├── tracing.go
│   │   └── logging.go
│   └── middleware/
│       └── observability.go
├── config/
│   ├── prometheus/
│   │   ├── prometheus.yml
│   │   └── alerts.yml
│   ├── grafana/
│   │   ├── dashboards/
│   │   │   └── service-overview.json
│   │   └── provisioning/
│   │       ├── datasources/
│   │       │   └── datasources.yml
│   │       └── dashboards/
│   │           └── dashboards.yml
│   └── loki/
│       └── loki-config.yml
└── README.md
```

## 🎯 Grafana Dashboard Requirements

Your unified dashboard must include:

### Row 1: Service Health (RED Metrics)
- **Request Rate:** `sum(rate(http_requests_total[5m])) by (method, path)`
- **Error Rate:** `sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))`
- **Duration p95:** `histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))`

### Row 2: Trace Integration
- **Recent Traces:** Panel showing clickable trace IDs (from exemplars)
- **Trace Timeline:** Tempo data source showing request flow

### Row 3: Logs Integration
- **Error Logs:** Loki query filtered by `level="error"` with trace ID links
- **Recent Activity:** Last 50 log entries with color coding by level

### Row 4: Business Metrics
- **Notifications Sent:** `sum(rate(notifications_sent_total[5m]))`
- **Notification Latency:** Custom histogram for business operation

### Links Configuration
```json
{
  "links": [
    {
      "title": "View Trace",
      "url": "http://localhost:3000/explore?left={\"datasource\":\"tempo\",\"queries\":[{\"query\":\"${__field.labels.trace_id}\"}]}"
    },
    {
      "title": "View Logs",
      "url": "http://localhost:3000/explore?left={\"datasource\":\"loki\",\"queries\":[{\"expr\":\"{job=\\\"app\\\"} |= \\\"${__field.labels.trace_id}\\\"\"}]}"
    }
  ]
}
```

## 🚀 Getting Started

```bash
# 1. Clone and setup
git clone <your-repo>
cd observability-kata

# 2. Start the stack
docker-compose up -d

# 3. Wait for services to be ready
curl http://localhost:8080/health

# 4. Access Grafana
open http://localhost:3000
# Login: admin/admin

# 5. Generate some traffic
for i in {1..100}; do
  curl -X POST http://localhost:8080/notifications \
    -H "Content-Type: application/json" \
    -d "{\"user_id\": \"user-$i\", \"message\": \"Test message $i\"}"
  sleep 0.1
done

# 6. Explore observability
# - Grafana: http://localhost:3000
# - Prometheus: http://localhost:9090
# - Jaeger: http://localhost:16686 (if using Jaeger)
# - App metrics: http://localhost:8080/metrics
```

## ✅ Success Criteria

You've mastered this kata when:

1. ✅ A single `curl` request generates correlated entries in all three systems
2. ✅ Clicking a Grafana alert links directly to relevant traces
3. ✅ Opening a trace in Jaeger/Tempo shows a "Logs" button that queries Loki
4. ✅ Your app continues working even if Jaeger/Tempo is down (logs warning)
5. ✅ You can answer "Why is the p95 high?" in < 2 minutes using the stack
6. ✅ No user IDs or sensitive data appear in metric labels
7. ✅ The entire stack starts with one command: `docker-compose up`

## 🎓 What You'll Learn

- How to design observability from day one (not bolted on later)
- The power of correlation: metrics → traces → logs navigation
- Production debugging workflows that scale
- OpenTelemetry best practices for Go
- Grafana advanced features (exemplars, data links, unified search)
- How to avoid common pitfalls (cardinality explosions, missing context, orphaned spans)

---

**Pro Tip:** Once you complete this kata, you'll have a production-ready observability template you can use for any Go service!
