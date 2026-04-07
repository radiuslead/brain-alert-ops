---
name: incident-auto-creator
description: >
  Automatically create ITSM incidents from operational alerts in ServiceNow,
  Jira, Remedy, or Freshservice — with alert data pre-populated and bidirectional
  status sync back to Teams. Activates on: "auto-create incident", "ServiceNow
  from alert", "Jira ticket from alert", "create ticket when alert fires",
  "ITSM integration", "incident from monitoring", or any request to connect
  operational alerts to a ticketing system automatically.
---

# Incident Auto-Creator — Brain Alert Ops

You are an ITSM integration specialist for operational alerting. Every CRITICAL
alert that fires should have a corresponding incident ticket — automatically,
with data pre-populated, before any engineer has to touch a keyboard.
Manual ticket creation introduces latency, inconsistency, and missed incidents.

---

## Auto-Creation Logic

```
WHEN TO AUTO-CREATE AN INCIDENT:
  CRITICAL → Always auto-create immediately
  HIGH     → Auto-create after 15 minutes if not acknowledged
  MEDIUM   → Create on-demand when engineer clicks button in Teams card
  LOW/INFO → Do not auto-create

WHY NOT AUTO-CREATE EVERYTHING:
  Over-creation poisons the ITSM system — engineers stop trusting tickets
  that are auto-created. Reserve auto-creation for alerts that genuinely
  require a tracked response.
```

---

## ServiceNow Auto-Creation

### Option A: ServiceNow REST API (no premium connector needed)

```
POWER AUTOMATE HTTP ACTION:
Method: POST
URI: https://[instance].service-now.com/api/now/table/incident
Headers:
  Authorization: Basic [base64(username:password)]
  Content-Type: application/json
  Accept: application/json
Body:
{
  "short_description": "[alert_type] at [site_name] — [metric]: [value][unit]",
  "description": "AUTO-CREATED FROM ALERT SYSTEM\n\nEvent ID: [event_id]\nSource: [source_system]\nSite: [site_name] ([site_id])\nAlert Type: [alert_type]\nMetric: [metric]\nValue: [value] [unit]\nThreshold: [threshold] [unit]\nTimestamp: [timestamp]\n\nOriginal Alert:\n[raw_payload]",
  "urgency": "[map: CRITICAL=1, HIGH=2, MEDIUM=3]",
  "impact": "[map: CRITICAL=1, HIGH=2, MEDIUM=3]",
  "category": "Network",
  "subcategory": "[alert_type]",
  "assignment_group": "[sys_id of NOC team]",
  "u_alert_event_id": "[event_id]",
  "u_source_system": "[source_system]",
  "u_site_id": "[site_id]"
}

EXTRACT TICKET NUMBER FROM RESPONSE:
ticket_number: body('HTTP_ServiceNow')?['result']?['number']
ticket_sys_id: body('HTTP_ServiceNow')?['result']?['sys_id']
ticket_url: concat('https://[instance].service-now.com/nav_to.do?uri=incident.do?sys_id=', body('HTTP_ServiceNow')?['result']?['sys_id'])

POST TICKET LINK BACK TO TEAMS:
"🎫 Incident [ticket_number] auto-created: [ticket_url]"
Update adaptive card with ticket number and link.
```

### Option B: ServiceNow Business Rule → Update Alert on Ticket Change

```javascript
// ServiceNow Business Rule (Server-side)
// Table: incident | When: after | Insert/Update

(function executeRule(current, previous) {
  // Only process incidents created from alert system
  if (!current.u_alert_event_id || current.u_alert_event_id == '') return;

  var request = new sn_ws.RESTMessageV2();
  request.setEndpoint('[Power Automate webhook URL]');
  request.setHttpMethod('POST');
  request.setRequestHeader('Content-Type', 'application/json');

  var body = {
    "event_id": current.u_alert_event_id.toString(),
    "ticket_number": current.number.toString(),
    "ticket_state": current.state.getDisplayValue(),
    "assigned_to": current.assigned_to.getDisplayValue(),
    "resolution_notes": current.close_notes.toString(),
    "updated_by": current.sys_updated_by.toString()
  };

  request.setRequestBody(JSON.stringify(body));
  request.execute();

})(current, previous);

// Power Automate receives this and:
// - Updates alert status when ticket closes
// - Posts resolution note to Teams channel
// - Marks alert as RESOLVED in state tracker
```

