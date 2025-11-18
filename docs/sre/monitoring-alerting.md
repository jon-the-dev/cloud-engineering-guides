# Monitoring and Alerting Best Practices

[TOC]

## The Four Golden Signals

Every service should be monitored for these four key metrics:

### 1. Latency

**What:** Time taken to serve a request

**Why it matters:** Directly impacts user experience

**How to measure:**
```python
# Track percentiles, not just averages!
metrics = {
    'p50_latency': 100ms,   # Median
    'p95_latency': 200ms,   # 95th percentile
    'p99_latency': 500ms,   # 99th percentile
    'p99.9_latency': 1000ms # 99.9th percentile
}
```

**Pro Tip:** Always separate successful request latency from failed request latency. A failed request often fails fast, which can skew your averages.

### 2. Traffic

**What:** How much demand is on your system

**Metrics to track:**
- Requests per second (RPS)
- Transactions per second (TPS)
- Concurrent connections
- Bandwidth usage

**Example Alert:**
```yaml
alert: HighTraffic
expr: rate(http_requests_total[5m]) > 10000
for: 5m
annotations:
  summary: "Traffic spike detected: {{ $value }} req/s"
```

### 3. Errors

**What:** Rate of failed requests

**Types of errors to track:**
- HTTP 5xx errors (server errors)
- HTTP 4xx errors (client errors, track separately)
- Timeouts
- Connection failures
- Application exceptions

**Example Metrics:**
```promql
# Error rate as percentage
(sum(rate(http_requests_total{status=~"5.."}[5m])) /
 sum(rate(http_requests_total[5m]))) * 100

# Absolute error count
sum(rate(http_requests_total{status=~"5.."}[5m]))
```

### 4. Saturation

**What:** How "full" your service is

**Resources to monitor:**
- CPU utilization
- Memory usage
- Disk I/O
- Network bandwidth
- Database connections
- Queue depth
- Thread pool utilization

**Warning Signs:**
```yaml
# CPU saturation
cpu_usage > 80%

# Memory pressure
memory_usage > 85%

# Disk saturation
disk_io_util > 90%

# Queue backing up
queue_depth > max_processing_rate * 60  # More than 1 min of work queued
```

## Alerting Best Practices

### The Philosophy of Good Alerts

**Golden Rule:** Every alert should be actionable. If an alert fires and no one needs to do anything, it's not an alert—it's noise.

### Alert Design Principles

#### 1. Alert on Symptoms, Not Causes

**❌ Bad Alert:**
```yaml
alert: HighCPU
expr: cpu_usage > 80%
```

**✅ Good Alert:**
```yaml
alert: HighLatency
expr: p95_latency > 200ms AND cpu_usage > 80%
annotations:
  description: "High latency detected, possibly due to CPU saturation"
```

**Why:** CPU might be high, but if users aren't affected, you don't need to wake someone up.

#### 2. Use Multiple Severity Levels

```yaml
severity_levels:
  page:
    # Wake someone up
    description: "User-impacting, requires immediate action"
    examples:
      - Service completely down
      - Error rate > 10%
      - SLO burn rate critical
    response_time: < 5 minutes

  urgent:
    # Notify during business hours
    description: "Needs attention soon, may impact users"
    examples:
      - Error rate > 1%
      - Elevated latency
      - Error budget 75% consumed
    response_time: < 1 hour

  warning:
    # Ticket for follow-up
    description: "Should be investigated, not urgent"
    examples:
      - Error rate > 0.1%
      - Resource usage trending up
      - Error budget 50% consumed
    response_time: < 1 day
```

#### 3. Include Runbook Links

Every alert should point to a runbook explaining what to do:

```yaml
alert: DatabaseConnectionPoolExhausted
expr: db_connection_pool_usage > 95%
for: 5m
labels:
  severity: page
annotations:
  summary: "Database connection pool nearly exhausted"
  description: "{{ $value }}% of DB connections in use on {{ $labels.instance }}"
  runbook_url: "https://wiki.company.com/runbooks/db-connection-pool"
  dashboard: "https://grafana.company.com/d/db-health"
```

#### 4. Avoid Alert Fatigue

**Symptoms of alert fatigue:**
- Alerts being ignored or silenced without investigation
- Alerts that fire constantly
- Alerts with no clear action
- Duplicate alerts for the same issue

