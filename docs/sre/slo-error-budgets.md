# SLOs and Error Budgets - Tips & Tricks

[TOC]

## Understanding SLOs

### What Makes a Good SLO?

A good Service Level Objective should be:

- **Measurable:** Based on concrete metrics you can track
- **User-Centric:** Reflect what users actually care about
- **Achievable:** Realistic given your current system
- **Actionable:** Drive clear decisions when violated

### SLO vs SLA vs SLI

| Term | Definition | Example |
|------|------------|---------|
| **SLI** (Service Level Indicator) | The actual measurement | "99.9% of requests succeeded in last 30 days" |
| **SLO** (Service Level Objective) | Internal target | "99.9% of requests should succeed" |
| **SLA** (Service Level Agreement) | External commitment with consequences | "We guarantee 99.5% uptime or you get a refund" |

**Tip:** Always set your SLO higher than your SLA to give yourself buffer room!

## Choosing the Right SLIs

### Common SLI Categories

1. **Availability:** Is the service up and responding?
2. **Latency:** How fast does the service respond?
3. **Throughput:** How many requests can the service handle?
4. **Correctness:** Are responses accurate?
5. **Durability:** Is data being persisted reliably?

### Example SLIs for Different Service Types

#### User-Facing Service

```yaml
SLIs:
  - Availability: Percentage of successful HTTP requests (200-299 status codes)
  - Latency: 95th percentile of request duration < 200ms
  - Error Rate: Percentage of 5xx errors < 0.1%
```

#### Data Pipeline

```yaml
SLIs:
  - Freshness: Data processing lag < 5 minutes
  - Completeness: Percentage of records processed successfully
  - Correctness: Percentage of records passing validation
```

#### Storage Service

```yaml
SLIs:
  - Availability: Percentage of successful read/write operations
  - Durability: Zero data loss events
  - Latency: 99th percentile read latency < 10ms
```

## Error Budgets

### What is an Error Budget?

An error budget is the maximum amount of time your service can be unavailable (or degraded) before violating your SLO.

**Formula:**
```
Error Budget = 100% - SLO
```

**Example:**
- SLO: 99.9% availability
- Error Budget: 0.1% unavailable time
- Per month (30 days): 43.2 minutes of downtime allowed

### Error Budget Policy

Create a clear policy for what happens when you're close to or exceed your error budget:

```markdown
Error Budget Policy:

1. **Budget > 50% remaining:**
   - Normal feature development velocity
   - Take calculated risks with new features
   - Experimental deployments allowed

2. **Budget 25-50% remaining:**
   - Review recent incidents
   - Slow down feature releases
   - Focus on stability improvements
   - Increase test coverage

3. **Budget < 25% remaining:**
   - Freeze non-critical feature releases
   - All hands on reliability improvements
   - Mandatory code reviews for all changes
   - No experimental features

4. **Budget exhausted:**
   - Complete feature freeze
   - Only SRE-approved changes
   - Focus 100% on reliability
   - Executive stakeholder notification
```

## Practical Tips

### Tip 1: Start Conservative

When first implementing SLOs, start with achievable targets based on current performance, then tighten them gradually.

```python
# Bad: Aspirational SLO without baseline
slo_target = 99.99  # "Four nines sounds good!"

# Good: Based on actual current performance
current_availability = calculate_last_90_days()  # Returns 99.7%
slo_target = 99.5  # Conservative, achievable starting point
```

### Tip 2: Use Multiple SLIs

Don't rely on a single metric. Combine multiple SLIs for a complete picture:

```yaml
Service Health =
  AND(
    availability >= 99.9%,
    p95_latency <= 200ms,
    p99_latency <= 500ms,
    error_rate <= 0.1%
  )
```

### Tip 3: Window-Based vs Request-Based

Choose the right SLO measurement approach:

**Request-Based (Good for high-traffic services):**
```
SLO: 99.9% of requests in the last 30 days were successful
```

**Window-Based (Good for low-traffic services):**
```
SLO: Service is available 99.9% of the time in rolling 30-day windows
```

### Tip 4: Set Up Error Budget Alerts

Don't wait until your budget is exhausted. Set up proactive alerts:

