# Incident Response - Tips & Tricks

[TOC]

## Incident Response Lifecycle

```
Detection → Response → Recovery → Analysis → Prevention
    ↑                                                ↓
    └────────────── Continuous Improvement ─────────┘
```

## Incident Severity Levels

Define clear severity levels to guide response:

### Severity Matrix

| Severity | Impact | Response Time | Examples |
|----------|--------|---------------|----------|
| **SEV-1** | Complete outage, all users affected | < 5 minutes | Service down, data loss, security breach |
| **SEV-2** | Major degradation, many users affected | < 15 minutes | Elevated error rates (>5%), severe latency issues |
| **SEV-3** | Partial degradation, some users affected | < 1 hour | Elevated error rates (1-5%), minor feature outage |
| **SEV-4** | Minor issue, minimal user impact | < 4 hours | Low error rates (<1%), performance degradation |
| **SEV-5** | No user impact, internal issue | Next business day | Monitoring alert, internal tool issue |

### Severity Assessment Questions

```
1. How many users are affected?
   - All users → SEV-1
   - >50% users → SEV-2
   - 10-50% users → SEV-3
   - <10% users → SEV-4

2. Is data at risk?
   - Data loss occurring → SEV-1
   - Potential data loss → SEV-2
   - No data risk → Based on user impact

3. Is there a security breach?
   - Active breach → SEV-1
   - Potential vulnerability → SEV-2
   - No security risk → Based on user impact

4. Are we violating SLAs/SLOs?
   - SLA violated → SEV-1 or SEV-2
   - SLO at risk → SEV-2 or SEV-3
   - Within budget → SEV-4 or SEV-5
```

## Incident Response Roles

### Incident Commander (IC)

**Responsibilities:**
- Owns the incident from detection to resolution
- Makes all tactical decisions
- Delegates tasks to responders
- Coordinates communication
- Declares incident resolved

**Key Skills:**
- Stay calm under pressure
- Clear communication
- Decisive decision-making
- Time management

**IC Checklist:**
```markdown
☐ Assess severity and declare incident
☐ Assemble response team
☐ Establish communication channels
☐ Create incident document/timeline
☐ Brief team on situation
☐ Delegate investigation tasks
☐ Make go/no-go decisions on fixes
☐ Coordinate status updates
☐ Declare resolution
☐ Schedule postmortem
```

### Communications Lead

**Responsibilities:**
- Manage internal/external communication
- Post status page updates
- Handle customer inquiries
- Escalate to executives
- Keep stakeholders informed

**Communication Template:**
```markdown
## Status Update [TIME]

**Current Status:** [Investigating/Identified/Monitoring/Resolved]

**User Impact:**
- What users are experiencing
- Which features are affected
- Workarounds (if any)

**Next Update:** [TIME]

**What We're Doing:**
- Current investigation focus
- Actions being taken

**Incident Commander:** [NAME]
```

### Technical Lead(s)

**Responsibilities:**
- Deep technical investigation
- Implement fixes
- Test solutions
- Provide technical guidance

**Investigation Checklist:**
```markdown
☐ Check recent deployments/changes
☐ Review monitoring dashboards
☐ Examine error logs
☐ Check dependency health
☐ Review resource utilization
☐ Analyze traffic patterns
☐ Check for security issues
☐ Verify data integrity
```

### Scribe

**Responsibilities:**
- Document timeline of events
- Record all actions taken
- Track decisions made
- Note who did what when

**Timeline Format:**
```markdown
## Incident Timeline

**[HH:MM]** - First alert fired: High error rate on user-api
**[HH:MM]** - IC [Name] declares SEV-2 incident
**[HH:MM]** - Incident channel created: #incident-1234
**[HH:MM]** - Team assembled: [Names]
**[HH:MM]** - Investigation begins: checking recent deploys
**[HH:MM]** - Root cause identified: database connection pool exhausted
**[HH:MM]** - Fix proposed: increase pool size from 100 to 200
**[HH:MM]** - Fix deployed to production
**[HH:MM]** - Monitoring recovery
**[HH:MM]** - Error rates back to normal
**[HH:MM]** - Incident declared resolved
```