**Solutions:**
```yaml
# Use 'for' clause to avoid flapping
alert: HighErrorRate
expr: error_rate > 5%
for: 10m  # Must be true for 10 minutes before alerting

# Group related alerts
alert: ServiceDegraded
expr: |
  (
    error_rate > 1% OR
    p95_latency > 500ms OR
    availability < 99%
  )
# This creates ONE alert instead of three
```

## Monitoring Strategy

### The RED Method (for request-driven services)

**R**ate - requests per second
**E**rrors - number of failed requests
**D**uration - distribution of request latency

```promql
# Rate
sum(rate(http_requests_total[5m])) by (service)

# Errors
sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)

# Duration
histogram_quantile(0.95,
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service)
)
```

### The USE Method (for resources)

**U**tilization - % time resource is busy
**S**aturation - amount of work resource can't process (queued)
**E**rrors - error events

```promql
# CPU Utilization
100 - (avg(irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Memory Saturation (swapping indicates saturation)
rate(node_vmstat_pswpin[5m]) + rate(node_vmstat_pswpout[5m])

# Disk Errors
rate(node_disk_io_errors_total[5m])
```

## Dashboard Design

### Dashboard Hierarchy

Create dashboards at different levels:

```
1. Executive Dashboard (Business Level)
   ├─ Overall availability %
   ├─ Revenue impact
   ├─ SLO compliance
   └─ Major incidents

2. Service Overview (Service Level)
   ├─ Four Golden Signals
   ├─ SLI metrics
   ├─ Dependency health
   └─ Recent deployments

3. Detailed Metrics (Component Level)
   ├─ Resource utilization
   ├─ Database performance
   ├─ Cache hit rates
   └─ Queue depths

4. Debug Dashboard (Investigation)
   ├─ Detailed traces
   ├─ Log aggregation
   ├─ Profiling data
   └─ System calls
```

### Dashboard Best Practices

#### ✅ Do's

1. **Put the most important graphs at the top**
2. **Use consistent color schemes**
   - Green = good
   - Yellow = warning
   - Red = critical
3. **Include time range selector**
4. **Add annotations for deployments and incidents**
5. **Use meaningful graph titles**
6. **Include links to runbooks and related dashboards**

#### ❌ Don'ts

1. **Don't overload a single dashboard** (max 12-15 graphs)
2. **Don't use pie charts for time-series data**
3. **Don't use default titles** like "Graph 1"
4. **Don't forget to set appropriate Y-axis ranges**
5. **Don't create dashboards without context/descriptions**

### Example Dashboard Layout

```yaml
Dashboard: User API Health
Sections:
  - name: "At a Glance"
    panels:
      - Current SLO Compliance (gauge)
      - Error Budget Remaining (bar)
      - Current RPS (stat)
      - Active Alerts (table)

  - name: "Golden Signals"
    panels:
      - Latency Percentiles (graph)
      - Request Rate (graph)
      - Error Rate (graph)
      - Saturation Metrics (graph)

  - name: "Dependencies"
    panels:
      - Database Latency (graph)
      - Cache Hit Rate (graph)
      - External API Health (status map)

  - name: "Recent Events"
    panels:
      - Deployment Timeline (annotations)
      - Recent Alerts (table)
```

## Logging Best Practices

### Structured Logging

**❌ Bad: Unstructured logs**
```python
logger.info(f"User {user_id} logged in from {ip_address}")
```

**✅ Good: Structured logs**
```python
logger.info("user_login", extra={
    "user_id": user_id,
    "ip_address": ip_address,
    "timestamp": datetime.utcnow(),
    "session_id": session_id,
    "user_agent": request.headers.get("User-Agent")
})
```

**Why:** Structured logs are easily searchable, parseable, and can be aggregated.

### Log Levels

Use log levels appropriately:

```python
# DEBUG: Detailed diagnostic information
logger.debug("Cache lookup", extra={"key": cache_key, "hit": True})

# INFO: General informational messages
logger.info("Request processed", extra={"user_id": user_id, "duration_ms": 150})

# WARNING: Something unexpected, but service continues
logger.warning("API rate limit approaching", extra={"current_rate": 950, "limit": 1000})

# ERROR: Error occurred, but service recovers
logger.error("Failed to send notification email", extra={"user_id": user_id, "error": str(e)})

# CRITICAL: Serious error, service may be unstable
logger.critical("Database connection pool exhausted", extra={"pool_size": 100, "active": 100})
```

