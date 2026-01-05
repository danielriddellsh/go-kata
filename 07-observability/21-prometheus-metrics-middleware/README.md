# Kata 21: Production-Grade HTTP Metrics Middleware

**Target Idioms:** Prometheus Integration, HTTP Middleware Patterns, Performance Observability
**Difficulty:** 🟡 Intermediate

## 🧠 The "Why"
Developers from other ecosystems often slap metrics libraries onto their code with global state and tight coupling. In Go, **idiomatic observability** means:
- Metrics collectors are **dependencies**, not globals
- Middleware should be **composable** and testable
- You must track **latency histograms**, not just counters (p50, p95, p99 matter in production)

Without proper middleware instrumentation, you're flying blind when incidents happen at 3 AM.

## 🎯 The Scenario
You're building a **payment processing API** that must be production-ready. Your SRE team requires:
- Request count by HTTP method and status code
- Request duration histogram with percentile tracking
- Active in-flight requests gauge
- All metrics must be exposed at `/metrics` in Prometheus format

If a sudden spike in 500s occurs at midnight, the on-call engineer needs to know **immediately** which endpoint is failing.

## 🛠 The Challenge
Build a Prometheus metrics middleware that instruments HTTP handlers without code duplication.

### 1. Functional Requirements
* [ ] Track total HTTP requests (counter) with labels: `method`, `path`, `status_code`
* [ ] Track request duration (histogram) with reasonable buckets (e.g., 0.1s, 0.5s, 1s, 2s, 5s)
* [ ] Track in-flight requests (gauge) that increments on entry and decrements on exit
* [ ] Expose metrics via `/metrics` endpoint using Prometheus HTTP handler
* [ ] Support custom metric namespaces (e.g., `myapp_http_*`)

### 2. The "Idiomatic" Constraints (Pass/Fail Criteria)
To pass this kata, you **must** strictly adhere to these rules:

* [ ] **NO Global Prometheus Registry:** Inject `prometheus.Registerer` as a dependency
* [ ] **Middleware Pattern:** Your solution must implement `func(http.Handler) http.Handler`
* [ ] **Functional Options:** Use functional options for middleware configuration (namespace, buckets)
* [ ] **Path Cardinality Control:** Prevent metric explosion - limit path labels (e.g., use route patterns like `/users/:id`, not actual IDs)
* [ ] **Cleanup on Panic:** Ensure in-flight gauge decrements even if handler panics (use `defer`)
* [ ] **No Allocation Hot Path:** Minimize allocations in the middleware (reuse label values where possible)

### 3. Bonus Challenges (Optional)
* [ ] Add support for custom metric labels via context values
* [ ] Implement a "slow request" log when duration exceeds a threshold
* [ ] Add support for response size tracking (bytes written)

## 🧪 Self-Correction (Test Yourself)

### Test 1: The "Label Explosion"
```go
// BAD: This will create unlimited unique metric series
http.HandleFunc("/users/123", handler)
http.HandleFunc("/users/456", handler)
// Each user ID creates a new metric time series!
```
**Pass Condition:** Your middleware must normalize paths like `/users/123` → `/users/:id`

### Test 2: The "Panic Safety"
```go
handler := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    panic("database exploded")
})
instrumentedHandler := metricsMiddleware(handler)
instrumentedHandler.ServeHTTP(w, r)
```
**Pass Condition:** Your in-flight requests gauge must return to 0 after panic recovery

### Test 3: The "Percentile Accuracy"
* Send 1000 requests with random durations (50-500ms)
* Query the `/metrics` endpoint
* **Pass Condition:** You should see histogram buckets like:
  ```
  http_request_duration_seconds_bucket{le="0.1"} 150
  http_request_duration_seconds_bucket{le="0.5"} 950
  http_request_duration_seconds_bucket{le="+Inf"} 1000
  ```

## 📚 Resources
* [Prometheus Go Client Library](https://github.com/prometheus/client_golang)
* [Prometheus Metric Types](https://prometheus.io/docs/concepts/metric_types/)
* [Prometheus Best Practices](https://prometheus.io/docs/practices/naming/)
* [Go HTTP Middleware Patterns](https://www.alexedwards.net/blog/making-and-using-middleware)

## 💡 Key Patterns to Master
1. **Dependency Injection for Testability:** Pass `prometheus.Registerer`, not global `prometheus.DefaultRegisterer`
2. **Histogram vs Summary:** Use histograms (server-side aggregation) not summaries (client-side)
3. **Label Cardinality:** Keep cardinality < 100 per metric (path patterns, not IDs!)
4. **defer for Cleanup:** Always decrement gauges in `defer` blocks
5. **MustRegister in Constructor:** Register metrics once in middleware constructor, not per-request