## Incident Response Workflow

### 1. Detection

**How incidents are detected:**
- Automated monitoring alerts
- Customer reports
- Internal user reports
- Scheduled health checks
- External monitoring services

**First Actions:**
```bash
# Immediate assessment
1. What is happening?
2. How many users are affected?
3. What is the severity?
4. Who needs to be notified?
```

### 2. Response

#### Assemble the Team

```yaml
SEV-1:
  - Incident Commander (on-call SRE)
  - Technical Lead (service owner)
  - Communications Lead
  - Scribe
  - Engineering Manager (if extended incident)
  - Executive stakeholder (if SLA violated)

SEV-2:
  - Incident Commander
  - Technical Lead
  - Scribe

SEV-3:
  - Technical Lead
  - Optional IC for coordination
```

#### Establish Communication Channels

```markdown
1. Create dedicated Slack channel: #incident-YYYY-MM-DD-DESCRIPTION
2. Start video call/war room
3. Create incident document (Google Doc/Confluence)
4. Update status page
```

#### Investigation Priorities

```python
# Priority order for investigation
investigation_steps = [
    "1. Stabilize the service (stop the bleeding)",
    "2. Restore user functionality",
    "3. Preserve evidence for postmortem",
    "4. Find root cause (only if it doesn't delay recovery)"
]

# Key principle: Recovery first, investigation second
```

### 3. Recovery

#### Mitigation Strategies

**Option 1: Rollback**
```bash
# Fastest way to recover from bad deploy
✅ Pros: Quick, low risk
❌ Cons: Rolls back good changes too

When to use: Recent deployment suspected
```

**Option 2: Roll Forward**
```bash
# Deploy a fix
✅ Pros: Keeps good changes, fixes root cause
❌ Cons: Takes longer, introduces risk

When to use: Rollback not possible, fix is simple
```

**Option 3: Feature Flag**
```bash
# Disable problematic feature
✅ Pros: Surgical, preserves most functionality
❌ Cons: Feature flags must exist beforehand

When to use: Issue isolated to specific feature
```

**Option 4: Traffic Shift**
```bash
# Route traffic away from affected systems
✅ Pros: Fast, preserves system for debugging
❌ Cons: Requires redundant infrastructure

When to use: Multi-region setup, can shift load
```

**Option 5: Scale Up**
```bash
# Add more resources
✅ Pros: Simple, buys time for proper fix
❌ Cons: Costly, may not solve root cause

When to use: Resource exhaustion suspected
```

#### Recovery Checklist

```markdown
☐ Implement fix/mitigation
☐ Monitor key metrics for 15-30 minutes
☐ Verify error rates returned to normal
☐ Confirm user reports stopped
☐ Check for side effects of fix
☐ Update status page: "Issue resolved"
☐ Notify stakeholders
☐ Continue monitoring for 24 hours
```

### 4. Resolution

**Declaring Resolution:**

Only declare resolved when:
- ✅ Core functionality restored
- ✅ Error rates within normal range for 30+ minutes
- ✅ No new related alerts
- ✅ Customer complaints ceased
- ✅ Monitoring shows stable state

**Don't declare resolved if:**
- ❌ Just applied a temporary band-aid
- ❌ Root cause still unknown and might recur
- ❌ Monitoring still shows abnormalities

## Communication During Incidents

### Internal Communication

**Status Update Cadence:**

```yaml
SEV-1:
  initial: "Immediately upon detection"
  updates: "Every 15 minutes"
  channels: [slack, email, status_page, executive_brief]

SEV-2:
  initial: "Within 5 minutes"
  updates: "Every 30 minutes"
  channels: [slack, status_page]

SEV-3:
  initial: "Within 15 minutes"
  updates: "Every hour or when major updates"
  channels: [slack, status_page]
```

