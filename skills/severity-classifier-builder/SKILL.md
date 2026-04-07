---
name: severity-classifier-builder
description: >
  Build severity classification logic for operational alert systems.
  Use when alerts need to be automatically classified as CRITICAL/HIGH/MEDIUM/LOW
  based on keywords, thresholds, site type, time of day, or business impact.
  Activates on: "classify alert severity", "priority logic", "severity rules",
  "how do we know which alerts are critical", "alert routing logic",
  "CRITICAL vs INFO", "all alerts look the same", or any request to build
  automatic priority detection for operational alerts.
---

# Severity Classifier Builder — Brain Alert Ops

You are an operational alerting logic specialist. You build the classification
layer that sits between raw alert ingestion and routing actions. Without it,
every alert is treated equally — which means critical events get buried in noise.

---

## The Classification Problem

```
CURRENT STATE (most orgs):
  All alerts → same Teams channel → same format → engineer reads manually
  Critical RF degradation: "Alert at TX-042"
  Informational status: "Scheduled maintenance TX-042"
  Both treated identically.

TARGET STATE:
  Alert arrives → classifier assigns severity → routes to correct channel
  CRITICAL → #critical-alerts + voice call + incident ticket
  HIGH     → #ops-alerts + Teams mention
  MEDIUM   → #ops-alerts (no mention)
  LOW/INFO → #system-notifications (muted)
```

---

## Classification Methods

### Method 1: Keyword Matching (Fastest, works with email source)

```
POWER AUTOMATE SWITCH EXPRESSION:

Initialize variable: AlertSeverity (String) = 'INFO'

Condition 1 — CRITICAL keywords:
if(
  or(
    contains(toUpper(triggerOutputs()?['body/subject']), 'CRITICAL'),
    contains(toUpper(triggerOutputs()?['body/subject']), 'EMERGENCY'),
    contains(toUpper(triggerOutputs()?['body/subject']), 'ALARM'),
    contains(toUpper(triggerOutputs()?['body/subject']), 'FAILURE'),
    contains(toUpper(triggerOutputs()?['body/subject']), 'DOWN'),
    contains(toUpper(triggerOutputs()?['body/subject']), 'OUTAGE')
  ),
  'CRITICAL',
  'CHECK_NEXT'
)

Condition 2 — HIGH keywords:
if(
  or(
    contains(toUpper(triggerOutputs()?['body/subject']), 'HIGH'),
    contains(toUpper(triggerOutputs()?['body/subject']), 'DEGRADED'),
    contains(toUpper(triggerOutputs()?['body/subject']), 'WARNING'),
    contains(toUpper(triggerOutputs()?['body/subject']), 'FAULT')
  ),
  'HIGH',
  'CHECK_NEXT'
)

Condition 3 — MEDIUM:
if(
  or(
    contains(toUpper(triggerOutputs()?['body/subject']), 'MEDIUM'),
    contains(toUpper(triggerOutputs()?['body/subject']), 'MINOR'),
    contains(toUpper(triggerOutputs()?['body/subject']), 'NOTICE')
  ),
  'MEDIUM',
  'LOW'
)
```

### Method 2: Threshold-Based Classification (Structured payload)

For monitoring systems that expose numeric values:

```
CLASSIFICATION RULES (RF/Telecom example):

RSSI threshold rules:
  value > -75 dBm  → INFO    (normal operation)
  -75 to -85 dBm   → MEDIUM  (degraded, monitor)
  -85 to -95 dBm   → HIGH    (significant degradation)
  < -95 dBm        → CRITICAL (service impacting)

BER (Bit Error Rate) rules:
  < 1e-6   → INFO
  1e-6 to 1e-4 → MEDIUM
  1e-4 to 1e-2 → HIGH
  > 1e-2   → CRITICAL

Power Automate expression:
if(
  less(float(variables('MetricValue')), -95),
  'CRITICAL',
  if(
    less(float(variables('MetricValue')), -85),
    'HIGH',
    if(
      less(float(variables('MetricValue')), -75),
      'MEDIUM',
      'INFO'
    )
  )
)
```

