# Kata 24: Golden Signals Dashboard with Prometheus & Grafana

**Target Idioms:** RED Metrics, SLI/SLO Patterns, Prometheus Queries, Observable Services
**Difficulty:** 🔴 Advanced

## 🧠 The "Why"
Many developers collect metrics but don't know **which** metrics matter. Google's SRE book defines the **Four Golden Signals**:
1. **Latency** - How long requests take
2. **Traffic** - How many requests you're serving
3. **Errors** - Rate of failing requests
4. **Saturation** - How "full" your service is

Without these, you're monitoring **vanity metrics** (uptime, CPU) while your users suffer from slow checkout flows and 500 errors.

## 🎯 The Scenario
You're the platform engineer for a **critical payment API**. Your SLA promises:
- 99.9% availability (< 43 minutes downtime/month)
- p95 latency < 500ms
- Error rate < 0.1%

Your task is to build a Prometheus-based monitoring stack that:
- Exposes RED metrics from your Go service
- Provides a Grafana dashboard for real-time monitoring
- Alerts when SLOs are breached

## 🛠 The Challenge
Create a comprehensive observability stack using Prometheus, Grafana, and Go instrumentation.

### 1. Functional Requirements
* [ ] Instrument a sample HTTP service with Prometheus metrics (use kata 21's patterns)
* [ ] Create a `docker-compose.yml` with Prometheus, Grafana, and your Go app
* [ ] Configure Prometheus scrape targets and retention
* [ ] Build a Grafana dashboard showing:
  - Request rate (per second) by endpoint
  - Error rate (%) by endpoint and status code
  - Latency histogram (p50, p95, p99)
  - Active connections gauge
  - Apdex score (Application Performance Index)
* [ ] Define alert rules for SLO violations (error rate > 1%, p95 > 500ms)
* [ ] Create a `/health` and `/ready` endpoint for Kubernetes-style health checks

### 2. The "Idiomatic" Constraints (Pass/Fail Criteria)
To pass this kata, you **must** strictly adhere to these rules:

* [ ] **RED Metrics:** Your service must expose Rate, Errors, Duration metrics
* [ ] **Histogram Buckets:** Use appropriate buckets for your SLA (e.g., 0.1, 0.25, 0.5, 1.0 seconds)
* [ ] **PromQL Queries:** Use rate/irate for counters, histogram_quantile for percentiles
* [ ] **Cardinality Control:** Limit path labels (no user IDs in metric labels!)
* [ ] **Recording Rules:** Pre-compute expensive queries using Prometheus recording rules
* [ ] **Alert Annotations:** Include runbook links in Prometheus alerts
* [ ] **Multi-Window Alerting:** Use both short (5m) and long (30m) windows to avoid flapping

### 3. Bonus Challenges (Optional)
* [ ] Add USE metrics (Utilization, Saturation, Errors) for backend resources
* [ ] Implement synthetic monitoring (blackbox exporter for external endpoints)
* [ ] Add multi-burnrate SLO alerts (Google SRE Workbook pattern)
* [ ] Create a "Service Health Score" panel combining multiple signals
* [ ] Add exemplar support for linking metrics to traces

## 🧪 Self-Correction (Test Yourself)

### Test 1: The "Cardinality Bomb"
```prometheus
# BAD: Creates millions of unique time series
http_requests_total{user_id="12345", session_id="abc..."}
```
**Pass Condition:** Your metrics should only have:
```prometheus
http_requests_total{method="GET", path="/api/users/:id", status="200"}
```

### Test 2: The "Percentile Math"
```prometheus
# BAD: Averaging percentiles is mathematically wrong!
avg(http_request_duration_seconds{quantile="0.95"})
```
**Pass Condition:** Use histogram_quantile:
```prometheus
histogram_quantile(0.95, 
  rate(http_request_duration_seconds_bucket[5m])
)
```

### Test 3: The "SLO Breach"
* Simulate 10% error rate for 2 minutes
* **Pass Condition:** Your Prometheus alert should fire within 30 seconds
* Check Grafana dashboard shows error spike in red

### Test 4: The "Missing Context"
* Trigger an alert
* Check the alert annotation
* **Pass Condition:** Alert should include:
  - Description of the problem
  - Link to runbook/documentation
  - Current values vs. threshold
  - Affected service/endpoint

## 📚 Resources
* [Prometheus Querying Basics](https://prometheus.io/docs/prometheus/latest/querying/basics/)
* [Grafana Dashboard Best Practices](https://grafana.com/docs/grafana/latest/dashboards/build-dashboards/best-practices/)
* [Google SRE Book - Monitoring](https://sre.google/sre-book/monitoring-distributed-systems/)
* [RED Method (Tom Wilkie)](https://www.weave.works/blog/the-red-method-key-metrics-for-microservices-architecture/)
* [Prometheus Alerting Best Practices](https://prometheus.io/docs/practices/alerting/)

## 💡 Key Patterns to Master
1. **Histogram vs Summary:** Always use histograms for server-side aggregation
2. **rate() vs irate():** Use `rate()` for alerts, `irate()` for graphs
3. **Label Consistency:** Use the same label names across all your metrics
4. **Recording Rules:** Pre-compute expensive percentile queries
5. **Alert Runbooks:** Every alert needs a runbook URL

## 🏗 Required Files

### `docker-compose.yml`
```yaml
version: '3.8'
services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      - LOG_LEVEL=info
    labels:
      - "prometheus.scrape=true"
      - "prometheus.port=8080"
      - "prometheus.path=/metrics"

  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - ./alerts.yml:/etc/prometheus/alerts.yml
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.retention.time=30d'

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - ./grafana/dashboards:/etc/grafana/provisioning/dashboards
      - ./grafana/datasources.yml:/etc/grafana/provisioning/datasources/datasources.yml
```

### `prometheus.yml`
```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - 'alerts.yml'

scrape_configs:
  - job_name: 'payment-api'
    static_configs:
      - targets: ['app:8080']
    metrics_path: '/metrics'
```

### `alerts.yml`
```yaml
groups:
  - name: slo_alerts
    interval: 30s
    rules:
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m])) 
          / 
          sum(rate(http_requests_total[5m])) > 0.01
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "High error rate detected"
          description: "Error rate is {{ $value | humanizePercentage }}"
          runbook_url: "https://wiki.example.com/runbooks/high-error-rate"

      - alert: HighLatency
        expr: |
          histogram_quantile(0.95,
            rate(http_request_duration_seconds_bucket[5m])
          ) > 0.5
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "p95 latency above SLO"
          description: "p95 latency is {{ $value }}s (SLO: 0.5s)"
```

## 🎯 Grafana Dashboard Panels

Your dashboard must include:

1. **Request Rate Panel**
   - Query: `sum(rate(http_requests_total[5m])) by (method, path)`
   - Type: Time series graph
   - Unit: requests/sec

2. **Error Rate Panel**
   - Query: `sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))`
   - Type: Stat panel with threshold (red > 1%)
   - Unit: percentunit

3. **Latency Percentiles Panel**
   - Queries:
     - p50: `histogram_quantile(0.50, rate(http_request_duration_seconds_bucket[5m]))`
     - p95: `histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))`
     - p99: `histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))`
   - Type: Time series graph with SLO threshold line
   - Unit: seconds

4. **Apdex Score Panel**
   - Query:
     ```promql
     (
       sum(rate(http_request_duration_seconds_bucket{le="0.5"}[5m]))
       + sum(rate(http_request_duration_seconds_bucket{le="2.0"}[5m])) / 2
     ) / sum(rate(http_request_duration_seconds_count[5m]))
     ```
   - Type: Gauge (0-1)
   - Thresholds: > 0.95 green, > 0.85 yellow, < 0.85 red

## 🚀 Testing Your Stack

```bash
# 1. Start the stack
docker-compose up -d

# 2. Generate load
for i in {1..1000}; do
  curl http://localhost:8080/api/checkout
done

# 3. Access Grafana
open http://localhost:3000
# Login: admin/admin

# 4. Access Prometheus
open http://localhost:9090

# 5. Simulate errors
for i in {1..100}; do
  curl -X POST http://localhost:8080/api/fail
done

# 6. Check alerts
curl http://localhost:9090/api/v1/alerts
```