**Update Template:**
```markdown
🚨 **INCIDENT UPDATE** - [SEV-X] - [Time]

**Status:** [Investigating/Mitigating/Monitoring/Resolved]

**Impact:**
- Service: [name]
- Affected users: [percentage/count]
- Symptoms: [what users see]

**Current Actions:**
- [What we're doing right now]

**ETA for next update:** [time]

**IC:** @username | **Channel:** #incident-1234
```

### External Communication

**Status Page Updates:**

```markdown
# Investigating
[TIME] - We are investigating reports of [issue] affecting [service].
Users may experience [symptoms]. We will provide updates as we learn more.

# Identified
[TIME] - We have identified the cause as [brief explanation].
Our team is working on a fix. Estimated time to resolution: [X minutes/hours].

# Monitoring
[TIME] - A fix has been deployed and we are monitoring the results.
The issue appears to be resolved for most users.

# Resolved
[TIME] - This incident has been resolved. All systems are operating normally.
We apologize for the inconvenience and will be publishing a postmortem within [X days].
```

**Customer Support Script:**
```markdown
Thank you for contacting us. We are aware of an issue affecting [service].

Current Status: [From status page]

Affected Features: [List]

Workaround: [If available]

Expected Resolution: [If known]

You can track real-time updates at: [status page URL]

We sincerely apologize for the inconvenience.
```

## Incident Response Best Practices

### ✅ Do's

1. **Declare incidents early** - False positive is better than late response
2. **Focus on recovery first** - Users don't care about root cause, they want service working
3. **Document everything** - Your future self will thank you during the postmortem
4. **Communicate proactively** - Over-communication is better than silence
5. **Use the buddy system** - Don't let responders work alone for extended periods
6. **Take breaks** - Tired engineers make mistakes
7. **Preserve evidence** - Logs, metrics, database snapshots before fixing
8. **Follow your runbooks** - They exist for a reason

### ❌ Don'ts

1. **Don't panic** - Stay calm, think clearly
2. **Don't make assumptions** - Verify, don't guess
3. **Don't experiment in production** - Unless absolutely necessary
4. **Don't skip communication** - Stakeholders need updates
5. **Don't work alone** - Get help, don't hero
6. **Don't ignore health** - Take breaks, eat, hydrate
7. **Don't forget to document** - "I'll remember" = you won't
8. **Don't blame** - Focus on systems, not people

## Incident Response Tools

### Essential Tools

```markdown
**Communication:**
- Slack/Teams: Real-time coordination
- Zoom/Google Meet: War room
- PagerDuty/Opsgenie: Escalation

**Documentation:**
- Google Docs: Real-time collaborative notes
- Confluence: Incident documentation
- Jira: Tracking follow-up work

**Monitoring:**
- Grafana: Dashboards
- Datadog: Full observability
- CloudWatch: AWS metrics

**Status Communication:**
- Statuspage.io
- Atlassian Statuspage
- Custom status page
```

### Incident Response Automation

```python
# Auto-create incident channel
def create_incident_channel(severity, description):
    """Creates dedicated Slack channel and documents"""
    timestamp = datetime.now().strftime("%Y%m%d-%H%M")
    channel_name = f"incident-{timestamp}-{description}"

    # Create Slack channel
    channel = slack.create_channel(channel_name)

    # Create incident doc
    doc = create_incident_doc(
        channel=channel_name,
        severity=severity,
        description=description
    )

    # Invite responders based on severity
    responders = get_responders_for_severity(severity)
    for responder in responders:
        slack.invite_to_channel(channel, responder)

    # Post initial status
    slack.post_message(channel, f"""
    🚨 **{severity} Incident Declared**

    Description: {description}
    Incident Doc: {doc.url}
    Status Page: {status_page.url}

    Roles needed:
    - Incident Commander: [volunteer]
    - Technical Lead: [volunteer]
    - Communications Lead: [volunteer]
    - Scribe: [volunteer]
    """)

    return channel, doc
```

## Mental Models for Incident Response

### The Triage Mindset

Think like an ER doctor:
1. **Stabilize** - Stop the bleeding (restore service)
2. **Assess** - Understand the damage (impact assessment)
3. **Treat** - Apply appropriate treatment (implement fix)
4. **Monitor** - Watch for complications (post-recovery monitoring)
5. **Analyze** - Prevent recurrence (postmortem)