### Method 3: AI-Assisted Classification (Modern approach — 2026)

Use Claude or Azure OpenAI to classify from unstructured text:

```json
CLAUDE API CALL (from Power Automate HTTP action):

POST https://api.anthropic.com/v1/messages
{
  "model": "claude-sonnet-4-20250514",
  "max_tokens": 100,
  "system": "You are an operational alert classifier for a telecom/utilities NOC. Classify the severity of the following alert as exactly one of: CRITICAL, HIGH, MEDIUM, LOW, INFO. Return ONLY the severity word, nothing else.",
  "messages": [{
    "role": "user",
    "content": "Alert subject: [subject]\nAlert body: [body preview]"
  }]
}

Response: "CRITICAL"

Use this when:
- Alert format is inconsistent across systems
- Keyword matching produces too many false positives
- Alert text is natural language with no standard structure

Cost: ~$0.001 per alert classification at Sonnet pricing
Latency: ~500ms — acceptable for alerting, not real-time telemetry
```

### Method 4: Multi-Factor Classification

Combine multiple signals for highest accuracy:

```
SEVERITY SCORE = base_severity + site_modifier + time_modifier

base_severity (from keyword/threshold):
  CRITICAL = 100
  HIGH = 75
  MEDIUM = 50
  LOW = 25
  INFO = 0

site_modifier (business criticality):
  Primary hub site: +25
  Secondary site: +10
  Test/lab site: -25

time_modifier (operational context):
  Business hours (8am-6pm weekday): 0
  After hours weekday: +15
  Weekend: +25
  Scheduled maintenance window: -50 (suppresses false alerts)

Final score → Severity:
  >= 100: CRITICAL
  >= 75:  HIGH
  >= 50:  MEDIUM
  >= 25:  LOW
  < 25:   INFO
```

---

## Classification Rule Document

Generate this for every alerting system:

```
SEVERITY CLASSIFICATION RULES
System: [Source system name]
Owner:  [Name]
Date:   [Today]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

CRITICAL — Immediate response required (< 15 minutes)
────────────────────────────────────────────────────
Definition: Service impacting, revenue loss, safety risk
Keywords:   [List]
Thresholds: [List metric/threshold pairs]
Action:     Voice call to on-call + Teams CRITICAL channel + auto-incident

HIGH — Response required within 1 hour
────────────────────────────────────────────────────
Definition: Significant degradation, risk of service impact
Keywords:   [List]
Thresholds: [List]
Action:     Teams HIGH channel + @mention on-call group

MEDIUM — Response required within 4 hours
────────────────────────────────────────────────────
Definition: Degraded but not impacting service
Keywords:   [List]
Thresholds: [List]
Action:     Teams OPS channel (no mention)

LOW / INFO — Next business day
────────────────────────────────────────────────────
Definition: Informational, no immediate action needed
Keywords:   [List]
Action:     System notifications channel (muted)

SUPPRESSION RULES
────────────────────────────────────────────────────
Maintenance windows: [Schedule — suppress all during window]
Known flapping:      [Sites/metrics that generate false positives]
Deduplication:       Same alert_type + site_id within 30 min = suppress repeat
```

---

## Example Invocations

> "RF Hawkeye sends emails with no consistent severity — how do we classify them?"
→ Method 1 keyword classifier + Method 3 AI fallback for edge cases.

> "We want CRITICAL alerts to be based on actual threshold values, not keywords"
→ Method 2 threshold-based rules with RF/telecom example thresholds.

> "Some of our critical alerts get buried because everything looks the same"
→ Full classification rule document + multi-factor scoring model.

> "Build severity logic that considers time of day and site importance"
→ Method 4 multi-factor scoring with site and time modifiers.
