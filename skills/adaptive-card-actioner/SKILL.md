---
name: adaptive-card-actioner
description: >
  Build actionable Teams adaptive cards for operational alerts — cards that
  engineers can acknowledge, escalate, or resolve directly from Teams without
  switching systems. Activates on: "actionable Teams alert", "adaptive card",
  "acknowledge from Teams", "Teams alert with buttons", "interactive alert card",
  "engineers should be able to act from Teams", or any request to build
  interactive alert notifications in Microsoft Teams.
---

# Adaptive Card Actioner — Brain Alert Ops

You are a Microsoft Teams adaptive card specialist for operational alerting.
A Teams message is passive — engineers read it. An adaptive card is active —
engineers act on it. The goal is zero context switching: see the alert,
acknowledge it, create a ticket, escalate — all without leaving Teams.

---

## The Difference

```
PASSIVE (plain Teams message):
  "CRITICAL: RF degradation at TX-042. RSSI -97dBm."
  → Engineer reads it, opens ServiceNow, creates ticket manually,
    posts "I'm on it" in the channel, then works the problem.
  → No tracking. No acknowledgment proof. No state.

ACTIVE (adaptive card):
  ┌─────────────────────────────────────────────────┐
  │ 🔴 CRITICAL ALERT                               │
  │ RF Hawkeye — TX-042 Dallas Tower 42             │
  ├─────────────────────────────────────────────────┤
  │ Metric:     RSSI                                │
  │ Value:      -97 dBm  (threshold: -85 dBm)       │
  │ Time:       2026-04-07 14:30:22 UTC             │
  │ Status:     OPEN — unacknowledged               │
  ├─────────────────────────────────────────────────┤
  │ [✅ Acknowledge] [🔺 Escalate] [🎫 Create Ticket]│
  └─────────────────────────────────────────────────┘
  → Engineer clicks Acknowledge → card updates instantly
  → ServiceNow ticket auto-created
  → State tracker updated
  → Voice call cancelled if pending
```

---

## CRITICAL Alert Adaptive Card JSON

```json
{
  "type": "AdaptiveCard",
  "version": "1.5",
  "body": [
    {
      "type": "Container",
      "style": "attention",
      "items": [
        {
          "type": "ColumnSet",
          "columns": [
            {
              "type": "Column",
              "width": "auto",
              "items": [{
                "type": "TextBlock",
                "text": "🔴",
                "size": "ExtraLarge"
              }]
            },
            {
              "type": "Column",
              "width": "stretch",
              "items": [
                {
                  "type": "TextBlock",
                  "text": "CRITICAL ALERT",
                  "weight": "Bolder",
                  "size": "Large",
                  "color": "Attention"
                },
                {
                  "type": "TextBlock",
                  "text": "${source_system} — ${site_name}",
                  "isSubtle": true,
                  "spacing": "None"
                }
              ]
            }
          ]
        }
      ]
    },
    {
      "type": "FactSet",
      "facts": [
        { "title": "Alert Type", "value": "${alert_type}" },
        { "title": "Metric", "value": "${metric}: **${value} ${unit}**" },
        { "title": "Threshold", "value": "${threshold} ${unit}" },
        { "title": "Site ID", "value": "${site_id}" },
        { "title": "Region", "value": "${region}" },
        { "title": "Time (UTC)", "value": "${timestamp}" },
        { "title": "Event ID", "value": "${event_id}" },
        { "title": "Status", "value": "${status}" }
      ]
    },
    {
      "type": "TextBlock",
      "text": "${message}",
      "wrap": true,
      "spacing": "Medium"
    },
    {
      "type": "Container",
      "id": "statusContainer",
      "style": "warning",
      "isVisible": true,
      "items": [{
        "type": "TextBlock",
        "text": "⏳ Awaiting acknowledgment — voice call will be placed in 5 minutes if not acknowledged",
        "wrap": true,
        "color": "Warning"
      }]
    }
  ],
  "actions": [
    {
      "type": "Action.Execute",
      "title": "✅ Acknowledge",
      "verb": "acknowledge",
      "style": "positive",
      "data": {
        "action": "acknowledge",
        "event_id": "${event_id}",
        "site_id": "${site_id}"
      }
    },
    {
      "type": "Action.Execute",
      "title": "🔺 Escalate",
      "verb": "escalate",
      "style": "destructive",
      "data": {
        "action": "escalate",
        "event_id": "${event_id}"
      }
    },
    {
      "type": "Action.OpenUrl",
      "title": "🎫 Create Ticket",
      "url": "https://[your-servicenow]/incident/new?category=infrastructure&short_description=Alert+${event_id}"
    },
    {
      "type": "Action.ShowCard",
      "title": "📋 Details",
      "card": {
        "type": "AdaptiveCard",
        "body": [{
          "type": "TextBlock",
          "text": "${raw_payload}",
          "wrap": true,
          "fontType": "Monospace",
          "size": "Small"
        }]
      }
    }
  ]
}
```

---

## Card Update After Acknowledgment

When engineer clicks Acknowledge, update the card in-place:

```json
UPDATED CARD BODY (replace original):
{
  "type": "Container",
  "style": "good",
  "items": [{
    "type": "TextBlock",
    "text": "✅ Acknowledged by ${acknowledged_by} at ${acknowledged_at}",
    "color": "Good",
    "weight": "Bolder"
  }]
}

REMOVE action buttons (replace actions array with empty):
"actions": []
```

---

## Power Automate: Post Card + Handle Response

```
STEP 1: Post adaptive card to Teams channel
Action: Post adaptive card and wait for a response
  Team: [team name]
  Channel: [critical-alerts]
  Message: [adaptive card JSON with variables filled in]
  Update message: "Alert ${event_id} — ${status}"

STEP 2: Process response
Switch on body('Post_adaptive_card')?['data']?['action']:

  Case "acknowledge":
    - Set alert status = ACKNOWLEDGED
    - Set acknowledged_by = body('Post_adaptive_card')?['responder']?['displayName']
    - Set acknowledged_at = utcNow()
    - Update SharePoint/Dataverse state record
    - Cancel pending voice call if applicable
    - Update card in Teams

  Case "escalate":
    - Set escalation_level = escalation_level + 1
    - Get next on-call contact
    - Trigger voice call to next contact
    - Post escalation notice to Teams
    - Update card

STEP 3: Handle timeout (no response)
  After: 15 minutes
  If alert still OPEN:
    - Trigger voice call sequence
    - Post reminder card
```

---

## Severity-Based Card Styles

```
CRITICAL: Container style = "attention" | Emoji: 🔴 | Mention @oncall
HIGH:     Container style = "warning"   | Emoji: 🟡 | No mention
MEDIUM:   Container style = "default"   | Emoji: 🟠 | No mention
LOW/INFO: Container style = "default"   | Emoji: 🔵 | No mention, muted channel
```

---

## Example Invocations

> "Build a Teams alert card that engineers can acknowledge without leaving Teams"
→ Full adaptive card JSON + Power Automate post + response handler.

> "When someone clicks Acknowledge, how do we update the card?"
→ Card update JSON + Power Automate update card action.

> "We want the card to show who acknowledged it and when"
→ Updated card body with acknowledged_by and acknowledged_at fields.

> "Build different card styles for CRITICAL vs HIGH vs INFO alerts"
→ Severity-based card style guide with emoji, color, and mention logic.
