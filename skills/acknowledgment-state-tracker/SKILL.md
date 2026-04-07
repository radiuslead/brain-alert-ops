---
name: acknowledgment-state-tracker
description: >
  Build state tracking for operational alerts — track which alerts are open,
  acknowledged, escalated, or resolved, and prevent duplicate notifications.
  Activates on: "track alert state", "who acknowledged this alert", "alert
  status tracking", "prevent duplicate alerts", "alert lifecycle", "was this
  acknowledged", "state management for alerts", or any request to track
  the status of operational alerts over time.
---

# Acknowledgment State Tracker — Brain Alert Ops

You are an operational alerting state management specialist. Without state
tracking, your alerting system is a fire-and-forget broadcaster. With it,
you have an operational system — you know which alerts are open, who
responded, how long it took, and whether the system is working.

---

## Why State Matters

```
WITHOUT STATE:
  Alert fires → Teams message → engineer responds (maybe)
  Five minutes later: same alert fires again → another Teams message
  Ten engineers all start working the same issue
  Nobody knows if anyone else is on it
  No audit trail. No SLA measurement. No accountability.

WITH STATE:
  Alert fires → state record created → Teams card posted
  Engineer acknowledges → state updated → voice call cancelled
  15 min timer → if not acknowledged → escalate + state updated
  Alert resolves → state closed → SLA recorded → Teams card updated
  Monthly report: "Average CRITICAL response time: 8 minutes"
```

---

## State Data Model

```
ALERT STATE RECORD:
{
  event_id:           "evt_20260407_143022_TX042"  (primary key)
  source_system:      "RF Hawkeye"
  site_id:            "TX-042"
  severity:           "CRITICAL"
  alert_type:         "signal_degradation"
  status:             "OPEN" | "ACKNOWLEDGED" | "ESCALATED" | "RESOLVED" | "SUPPRESSED"
  created_at:         "2026-04-07T14:30:22Z"
  acknowledged:       false → true
  acknowledged_by:    null → "john.smith@company.com"
  acknowledged_at:    null → "2026-04-07T14:38:15Z"
  time_to_ack_mins:   null → 7.9
  escalation_level:   0 → 1 → 2
  escalated_at:       null → "2026-04-07T14:45:22Z"
  incident_id:        null → "INC0234567"
  teams_message_id:   "[Teams message ID for card updates]"
  teams_channel_id:   "[Channel ID]"
  resolved_at:        null → "2026-04-07T15:22:00Z"
  time_to_resolve_mins: null → 51.6
  resolution_note:    null → "Antenna realigned, signal restored"
}
```

---

## Implementation Options

### Option A: SharePoint List (Free, no extra license)

```
CREATE SHAREPOINT LIST: "AlertStateTracker"
Columns:
  Title (text):              event_id
  SourceSystem (text):       source_system
  SiteID (text):             site_id
  Severity (choice):         CRITICAL/HIGH/MEDIUM/LOW/INFO
  AlertType (text):          alert_type
  Status (choice):           OPEN/ACKNOWLEDGED/ESCALATED/RESOLVED/SUPPRESSED
  CreatedAt (date/time):     auto-filled
  Acknowledged (yes/no):     default: No
  AcknowledgedBy (text):     UPN
  AcknowledgedAt (date/time)
  TimeToAckMins (number)
  EscalationLevel (number):  default: 0
  IncidentID (text)
  TeamsMessageID (text)
  ResolvedAt (date/time)
  TimeToResolveMins (number)
  ResolutionNote (text, multiline)

POWER AUTOMATE — CREATE STATE RECORD:
Action: Create item (SharePoint)
Site: [your SharePoint site]
List: AlertStateTracker
Fields:
  Title: [event_id]
  Status: OPEN
  Severity: [severity]
  SiteID: [site_id]
  [all other fields from event schema]

POWER AUTOMATE — UPDATE STATE ON ACKNOWLEDGMENT:
Action: Update item (SharePoint)
Filter: Title eq '[event_id]'
Fields:
  Status: ACKNOWLEDGED
  Acknowledged: Yes
  AcknowledgedBy: [responder UPN from Teams card action]
  AcknowledgedAt: [utcNow()]
  TimeToAckMins: [calculate: diff between CreatedAt and now]
```

### Option B: Dataverse (Power Platform native, more powerful)

```
CREATE TABLE: alert_events
  event_id (text, primary key)
  [all fields from data model above]
  [add: audit columns, relationships to other Dataverse tables]

Advantage over SharePoint:
  - Better querying (full OData support)
  - Relationships to other Dataverse tables
  - Power BI built-in reporting
  - Better concurrency handling

Requires: Power Platform environment with Dataverse (standard M365 or premium)
```

### Option C: Azure Table Storage (Cheapest at scale)

```
TABLE: alertstates
PartitionKey: [site_id]
RowKey: [event_id]
[all fields as table entities]

Cost: ~$0.045 per million operations — essentially free for alerting volumes
Requires: Azure Storage account

POWER AUTOMATE → AZURE TABLE STORAGE:
Use Azure Table Storage connector (standard — no premium needed)
Action: Insert or Replace Entity
  Table: alertstates
  PartitionKey: [site_id]
  RowKey: [event_id]
  Entity: [JSON of all state fields]
```

---

## Deduplication Check

Before processing any alert:

```
POWER AUTOMATE — CHECK FOR EXISTING OPEN ALERT:
Action: Get items (SharePoint)
Filter: Title eq '[event_id]' and Status ne 'RESOLVED'

If items found (Count > 0):
  DUPLICATE — do not create new Teams card or voice call
  Instead: add comment/update to existing card
  "Alert re-triggered at [time]. Currently ACKNOWLEDGED by [person]."

If no items found:
  NEW ALERT — proceed with full processing pipeline
```

---

## Alert Deduplication Window

For alerts from systems that send repeated notifications for the same event:

```
POWER AUTOMATE — CHECK RECENT SAME-TYPE ALERTS:
Action: Get items (SharePoint)
Filter: SiteID eq '[site_id]' 
        and AlertType eq '[alert_type]'
        and Status ne 'RESOLVED'
        and CreatedAt ge '[utcNow() minus 30 minutes]'

If similar recent alert exists:
  Suppress new alert — add to existing record as re-trigger count
  Do NOT send new Teams message or voice call

This prevents alert storms from a flapping sensor.
```

---

## SLA Reporting

Generate monthly SLA reports from state data:

```
POWER AUTOMATE (Monthly scheduled flow):
Get all AlertStateTracker items where:
  CreatedAt in last 30 days
  Severity = CRITICAL

Calculate:
  Average TimeToAckMins
  % Acknowledged within 15 minutes (SLA target)
  % Escalated
  Total CRITICAL alerts
  Top 3 sites by alert volume
  Top 3 alert types by frequency

Post to Teams or send as email:
"NOC ALERT SLA REPORT — [Month]
CRITICAL alerts: [N]
Avg acknowledgment time: [X] minutes
SLA compliance (< 15 min): [X%]
Escalation rate: [X%]
Top alert site: [site_name] ([N] alerts)"
```

---

## Example Invocations

> "How do we track whether alerts were acknowledged and by whom?"
→ SharePoint list schema + Power Automate create/update actions.

> "We keep sending duplicate alerts for the same issue — how do we stop it?"
→ Deduplication check + flapping suppression window logic.

> "Management wants to know our average response time to critical alerts"
→ SLA reporting from state data with monthly scheduled flow.

> "We need to know if the voice call was successful or if it needs to escalate"
→ State update flow on voice call result with escalation trigger.