### What to Log

**✅ Do Log:**
- Request/response metadata (user ID, endpoint, status, duration)
- State changes (order created, payment processed, user registered)
- External API calls (which API, duration, status)
- Authentication events (login, logout, failed attempts)
- Errors and exceptions (with stack traces)
- Performance metrics (query times, cache hits/misses)

**❌ Don't Log:**
- Passwords or secrets
- Credit card numbers or PII
- API keys or tokens
- Session IDs in plain text
- Full request/response bodies (unless debugging)

### Log Aggregation

Centralize logs from all services:

```
Application Logs ──┐
Container Logs ────┼──> Log Shipper ──> Aggregation ──> Storage ──> Search/Analysis
System Logs ───────┘    (Fluentd)       (Kafka)        (ES)        (Kibana)
```

## Distributed Tracing

### When to Use Tracing

Tracing is essential for:
- Microservices architectures
- Complex distributed systems
- Performance debugging
- Understanding request flows

### Implementing Tracing

```python
from opentelemetry import trace
from opentelemetry.instrumentation.flask import FlaskInstrumentor

# Initialize tracer
tracer = trace.get_tracer(__name__)

@app.route("/api/user/<user_id>")
def get_user(user_id):
    with tracer.start_as_current_span("get_user") as span:
        span.set_attribute("user.id", user_id)

        # Database call
        with tracer.start_as_current_span("db.query") as db_span:
            db_span.set_attribute("db.system", "postgresql")
            db_span.set_attribute("db.statement", "SELECT * FROM users WHERE id = ?")
            user = db.get_user(user_id)

        # Cache check
        with tracer.start_as_current_span("cache.check") as cache_span:
            cache_span.set_attribute("cache.hit", False)
            # Cache logic here

        return jsonify(user)
```

### Trace Sampling

Don't trace every request—it's expensive:

```python
sampling_strategies = {
    # Always trace errors
    "error_requests": 1.0,  # 100%

    # Sample successful requests
    "successful_requests": 0.01,  # 1%

    # Higher sampling for slow requests
    "slow_requests": 0.5,  # 50% if duration > threshold

    # Lower sampling for health checks
    "health_checks": 0.001  # 0.1%
}
```

## Common Monitoring Anti-Patterns

### 1. Monitoring Everything

**Problem:** Too many metrics = high cost, slow queries, alert fatigue

**Solution:** Focus on signals that indicate user impact

### 2. Only Monitoring Infrastructure

**Problem:** CPU/memory/disk are fine, but users can't log in

**Solution:** Monitor user-facing functionality (black-box monitoring)

### 3. No Baseline Metrics

**Problem:** Is 1000 req/s high or low? Can't tell without context

**Solution:** Establish baselines and track trends over time

### 4. Ignoring Alert Fatigue

**Problem:** Team ignores alerts because too many are false positives

**Solution:** Regular alert hygiene—remove/tune noisy alerts

### 5. Dashboard Sprawl

**Problem:** 50+ dashboards, no one knows which to use

**Solution:** Organize hierarchically, deprecate unused dashboards

## Monitoring Tools Comparison

| Tool | Best For | Pros | Cons |
|------|----------|------|------|
| **Prometheus + Grafana** | Open-source, Kubernetes | Free, flexible, great community | Self-hosted, requires expertise |
| **Datadog** | Full-stack observability | Easy setup, great UX | Expensive at scale |
| **New Relic** | APM, distributed tracing | Excellent APM features | Can be complex, costly |
| **CloudWatch** | AWS-native services | Native AWS integration | Limited features, AWS-only |
| **Splunk** | Log analysis, security | Powerful search, enterprise features | Very expensive |
| **Elastic Stack** | Log aggregation | Flexible, powerful search | Resource-intensive, complex |

## Key Takeaways

1. **Start with the Four Golden Signals:** Latency, Traffic, Errors, Saturation
2. **Alert on Symptoms:** Don't wake people up for infrastructure metrics unless users are affected
3. **Every Alert Must Be Actionable:** If you can't do anything about it, don't alert on it
4. **Use Structured Logging:** Makes debugging 10x easier
5. **Trace Selectively:** Full tracing is expensive; sample intelligently
6. **Dashboard Hierarchy:** Executive → Service → Component → Debug
7. **Fight Alert Fatigue:** Regular alert hygiene is critical
8. **Monitor User Experience:** Not just server health
