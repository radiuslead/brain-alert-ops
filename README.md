# Brain: Alert Ops — AI Skills for Event-Driven Operational Alerting

**Stop broadcasting alerts. Start operating on them.**

8 skills for NOC engineers, infrastructure ops managers, and IT architects
who need to transform passive notification pipelines into real-time,
event-driven operational systems — with AI voice escalation, actionable
Teams cards, automatic incident creation, and full state tracking.

---

## The Problem This Solves

Most enterprise alerting looks like this:

```
Monitoring tool → email → shared mailbox → Power Automate → Teams channel
```

That's a notification system, not an operational system. Engineers monitor
a channel manually, miss alerts, create tickets by hand, and have no way
to prove who responded or how fast.

This brain pack builds the real system:

```
Signal (any monitoring source)
    ↓
Structured Event (schema with severity, site, metric, status)
    ↓
Classification (CRITICAL/HIGH/MEDIUM/LOW — keyword, threshold, or AI)
    ↓
Routing Logic (right team, right time, right channel)
    ↓
Actions:
  → Adaptive card with Acknowledge / Escalate / Create Ticket buttons
  → AI voice call to on-call engineer (Vapi / Azure ACS / Synthflow)
  → Auto-created ServiceNow/Jira incident
  → 15-minute escalation timer with on-call rotation lookup
    ↓
State Tracking (acknowledged, escalated, resolved — with SLA measurement)
```

---

## Install

```bash
/plugin marketplace add radiuslead/brain-alert-ops
/plugin install brain-alert-ops
```

---

## What's Inside

| Skill | What it does |
|-------|-------------|
| `alert-event-schema-designer` | Design structured event schemas — transform email blobs into routable JSON with severity, site, metric, lifecycle fields |
| `severity-classifier-builder` | Build classification logic — keywords, thresholds, AI via Claude API, multi-factor scoring with site and time modifiers |
| `ai-voice-escalator` | AI voice call escalation — Azure ACS (enterprise), Vapi (developer), or Synthflow (no-code) with escalation chain |
| `adaptive-card-actioner` | Actionable Teams cards — Acknowledge / Escalate / Create Ticket from Teams, card updates on state change |
| `incident-auto-creator` | Auto-create ServiceNow/Jira incidents from CRITICAL alerts — REST API (no premium connector), bidirectional sync, deduplication |
| `acknowledgment-state-tracker` | Track full alert lifecycle — SharePoint/Dataverse/Azure Table Storage, SLA measurement, flapping suppression |
| `on-call-rotation-manager` | Route to the right engineer — schedule lookup, specialty routing, business hours logic, escalation levels |
| `ops-runbook-writer` | Generate response runbooks, architecture docs, and handover packages for NOC/ops teams |

---

## The Voice Escalation Advantage

The `ai-voice-escalator` skill covers what your team might not be able to get approved — and offers alternatives for every enterprise procurement reality:

**Azure ACS** — native Microsoft, enterprise-grade, requires Azure approval
**Vapi** — third-party SaaS, working in days, AI voice agent that can take spoken acknowledgment
**Synthflow** — no-code, no developer needed, working same day

The skill also covers the enterprise reality: often the right solution isn't the approved solution. It gives you the pragmatic path for today and the migration path for when you get proper approval.

---

## Who This Is For

- NOC engineers tired of monitoring a Teams channel manually
- Infrastructure ops managers who need SLA accountability for alert response
- IT architects building proper event-driven systems to replace email-based alerting
- Telecom, utilities, and industrial organizations with RF/SCADA monitoring systems
- Anyone who built a Power Automate workaround and knows the real system is still missing

---

## Built From Real Experience

This brain pack was designed from first-hand experience solving the RF Hawkeye
alert routing problem at an enterprise telecom organization — including the
realities of enterprise procurement, approved toolsets, and the gap between
the right architecture and the one you can actually deploy.

Every skill has two paths: the ideal solution and the pragmatic solution for
organizations where the ideal isn't yet approved.

---

## Pricing & Access

| Option | Price | What you get |
|--------|-------|-------------|
| Brain pack (self-install) | $497 one-time | All 8 skills, GitHub access |
| DFY Implementation | $5,000–$8,000 | Full event-driven alerting system built end-to-end |
| Subscription | $99/month | Hosted + updated as platforms evolve |

**Get access → [partnerships@radiuslead.com](mailto:partnerships@radiuslead.com)**

---

## Built by AgentBrains | github.com/radiuslead | MIT License