### Decision Making Under Pressure

```python
def make_incident_decision(options):
    """Framework for making decisions during incidents"""

    for option in options:
        score = evaluate(
            time_to_implement=1-10,  # Lower is better
            risk_level=1-10,         # Lower is better
            probability_of_success=1-10,  # Higher is better
            user_impact_reduction=1-10    # Higher is better
        )

    # Choose option with best score
    # When in doubt, choose faster, lower-risk option
    # Perfect is the enemy of good during incidents
```

### The Incident Spiral of Doom

**How incidents get worse:**
```
Incident occurs
    → Panic sets in
        → Rushed "fixes" without proper testing
            → More things break
                → More panic
                    → Even more rushed "fixes"
                        → Complete disaster
```

**How to avoid it:**
```
Incident occurs
    → Follow incident response process
        → Clear roles and communication
            → Methodical investigation
                → Tested mitigation
                    → Careful rollout
                        → Monitoring
                            → Recovery
```

## Incident Response Training

### Fire Drills

Run practice incidents quarterly:

```markdown
**Chaos Game Day Format:**

1. Inject realistic failure (with safety limits)
2. See how team responds
3. Time how long detection and recovery take
4. Identify gaps in runbooks, monitoring, automation
5. Update processes based on learnings

**Example Scenarios:**
- Database goes read-only
- Payment provider is down
- DDoS attack
- Certificate expiration
- Cloud region outage
- Accidental data deletion
```

### Runbook Exercises

Validate runbooks by walking through them:
- Can a new team member follow them?
- Are steps still accurate?
- Are all tools/access documented?
- Are there screenshots/examples?

## Key Takeaways

1. **Have a clear process** - Don't figure it out during the incident
2. **Assign roles explicitly** - IC, tech lead, comms, scribe
3. **Recovery > Root Cause** - Users want service working, not explanations
4. **Document everything** - The scribe role is critical
5. **Communicate proactively** - Updates every 15-30 min for SEV-1/2
6. **Preserve evidence** - You'll need it for the postmortem
7. **Don't hero** - Tag in help, take breaks
8. **Practice** - Run fire drills before real incidents
9. **Learn and improve** - Every incident should result in prevention work
10. **Blameless culture** - Focus on systems, not people

## Quick Reference Card

```markdown
┌─────────────────────────────────────────────────────┐
│ INCIDENT RESPONSE QUICK REFERENCE                   │
├─────────────────────────────────────────────────────┤
│ 1. DETECT                                           │
│    ├─ Assess severity (SEV-1 to SEV-5)              │
│    └─ Page appropriate responders                   │
│                                                      │
│ 2. RESPOND                                          │
│    ├─ Create incident channel                       │
│    ├─ Assign roles (IC, Tech, Comms, Scribe)        │
│    ├─ Start incident doc/timeline                   │
│    └─ Post initial status update                    │
│                                                      │
│ 3. INVESTIGATE                                      │
│    ├─ Check recent changes                          │
│    ├─ Review dashboards                             │
│    ├─ Examine logs                                  │
│    └─ Test hypotheses                               │
│                                                      │
│ 4. MITIGATE                                         │
│    ├─ Choose mitigation (rollback/fix/scale)        │
│    ├─ Test in safe environment if possible          │
│    ├─ Deploy carefully                              │
│    └─ Monitor for improvement                       │
│                                                      │
│ 5. RESOLVE                                          │
│    ├─ Verify metrics normalized                     │
│    ├─ Confirm user reports stopped                  │
│    ├─ Update status page: "Resolved"                │
│    └─ Thank the team                                │
│                                                      │
│ 6. LEARN                                            │
│    ├─ Schedule postmortem within 48h                │
│    ├─ Identify action items                         │
│    └─ Track to completion                           │
│                                                      │
│ REMEMBER: Recovery first, root cause second!        │
└─────────────────────────────────────────────────────┘
```
