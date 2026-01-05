# Kata 22: Distributed Tracing with OpenTelemetry

**Target Idioms:** Context Propagation, OpenTelemetry Integration, Span Lifecycle Management
**Difficulty:** 🔴 Advanced

## 🧠 The "Why"
In microservices architectures, a single user request might traverse 10+ services. When latency spikes, you need to know **which** service is slow. Developers from monolithic backgrounds often:
- Forget to propagate trace context across service boundaries
- Create orphaned spans that don't connect to the parent trace
- Leak spans by not calling `span.End()`

**Idiomatic Go tracing** means treating spans as resources that must be cleaned up, using `context.Context` as the carrier for trace metadata.

## 🎯 The Scenario
You're building an **e-commerce checkout flow** with three services:
1. **API Gateway** - Receives the HTTP request
2. **Inventory Service** - Checks stock availability (HTTP call)
3. **Payment Service** - Processes payment (HTTP call)

When a checkout takes 5 seconds instead of 500ms, your distributed trace should show **exactly** where the time was spent: network latency, DB queries, external API calls, etc.

## 🛠 The Challenge
Implement a distributed tracing solution using OpenTelemetry that spans across multiple function calls and HTTP requests.

### 1. Functional Requirements
* [ ] Initialize an OpenTelemetry tracer with a Jaeger or OTLP exporter
* [ ] Create a root span for incoming HTTP requests
* [ ] Create child spans for downstream HTTP calls (with proper parent-child relationship)
* [ ] Propagate trace context via HTTP headers (W3C Trace Context)
* [ ] Add span attributes (e.g., `http.method`, `http.status_code`, user IDs)
* [ ] Record errors and exceptions in spans with proper status codes
* [ ] Gracefully shutdown tracer provider on application exit

### 2. The "Idiomatic" Constraints (Pass/Fail Criteria)
To pass this kata, you **must** strictly adhere to these rules:

* [ ] **Context is Mandatory:** Every span creation must use `context.Context` (not globals)
* [ ] **defer span.End():** All spans must be ended in `defer` blocks immediately after creation
* [ ] **Extract/Inject Pattern:** Use `propagation.TraceContext{}` to extract incoming context and inject into outgoing requests
* [ ] **Span Status on Error:** Set `span.SetStatus(codes.Error, msg)` when operations fail
* [ ] **Resource Detection:** Configure service name and version via `resource.NewWithAttributes()`
* [ ] **No Sampling in Code:** Sampling decisions should be in the collector/exporter config, not hardcoded
* [ ] **Graceful Shutdown:** Call `tracerProvider.Shutdown(ctx)` with a timeout context

### 3. Bonus Challenges (Optional)
* [ ] Add span events for significant operations (e.g., "cache hit", "retrying request")
* [ ] Implement automatic DB query tracing using a custom driver wrapper
* [ ] Add baggage propagation for tenant/user IDs across service boundaries
* [ ] Create custom span processors for filtering PII from attributes

## 🧪 Self-Correction (Test Yourself)

### Test 1: The "Orphaned Span"
```go
// BAD: This span won't connect to the parent
func handler(w http.ResponseWriter, r *http.Request) {
    ctx := context.Background() // ❌ Ignores incoming trace context!
    ctx, span := tracer.Start(ctx, "process-request")
    defer span.End()
}
```
**Pass Condition:** Use the request's context: `ctx := r.Context()`

### Test 2: The "Span Leak"
```go
// BAD: Span never ends if function panics
func riskyOperation(ctx context.Context) {
    ctx, span := tracer.Start(ctx, "risky")
    // If panic happens here, span.End() never called
    doSomethingDangerous()
    span.End()
}
```
**Pass Condition:** `defer span.End()` must be called immediately after `tracer.Start()`

### Test 3: The "Broken Chain"
* Send a request from Service A → Service B → Service C
* Check your tracing backend (Jaeger/Grafana Tempo)
* **Pass Condition:** All three spans should appear in the **same trace**, with proper parent-child relationships

### Test 4: The "Error Blindness"
```go
// BAD: Error happened but span shows success
if err != nil {
    return err // ❌ Span still has OK status
}
```
**Pass Condition:** Your span should have:
```go
span.RecordError(err)
span.SetStatus(codes.Error, "operation failed")
```

## 📚 Resources
* [OpenTelemetry Go SDK](https://github.com/open-telemetry/opentelemetry-go)
* [W3C Trace Context Spec](https://www.w3.org/TR/trace-context/)
* [OpenTelemetry Best Practices](https://opentelemetry.io/docs/instrumentation/go/manual/)
* [Jaeger Quickstart](https://www.jaegertracing.io/docs/latest/getting-started/)
* [Go Context Propagation](https://go.dev/blog/context)

## 💡 Key Patterns to Master
1. **Span Lifecycle:** `defer span.End()` is non-negotiable
2. **Context Flow:** `ctx` flows top-to-bottom through your call stack
3. **Extract Before Start:** Extract trace context from headers **before** starting root span
4. **Inject Before Send:** Inject trace context into headers **before** making HTTP calls
5. **Attribute Naming:** Use semantic conventions (`http.method`, not `method`)
6. **Shutdown with Timeout:** Always shutdown with a context timeout (5-10s)

## 🏗 Suggested Structure
```go
// 1. Initialize once at startup
func initTracer() (*sdktrace.TracerProvider, error) {
    // Configure exporter (Jaeger/OTLP)
    // Set resource (service.name, service.version)
    // Configure sampler
    // Return provider for graceful shutdown
}

// 2. Middleware for HTTP servers
func tracingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Extract context from headers
        // Start root span
        // defer span.End()
        // Inject context into request
        // Call next.ServeHTTP
        // Set span status based on response
    })
}

// 3. Client wrapper for HTTP calls
func doTracedRequest(ctx context.Context, req *http.Request) (*http.Response, error) {
    // Start child span
    // defer span.End()
    // Inject context into request headers
    // Make HTTP call
    // Record errors if any
}
```
