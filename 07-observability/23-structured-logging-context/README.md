# Kata 23: Context-Aware Structured Logging with slog

**Target Idioms:** log/slog, Context Propagation, Log Correlation, Zero-Allocation Logging
**Difficulty:** 🟡 Intermediate

## 🧠 The "Why"
Developers from other ecosystems often use `fmt.Printf` or global logger instances. In Go 1.21+, **log/slog** is the standard for structured logging. Common mistakes include:
- Not using structured key-value pairs (makes logs unsearchable)
- Creating new logger instances per-request (performance hit)
- Not propagating request/trace IDs through the call stack
- Missing critical context when debugging production issues

**Idiomatic Go logging** means: structured fields, context propagation, and log levels that can be changed without code changes.

## 🎯 The Scenario
You're maintaining a **multi-tenant SaaS API**. When debugging issues, you need to:
- Filter logs by specific tenant ID
- Correlate logs with distributed traces
- Track request flow across multiple goroutines
- Differentiate between production errors and debug noise

Without proper structured logging, your `grep` commands become nightmares, and finding the root cause takes hours instead of minutes.

## 🛠 The Challenge
Build a context-aware logging system using `log/slog` that enriches logs with trace IDs, user IDs, and tenant IDs.

### 1. Functional Requirements
* [ ] Create a logger with JSON output format (for production) and text format (for development)
* [ ] Support dynamic log levels (INFO, WARN, ERROR, DEBUG)
* [ ] Automatically add request ID to all logs within a request
* [ ] Automatically add trace/span IDs from OpenTelemetry context (if present)
* [ ] Support tenant/user ID propagation via context
* [ ] Create a helper to extract logger from context with pre-configured fields
* [ ] Add source location (file:line) for ERROR and above levels

### 2. The "Idiomatic" Constraints (Pass/Fail Criteria)
To pass this kata, you **must** strictly adhere to these rules:

* [ ] **Use log/slog Only:** No third-party logging libraries (zap, zerolog, logrus)
* [ ] **Context Propagation:** Store logger in `context.Context`, not global variables
* [ ] **Structured Fields:** All logs must use key-value pairs (e.g., `slog.String("user_id", id)`)
* [ ] **Custom Attributes:** Use `slog.LogAttrs()` for zero-allocation logging in hot paths
* [ ] **Log Levels:** Support environment-based level configuration (e.g., `LOG_LEVEL=debug`)
* [ ] **No String Concatenation:** Never use `fmt.Sprintf` to build log messages
* [ ] **Handler Composition:** Use custom `slog.Handler` for adding trace context automatically

### 3. Bonus Challenges (Optional)
* [ ] Implement log sampling (only log 1 in N debug messages under high load)
* [ ] Add caller information (function name) for specific log levels
* [ ] Create a middleware that logs request start/finish with duration
* [ ] Implement log redaction for sensitive fields (PII, passwords)
* [ ] Add support for log correlation across goroutines

## 🧪 Self-Correction (Test Yourself)

### Test 1: The "Context Loss"
```go
// BAD: Starting a new goroutine loses request context
func handler(ctx context.Context) {
    logger := LoggerFromContext(ctx)
    
    go func() {
        // ❌ logger here doesn't have request_id!
        logger.Info("background job started")
    }()
}
```
**Pass Condition:** Pass context to goroutine:
```go
go func(ctx context.Context) {
    logger := LoggerFromContext(ctx)
    logger.Info("background job started") // ✅ Has request_id
}(ctx)
```

### Test 2: The "Allocation Storm"
```go
// BAD: Creates allocations on every log
logger.Info("user logged in",
    slog.String("user_id", userID),
    slog.String("tenant_id", tenantID),
    slog.Time("timestamp", time.Now()),
)
```
**Pass Condition:** Use `LogAttrs` in hot paths:
```go
logger.LogAttrs(ctx, slog.LevelInfo, "user logged in",
    slog.String("user_id", userID),
    slog.String("tenant_id", tenantID),
)
```

### Test 3: The "Missing Correlation"
* Generate a request ID in middleware
* Make 5 downstream function calls
* **Pass Condition:** All logs should contain the **same** request_id, even in different functions

### Test 4: The "Production Noise"
* Set log level to INFO in production
* Call `logger.Debug("debugging stuff")`
* **Pass Condition:** Debug logs should NOT appear in production output

## 📚 Resources
* [log/slog Official Docs](https://pkg.go.dev/log/slog)
* [slog Design Proposal](https://go.dev/blog/slog)
* [Structured Logging Best Practices](https://www.honeycomb.io/blog/structured-logging-best-practices)
* [OpenTelemetry + slog Integration](https://github.com/open-telemetry/opentelemetry-go-contrib/tree/main/bridges/otelslog)

## 💡 Key Patterns to Master
1. **Logger in Context:** Store logger with enriched fields in `context.Context`
2. **Handler Chaining:** Wrap `slog.Handler` to automatically add trace IDs
3. **Attribute Reuse:** Create constant attributes for high-cardinality fields
4. **Level Guards:** Use `logger.Enabled(level)` to avoid expensive operations
5. **Group Fields:** Use `slog.Group()` to namespace related fields

## 🏗 Suggested Structure
```go
// 1. Initialize logger at startup
func NewLogger(level slog.Level, format string) *slog.Logger {
    var handler slog.Handler
    if format == "json" {
        handler = slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
            Level: level,
            AddSource: true,
        })
    } else {
        handler = slog.NewTextHandler(os.Stdout, &slog.HandlerOptions{
            Level: level,
        })
    }
    
    // Wrap with trace context handler
    handler = NewTraceContextHandler(handler)
    return slog.New(handler)
}

// 2. Custom handler for trace context
type TraceContextHandler struct {
    slog.Handler
}

func (h *TraceContextHandler) Handle(ctx context.Context, r slog.Record) error {
    // Extract trace ID from context
    // Add as attribute to record
    // Call underlying handler
}

// 3. Context helpers
type contextKey string

const loggerKey contextKey = "logger"

func WithLogger(ctx context.Context, logger *slog.Logger) context.Context {
    return context.WithValue(ctx, loggerKey, logger)
}

func LoggerFromContext(ctx context.Context) *slog.Logger {
    if logger, ok := ctx.Value(loggerKey).(*slog.Logger); ok {
        return logger
    }
    return slog.Default() // Fallback
}

// 4. Middleware for request enrichment
func LoggingMiddleware(logger *slog.Logger) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            requestID := GenerateRequestID()
            
            // Create request-scoped logger
            reqLogger := logger.With(
                slog.String("request_id", requestID),
                slog.String("method", r.Method),
                slog.String("path", r.URL.Path),
            )
            
            // Store in context
            ctx := WithLogger(r.Context(), reqLogger)
            r = r.WithContext(ctx)
            
            reqLogger.Info("request started")
            next.ServeHTTP(w, r)
            reqLogger.Info("request completed")
        })
    }
}
```

## 🎯 Expected Output Format
```json
{
  "time": "2025-01-05T10:30:15.123Z",
  "level": "INFO",
  "msg": "payment processed",
  "request_id": "req-abc123",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "span_id": "00f067aa0ba902b7",
  "user_id": "user-456",
  "tenant_id": "tenant-789",
  "amount": 99.99,
  "currency": "USD",
  "source": "payment/handler.go:42"
}
```
