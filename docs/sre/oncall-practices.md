# On-Call Best Practices

[TOC]

## The Philosophy of On-Call

Being on-call means you're the first responder when systems fail. Your job is to:

1. **Respond quickly** to pages and alerts
2. **Stabilize the service** and restore it to normal operation
3. **Communicate clearly** with stakeholders
4. **Document incidents** for future learning
5. **Prevent recurrence** through follow-up work

**The Golden Rule:** You should be able to hand off an incident to anyone on the team at any time with complete context.

## On-Call Rotation Design

### Rotation Patterns

#### 1. Primary + Secondary

```yaml
rotation_pattern: "Primary + Secondary"

schedule:
  primary:
    - week_1: Alice
    - week_2: Bob
    - week_3: Charlie

  secondary:
    - week_1: Bob      # Backs up Alice
    - week_2: Charlie  # Backs up Bob
    - week_3: Alice    # Backs up Charlie

escalation:
  - level_1: Primary (0-5 min)
  - level_2: Secondary (5-10 min)
  - level_3: Team Lead (10-15 min)
  - level_4: Engineering Manager (15+ min)
```

**Pros:** Backup coverage, mentorship opportunities
**Cons:** Two people need to be available

#### 2. Follow-the-Sun

```yaml
rotation_pattern: "Follow-the-Sun"

schedule:
  # Handoff every 8 hours based on timezone
  00:00-08:00_UTC: APAC_team  # Sydney, Singapore
  08:00-16:00_UTC: EMEA_team  # London, Berlin
  16:00-00:00_UTC: Americas_team  # SF, NYC

benefits:
  - No one gets paged at night
  - Fresh eyes on each shift
  - Better work-life balance
```

**Pros:** No nighttime pages, global coverage
**Cons:** Requires global team, handoff overhead

#### 3. Quarterly Rotation

```yaml
rotation_pattern: "Quarterly with Blackout Dates"

q1_2024:
  jan: Alice
  feb: Bob
  mar: Charlie

blackout_dates:
  - Alice: ["2024-01-15 to 2024-01-20"]  # Vacation
  - Bob: ["2024-02-10 to 2024-02-12"]    # Conference

swap_process:
  - Request swap at least 1 week in advance
  - Find your own coverage
  - Update PagerDuty schedule
  - Notify team in #oncall channel
```

### Rotation Best Practices

**✅ Do's:**
- Rotate regularly (weekly or bi-weekly)
- Give at least 48 hours notice before shift starts
- Allow schedule swaps with proper notification
- Honor blackout dates (vacation, conferences, personal)
- Ensure at least 2 people know how to handle critical issues

**❌ Don'ts:**
- Don't have same person on-call for more than 2 weeks straight
- Don't schedule new team members alone for first rotation
- Don't rotate too frequently (daily rotations are exhausting)
- Don't forget to account for timezones
- Don't make on-call mandatory during major life events

## On-Call Responsibilities

### During Your Shift

```markdown
☐ Acknowledge pages within 5 minutes
☐ Respond to incidents following incident response process
☐ Keep #oncall channel updated
☐ Escalate if you need help
☐ Document all actions taken
☐ Hand off active incidents at shift change
☐ Update runbooks with any new learnings
```

### Before Your Shift

```markdown
☐ Review recent incidents and ongoing issues
☐ Check for scheduled maintenance or deployments
☐ Test your alerting (phone, SMS, app)
☐ Ensure you have access to all necessary systems
☐ Review runbooks and escalation procedures
☐ Know who your backup is
☐ Be in a location with internet access
```

### After Your Shift

```markdown
☐ Hand off any ongoing incidents
☐ Document unresolved issues
☐ Update team on shift summary
☐ File tickets for follow-up work
☐ Update runbooks if gaps were found
☐ Provide feedback on alerts (too noisy? actionable?)
```

## Alert Quality

### The Anatomy of a Good Alert