```yaml
alerts:
  - name: "Error Budget 50% Consumed"
    condition: error_budget_remaining < 0.5
    severity: warning
    notification: slack

  - name: "Error Budget 75% Consumed"
    condition: error_budget_remaining < 0.25
    severity: high
    notification: [slack, email, pagerduty]

  - name: "Error Budget 90% Consumed"
    condition: error_budget_remaining < 0.1
    severity: critical
    notification: [slack, email, pagerduty, executive_escalation]
```

### Tip 5: Document Your SLO Calculations

Make SLO calculations transparent and reviewable:

```python
"""
SLO Calculation for User API Service

SLI: Availability of API requests
Measurement: Ratio of successful requests to total requests
Success Criteria: HTTP status codes 200-299 and 4xx
Failure Criteria: HTTP status codes 5xx, timeouts, connection errors
Time Window: Rolling 30 days
Target: 99.9%
Error Budget: 43.2 minutes per 30 days

Data Source: Prometheus metric 'http_requests_total'
Query:
  sum(rate(http_requests_total{status=~"[2-4].*"}[30d])) /
  sum(rate(http_requests_total[30d]))
"""

def calculate_slo():
    successful = query_prometheus(
        'sum(rate(http_requests_total{status=~"[2-4].*"}[30d]))'
    )
    total = query_prometheus(
        'sum(rate(http_requests_total[30d]))'
    )
    return (successful / total) * 100
```

## Common Pitfalls to Avoid

### ❌ Don't: Set Too Many SLOs

**Bad:** 15 different SLOs for a single service
**Good:** 3-5 key SLOs that matter most to users

### ❌ Don't: Use Internal Metrics Only

**Bad:** SLO based on container CPU usage
**Good:** SLO based on user-experienced latency

### ❌ Don't: Ignore Your Error Budget

**Bad:** Exceed error budget but keep shipping features anyway
**Good:** Use error budget to guide release decisions

### ❌ Don't: Make SLOs Invisible

**Bad:** SLOs documented in a wiki no one reads
**Good:** SLO dashboards visible to entire team, reviewed weekly

## SLO Review Cadence

Establish a regular review schedule:

```markdown
Weekly:
- Review current SLO compliance
- Check error budget burn rate
- Assess impact of recent changes

Monthly:
- Deep dive into SLO violations
- Adjust error budget policy if needed
- Review whether SLOs still reflect user needs

Quarterly:
- Re-evaluate SLO targets
- Update SLIs based on service evolution
- Align SLOs with business objectives
```

## Tools for SLO Management

- **Prometheus + Grafana:** Open-source metrics and dashboards
- **Datadog SLOs:** Built-in SLO tracking and alerting
- **Google Cloud Operations (formerly Stackdriver):** Native SLO support
- **Nobl9:** Dedicated SLO platform
- **Sloth:** SLO generator for Prometheus

## Example: Complete SLO Implementation

```yaml
service: user-authentication-api
owner: identity-team@company.com

slos:
  - name: API Availability
    sli:
      metric: http_requests_total
      success_criteria: status_code in [200-299, 401, 403, 404]
      failure_criteria: status_code in [500-599], timeout, connection_error
    objective: 99.9
    window: 30d

  - name: API Latency
    sli:
      metric: http_request_duration_seconds
      percentile: 95
      success_criteria: duration < 0.2
    objective: 99.0
    window: 30d

  - name: Token Generation Success
    sli:
      metric: token_generation_total
      success_criteria: status = "success"
      failure_criteria: status in ["error", "timeout"]
    objective: 99.95
    window: 30d

error_budget_policy:
  remaining_above_50_percent:
    - normal_feature_velocity
    - experimental_changes_allowed
  remaining_25_to_50_percent:
    - slow_feature_releases
    - increase_testing
  remaining_below_25_percent:
    - freeze_non_critical_features
    - focus_on_reliability
  budget_exhausted:
    - complete_feature_freeze
    - sre_approval_required
    - executive_notification

alerts:
  - condition: error_budget_remaining < 0.5
    severity: warning
  - condition: error_budget_remaining < 0.25
    severity: high
  - condition: error_budget_remaining < 0.1
    severity: critical
```

## Key Takeaways

1. **Start Simple:** Begin with 2-3 key SLOs, expand as needed
2. **User Focus:** Always ask "Does this metric matter to users?"
3. **Use Error Budgets:** Let data drive feature vs. reliability trade-offs
4. **Review Regularly:** SLOs should evolve with your service
5. **Make it Visible:** Dashboard everything, communicate widely
6. **Enforce Policies:** Error budget policies only work if you follow them