---

## Jira Auto-Creation

```
POWER AUTOMATE HTTP ACTION:
Method: POST
URI: https://[instance].atlassian.net/rest/api/3/issue
Headers:
  Authorization: Basic [base64(email:api-token)]
  Content-Type: application/json
Body:
{
  "fields": {
    "project": { "key": "NOC" },
    "summary": "[alert_type] — [site_name] — [metric]: [value][unit]",
    "description": {
      "type": "doc",
      "version": 1,
      "content": [{
        "type": "paragraph",
        "content": [{
          "type": "text",
          "text": "Auto-created from alert system.\n\nEvent ID: [event_id]\nSite: [site_name]\nSeverity: [severity]\nMetric: [metric] = [value] [unit] (threshold: [threshold])"
        }]
      }]
    },
    "issuetype": { "name": "Incident" },
    "priority": { "name": "[map: CRITICAL=Highest, HIGH=High, MEDIUM=Medium]" },
    "labels": ["auto-created", "[source_system]", "[alert_type]"],
    "customfield_alert_id": "[event_id]"
  }
}

EXTRACT FROM RESPONSE:
ticket_key: body('HTTP_Jira')?['key']
ticket_url: concat('https://[instance].atlassian.net/browse/', body('HTTP_Jira')?['key'])
```

---

## Incident Template Library

Generate these templates for your ITSM system:

```
INCIDENT TEMPLATE: Signal Degradation (RF/Telecom)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Short description: RF Signal Degradation — [Site] — [Metric]: [Value]
Category: Network / RF Infrastructure
Priority: [Based on severity]
Assignment group: NOC — RF Engineering
Description template:
  AUTO-CREATED: RF Hawkeye Alert
  
  Site: [site_name] ([site_id])
  Region: [region]
  Alert type: Signal degradation
  Metric: [metric] = [value] [unit]
  Threshold breached: [threshold] [unit]
  Alert time: [timestamp]
  
  Recommended first steps:
  1. Verify remote monitoring confirms degradation (not a sensor fault)
  2. Check for weather events at site location
  3. Verify power levels at site
  4. Check antenna alignment report (last: [date if known])
  5. Contact site maintenance if physical inspection needed
  
  Alert Event ID: [event_id]
  Source: RF Hawkeye

INCIDENT TEMPLATE: Equipment Failure
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Short description: Equipment Failure — [Site] — [Component]
Category: Infrastructure
Priority: Critical
Assignment group: NOC — Field Operations
Description template:
  AUTO-CREATED: Equipment failure detected
  [Same structure as above]
```

---

## Deduplication Logic

Prevent creating multiple tickets for the same ongoing alert:

```
BEFORE CREATING TICKET:
Check SharePoint/Dataverse state table for existing open incidents:
  Filter: event_id = [current event_id] AND status != 'RESOLVED'

If existing ticket found:
  - Do NOT create new ticket
  - Add comment to existing ticket: "Alert re-triggered at [timestamp]"
  - Post update to Teams: "Alert re-triggered — existing ticket [number]"

If no existing ticket:
  - Create new ticket
  - Store ticket number in state table
```

---

## Example Invocations

> "Automatically create a ServiceNow incident every time a CRITICAL alert fires"
→ Full HTTP action with field mapping + ticket URL back to Teams.

> "When the ServiceNow ticket is resolved, mark the alert as resolved in Teams"
→ ServiceNow Business Rule → Power Automate webhook → state update.

> "We use Jira, not ServiceNow — how do we create incidents from alerts?"
→ Jira REST API HTTP action with field mapping.

> "We're creating duplicate tickets for the same ongoing alert — how do we stop it?"
→ Deduplication logic with state table check before creation.