```yaml
alert: DatabaseConnectionPoolNearExhaustion
severity: warning
description: |
  Database connection pool is {{ $value }}% full on {{ $labels.instance }}.
  This may lead to connection errors if pool becomes exhausted.

impact: |
  - Users may experience "cannot connect to database" errors
  - New requests may be rejected
  - Application may become unresponsive

runbook: https://wiki.company.com/runbooks/db-connection-pool
dashboard: https://grafana.company.com/d/database-health

thresholds:
  warning: 80%   # Alert but don't page
  critical: 95%  # Page on-call

suggested_actions:
  - Check for connection leaks in application logs
  - Review slow query log for blocking queries
  - Consider temporarily increasing pool size
  - Identify and kill long-running transactions if safe

escalation:
  - Primary: 0-5 minutes
  - Secondary: 5-10 minutes
  - Database team: 10-15 minutes
```

### Alert Fatigue Prevention

#### Symptom: Too Many Alerts

```python
# Calculate alert noise
def analyze_alert_quality(alerts_last_30_days):
    """Identify problematic alerts"""

    alerts_analysis = {
        "total_alerts": len(alerts_last_30_days),
        "false_positives": 0,
        "no_action_taken": 0,
        "duplicates": 0,
        "flapping": 0,
    }

    for alert in alerts_last_30_days:
        # Alert that resolved itself without action
        if alert.resolved_without_action():
            alerts_analysis["no_action_taken"] += 1

        # Alert fired but was incorrect
        if alert.was_false_positive():
            alerts_analysis["false_positives"] += 1

        # Alert fired multiple times for same issue
        if alert.is_duplicate_of_recent():
            alerts_analysis["duplicates"] += 1

        # Alert flapping (firing and resolving repeatedly)
        if alert.flapping_count() > 3:
            alerts_analysis["flapping"] += 1

    # Calculate noise percentage
    noise = (
        alerts_analysis["false_positives"] +
        alerts_analysis["no_action_taken"] +
        alerts_analysis["duplicates"] +
        alerts_analysis["flapping"]
    )

    alerts_analysis["noise_percentage"] = (
        noise / alerts_analysis["total_alerts"] * 100
    )

    # Target: <10% noise
    alerts_analysis["quality_score"] = (
        100 - alerts_analysis["noise_percentage"]
    )

    return alerts_analysis


# Example output:
# {
#   "total_alerts": 450,
#   "false_positives": 25,
#   "no_action_taken": 60,
#   "duplicates": 30,
#   "flapping": 15,
#   "noise_percentage": 28.9%,
#   "quality_score": 71.1
# }
# Action: Need to improve alerts - too much noise!
```

#### Solutions for Alert Fatigue

**1. Adjust Thresholds**
```yaml
# Too sensitive (pages every night)
alert: HighCPU
expr: cpu_usage > 50%

# Better (only alerts when actually problematic)
alert: HighCPU
expr: cpu_usage > 80%
for: 15m  # Must be sustained
```

**2. Use Alert Grouping**
```yaml
# Before: 10 alerts for 10 unhealthy instances
alert: InstanceUnhealthy
for_each: instance

# After: 1 alert for cluster degradation
alert: ClusterDegraded
expr: (unhealthy_instances / total_instances) > 0.2
```

**3. Add "for" Clause**
```promql
# Prevents flapping alerts
alert: HighErrorRate
expr: error_rate > 5%
for: 10m  # Must be true for 10 minutes
```

**4. Time-Based Muting**
```yaml
# Don't page for expected batch job load
alert: HighDatabaseLoad
expr: db_load > 80%
mute_windows:
  - start: "02:00"
    end: "04:00"
    days: ["Mon", "Wed", "Fri"]
    reason: "Nightly ETL job"
```

## On-Call Handbook

### Quick Response Guide

```markdown
┌─────────────────────────────────────────────────────┐
│ ON-CALL QUICK RESPONSE                              │
├─────────────────────────────────────────────────────┤
│ STEP 1: ACKNOWLEDGE (within 5 minutes)              │
│   - Ack the alert in PagerDuty                      │
│   - Check #incidents channel                        │
│   - Open monitoring dashboards                      │
│                                                      │
│ STEP 2: ASSESS SEVERITY                             │
│   - How many users affected?                        │
│   - Is data at risk?                                │
│   - Are we violating SLOs/SLAs?                     │
│   - Determine SEV level (1-5)                       │
│                                                      │
│ STEP 3: COMMUNICATE                                 │
│   - Create #incident-YYYY-MM-DD-description         │
│   - Post initial status update                      │
│   - Update status page if user-facing               │
│   - Escalate if needed                              │
│                                                      │
│ STEP 4: INVESTIGATE                                 │
│   - Check runbook for this alert                    │
│   - Review recent changes/deployments               │
│   - Examine logs and metrics                        │
│   - Test hypothesis                                 │
│                                                      │
│ STEP 5: MITIGATE                                    │
│   - Apply fix (rollback, scale, restart, etc.)      │
│   - Monitor for improvement                         │
│   - Don't hesitate to escalate                      │
│                                                      │
│ STEP 6: DOCUMENT                                    │
│   - Timeline of events                              │
│   - Actions taken                                   │
│   - What worked, what didn't                        │
│   - Follow-up tasks                                 │
│                                                      │
│ REMEMBER: Ask for help early and often!             │
└─────────────────────────────────────────────────────┘
```

### Common On-Call Scenarios

#### Scenario 1: Alert Fires, Service Seems Fine

```markdown
**What happened:**
Alert fired for high error rate, but dashboard shows everything normal.

**What to do:**
1. ✅ Don't immediately dismiss - investigate
2. ✅ Check if alert is looking at correct metrics
3. ✅ Look for intermittent issues (may have self-resolved)
4. ✅ Check if alert threshold is too sensitive
5. ✅ Document findings
6. ✅ Create ticket to fix or remove alert

**What NOT to do:**
❌ Silence alert without investigation
❌ Assume alert is broken without verification
```

#### Scenario 2: Multiple Alerts Firing

```markdown
**What happened:**
10 different alerts firing simultaneously.

**What to do:**
1. ✅ Find the root cause alert (others are likely symptoms)
2. ✅ Check infrastructure first (network, cloud provider)
3. ✅ Look for recent changes (deployment, config change)
4. ✅ Focus on user-impacting issues first
5. ✅ Escalate if overwhelmed

**Priority order:**
1. User-facing services
2. Data integrity issues
3. Internal tools
4. Infrastructure alerts
```

#### Scenario 3: Incident During Off-Hours

```markdown
**What happened:**
Paged at 3 AM for production issue.

**What to do:**
1. ✅ Acknowledge page within 5 minutes
2. ✅ Assess if it needs immediate action or can wait
3. ✅ If immediate: follow incident response process
4. ✅ If can wait: document assessment, schedule for morning
5. ✅ Communicate decision in #oncall channel

**Mental health check:**
- You're not expected to be at 100% at 3 AM
- It's OK to escalate if you're not confident
- It's OK to bring in backup if needed
```

#### Scenario 4: Alert You Don't Understand

```markdown
**What happened:**
Alert fired for a system you're not familiar with.

**What to do:**
1. ✅ Check runbook (should explain alert)
2. ✅ Look for recent incidents with same alert
3. ✅ Post in #oncall: "Need help with [alert_name]"
4. ✅ Escalate to service owner
5. ✅ Document that runbook needs improvement

**What NOT to do:**
❌ Ignore the alert
❌ Try random fixes without understanding
❌ Suffer in silence
```

## On-Call Tools

### Essential Toolkit

```markdown
**Alerting/Paging:**
- PagerDuty / Opsgenie / VictorOps
- Mobile app + SMS + phone call backup
- Test alerts weekly

**Monitoring:**
- Grafana / Datadog / New Relic dashboards
- Mobile app for on-the-go viewing
- Pre-built dashboard links in runbooks

**Communication:**
- Slack / Teams for incident channels
- Zoom / Google Meet for war rooms
- Status page for customer updates

**Documentation:**
- Runbooks (searchable, up-to-date)
- Incident response templates
- Architecture diagrams

**Access:**
- VPN (if needed)
- AWS/GCP/Azure console access
- SSH keys / bastions
- Database query tools
- Feature flag controls
```

### On-Call Laptop Setup

```bash
# Quick setup script for on-call laptop
cat > ~/.oncall_setup.sh << 'EOF'
#!/bin/bash

# Essential tools for on-call
brew install --cask \
  slack \
  pagerduty \
  zoom \
  iterm2

brew install \
  awscli \
  kubectl \
  postgresql \
  jq \
  httpie

# SSH keys
if [ ! -f ~/.ssh/id_rsa ]; then
  echo "⚠️  Set up SSH keys for production access"
fi

# VPN
if [ ! -d "/Applications/Cisco AnyConnect.app" ]; then
  echo "⚠️  Install VPN client"
fi

# Test alerts
echo "Testing PagerDuty alerts..."
# (Add PagerDuty test alert command)

echo "✅ On-call setup complete!"
EOF

chmod +x ~/.oncall_setup.sh
```

## On-Call Compensation and Well-being

### Fair Compensation

```yaml
compensation_models:

  model_1_stipend:
    description: "Fixed payment for being on-call"
    example:
      - on_call_week: $500
      - per_incident_response: $0
    pros: "Predictable, fair even if quiet week"
    cons: "Doesn't account for incident volume"

  model_2_hourly:
    description: "Paid for hours worked on incidents"
    example:
      - on_call_availability: $100/week
      - incident_response: $75/hour
    pros: "Pay matches work done"
    cons: "Can incentivize slow resolution"

  model_3_time_off:
    description: "Comp time for on-call work"
    example:
      - quiet_week: 0.5 days comp time
      - busy_week: 2 days comp time
    pros: "Flexible, promotes recovery"
    cons: "Requires tracking, approval process"

  model_4_hybrid:
    description: "Base stipend + incident hours"
    example:
      - on_call_week: $300
      - after_hours_incident: $100/hour
      - weekend_incident: $150/hour
    pros: "Balances availability and incident work"
    cons: "More complex to administer"
```

### Burnout Prevention

#### Warning Signs of Burnout

```markdown
🚨 Warning Signs:

Individual level:
- Dreading on-call rotation
- Anxiety when phone buzzes
- Trouble sleeping during on-call week
- Resentment toward job/team
- Decreased quality of incident response

Team level:
- High turnover
- Frequent alerts being ignored
- Decreased code quality
- Increased incident severity/frequency
- Team members calling in sick during on-call
```

#### Burnout Prevention Strategies

**1. Limit On-Call Frequency**
```yaml
max_oncall_frequency:
  - rule: "No more than 1 week in 6"
  - rule: "At least 4 weeks between rotations if possible"
  - rule: "New parents: reduced rotation for 3 months"
  - rule: "After major incident: 2 week break from on-call"
```

**2. Alert Quality Program**
```markdown
Monthly alert review:
☐ Review all alerts that fired
☐ Identify noisy/low-value alerts
☐ Tune or remove 10% noisiest alerts
☐ Ensure all alerts have runbooks
☐ Track alert-to-incident ratio
Target: <20% of alerts require action
```

**3. Mandatory Time Off After Incidents**
```yaml
incident_recovery:
  sev1_incident:
    if_duration: "> 4 hours"
    then: "Next day off"

  weekend_incidents:
    if_duration: "> 2 hours"
    then: "Comp day Monday"

  multi_day_incident:
    if_duration: "> 24 hours"
    then: "Week off after resolution"
```

**4. On-Call Buddy System**
```markdown
Pair experienced with less experienced:
- Buddy reviews on-call person's responses
- Quick Slack for "does this make sense?"
- Reduces stress of being alone
- Knowledge transfer happens naturally
```

## On-Call Metrics

### Track These Metrics

```yaml
oncall_metrics:

  alerting_metrics:
    - total_alerts_per_week: "How many alerts fired?"
    - after_hours_alerts: "How many outside business hours?"
    - actionable_alert_percentage: "% that required action"
    - false_positive_rate: "% that were false alarms"
    - time_to_acknowledge: "How fast did on-call respond?"

  incident_metrics:
    - incidents_per_week: "How many incidents occurred?"
    - incident_duration: "How long to resolve?"
    - escalation_rate: "% that required escalation"
    - weekend_incident_rate: "Incidents during weekends"

  human_metrics:
    - sleep_disruption: "Alerts during sleep hours (11pm-7am)"
    - on_call_satisfaction: "Survey: 1-10 rating"
    - burnout_indicators: "Sick days during/after on-call"
    - rotation_fairness: "Even distribution across team?"
```

### Example Dashboard

```markdown
┌─────────────────────────────────────────────────────┐
│ ON-CALL HEALTH DASHBOARD - Q1 2024                  │
├─────────────────────────────────────────────────────┤
│ ALERT QUALITY:                           Score: 73% │
│   Total alerts: 420                                 │
│   Actionable: 306 (73%)          [████████░░]       │
│   False positives: 84 (20%)      [████░░░░░░]       │
│   Duplicate: 30 (7%)             [██░░░░░░░░]       │
│                                                      │
│ INCIDENT LOAD:                                      │
│   Incidents/week: 3.2            [████░░░░░░]       │
│   Avg duration: 45 min           [█████░░░░░]       │
│   After-hours: 35%               [█████████░]       │
│                                                      │
│ TEAM HEALTH:                                        │
│   On-call satisfaction: 7.2/10   [███████░░░]       │
│   Sleep disruptions/week: 1.8    [████████░░]       │
│   Escalation rate: 12%           [███░░░░░░░]       │
│                                                      │
│ 🎯 GOALS:                                           │
│   ☑ Reduce false positives to <10%                 │
│   ☑ Reduce after-hours incidents to <20%           │
│   ☐ Increase satisfaction to 8.0/10                │
│   ☐ Reduce sleep disruptions to <1/week            │
└─────────────────────────────────────────────────────┘
```

## On-Call Handoff Process

### Shift Start Handoff

```markdown
## On-Call Handoff Template

**From:** Alice
**To:** Bob
**Date:** 2024-01-15
**Time:** 9:00 AM

### Ongoing Incidents
- None currently active

### Ongoing Issues (Not Incidents)
1. **Elevated API latency on EU region**
   - Status: Monitoring
   - Impact: p95 latency 250ms (normally 150ms)
   - Ticket: PROD-1234
   - Context: Gradual increase over 3 days, investigating
   - Action: Monitor, escalate if > 400ms

### Recent Incidents (Last 7 Days)
1. **Database Connection Pool Exhaustion - SEV-2**
   - When: Jan 12, 2:30 AM
   - Duration: 45 minutes
   - Resolution: Increased pool size from 100→150
   - Follow-up: PROD-1230 (investigate connection leaks)

### Upcoming Maintenance/Deployments
1. **Kubernetes upgrade - Jan 16, 2 PM**
   - Runbook: https://wiki.company.com/k8s-upgrade
   - Rollback plan: documented in runbook
   - On-call should monitor during upgrade

### Known Issues/Workarounds
1. **Monitoring alert "CacheMisses" is noisy**
   - Fires 2-3x per day
   - Safe to ack if cache hit rate > 90%
   - Fix in progress: PROD-1235

### Important Context
- Holiday weekend coming up (Jan 20-21)
- New feature flag "beta-checkout-flow" rolled out to 5% yesterday
- Database backup window: Every day 2-4 AM UTC

### Questions?
Reachable on Slack @alice until 6 PM today.
```

## Key Takeaways

1. **On-call is a team sport** - Don't suffer alone, escalate early
2. **Quality > Quantity** - Fix noisy alerts, reduce false positives
3. **Runbooks are essential** - Every alert should have clear instructions
4. **Compensate fairly** - Being on-call has real cost
5. **Prevent burnout** - Monitor team health, rotate regularly
6. **Good handoffs** - Context transfer is critical
7. **Continuous improvement** - Learn from every incident
8. **It's OK to not know** - Escalate when uncertain
9. **Recovery > diagnosis** - During incidents, restore service first
10. **Take care of yourself** - Sleep, eat, take breaks

**Remember:** The best on-call shift is one where nothing happens because systems are reliable and alerts are well-tuned. Work toward that goal.
