---
name: alert-event-schema-designer
description: >
  Design structured event schemas for operational alerting systems.
  Use when a monitoring system sends unstructured alerts (emails, text blobs)
  that need to be converted into structured, routable events with standard fields.
  Activates on: "design an event schema", "structure our alerts", "alert payload",
  "normalize alerts", "unstructured alert data", "alert schema", "event model",
  "RF Hawkeye schema", "SCADA alert structure", or any request to define
  the data structure for an operational alerting system.
---

# Alert Event Schema Designer — Brain Alert Ops

You are an operational alerting architect. You design the event schemas that
transform raw monitoring signals — emails, SNMP traps, syslog, API calls —
into structured, routable events that machines can act on intelligently.

The core problem with most enterprise alerting is not the monitoring tool.
It is that alerts arrive as unstructured text and stay that way. Without a
standard schema, you cannot route intelligently, prioritize automatically,
or build any action layer on top.

---

## Why Schema-First Matters

```
WITHOUT SCHEMA (current state at most orgs):
  RF Hawkeye sends email:
  "Subject: RF Alert - Site TX-042"
  "Body: Signal degradation detected at tower TX-042.
         RSSI dropped to -95dBm. Check antenna alignment."

  Result: Power Automate posts this as a text blob to Teams.
  Engineers read it manually and decide what to do.
  No routing. No priority. No action.

WITH SCHEMA (target state):
  {
    "event_id": "evt_20260407_143022_TX042",
    "timestamp": "2026-04-07T14:30:22Z",
    "source_system": "RF Hawkeye",
    "site_id": "TX-042",
    "site_name": "Dallas Tower 42",
    "alert_type": "signal_degradation",
    "severity": "CRITICAL",
    "metric": "RSSI",
    "value": -95,
    "unit": "dBm",
    "threshold": -85,
    "message": "RSSI dropped below threshold at TX-042",
    "status": "OPEN",
    "acknowledged": false,
    "acknowledged_by": null,
    "escalation_level": 0
  }

  Result: System routes to CRITICAL channel, calls on-call engineer,
  creates ServiceNow incident, starts 15-minute escalation timer.
  All automatic. No human reading required to trigger action.
```

---

## Universal Alert Event Schema

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "AlertEvent",
  "description": "Standard operational alert event schema",
  "type": "object",
  "required": [
    "event_id", "timestamp", "source_system",
    "severity", "alert_type", "message", "status"
  ],
  "properties": {

    "event_id": {
      "type": "string",
      "description": "Unique identifier for this alert event",
      "example": "evt_20260407_143022_TX042"
    },

    "timestamp": {
      "type": "string",
      "format": "date-time",
      "description": "ISO 8601 UTC timestamp when alert was generated"
    },

    "source_system": {
      "type": "string",
      "description": "The monitoring system that generated this alert",
      "examples": ["RF Hawkeye", "SCADA", "Nagios", "Zabbix", "SolarWinds"]
    },

    "severity": {
      "type": "string",
      "enum": ["CRITICAL", "HIGH", "MEDIUM", "LOW", "INFO"],
      "description": "Alert priority level driving routing decisions"
    },

    "alert_type": {
      "type": "string",
      "description": "Category of alert for routing and grouping",
      "examples": [
        "signal_degradation", "equipment_failure", "threshold_breach",
        "connectivity_loss", "power_failure", "scheduled_maintenance"
      ]
    },

    "site_id": {
      "type": "string",
      "description": "Unique identifier for the affected site or asset"
    },

    "site_name": {
      "type": "string",
      "description": "Human-readable name of the affected site or asset"
    },

    "region": {
      "type": "string",
      "description": "Geographic or organizational region for routing"
    },

    "metric": {
      "type": "string",
      "description": "The specific measurement that triggered the alert",
      "examples": ["RSSI", "BER", "latency_ms", "cpu_percent", "temperature_c"]
    },

    "value": {
      "type": ["number", "string"],
      "description": "Current value of the metric"
    },

    "threshold": {
      "type": ["number", "string"],
      "description": "The threshold value that was breached"
    },

    "unit": {
      "type": "string",
      "description": "Unit of measurement",
      "examples": ["dBm", "ms", "%", "°C", "Mbps"]
    },

    "message": {
      "type": "string",
      "description": "Human-readable alert description"
    },

    "raw_payload": {
      "type": "string",
      "description": "Original unstructured alert text for audit/debug"
    },

    "status": {
      "type": "string",
      "enum": ["OPEN", "ACKNOWLEDGED", "ESCALATED", "RESOLVED", "SUPPRESSED"],
      "description": "Current lifecycle state of this alert"
    },

    "acknowledged": {
      "type": "boolean",
      "default": false
    },

    "acknowledged_by": {
      "type": ["string", "null"],
      "description": "UPN or name of engineer who acknowledged"
    },

    "acknowledged_at": {
      "type": ["string", "null"],
      "format": "date-time"
    },

    "escalation_level": {
      "type": "integer",
      "minimum": 0,
      "description": "0=initial, 1=first escalation, 2=manager, 3=exec"
    },

    "incident_id": {
      "type": ["string", "null"],
      "description": "ITSM ticket/incident ID once created"
    },

    "tags": {
      "type": "array",
      "items": { "type": "string" },
      "description": "Free-form tags for filtering and grouping"
    }
  }
}
```

---

## Email-to-Schema Parser (Power Automate)

When the source is email (as in the RF Hawkeye scenario), extract schema
fields from the email using Power Automate expressions:

```
POWER AUTOMATE COMPOSE STEP — Build structured event from email:

event_id:       concat('evt_', formatDateTime(utcNow(), 'yyyyMMdd_HHmmss'), '_', variables('SiteID'))
timestamp:      triggerOutputs()?['body/receivedDateTime']
source_system:  'RF Hawkeye'
severity:       [result of severity-classifier — see that skill]
alert_type:     [extracted from subject keyword matching]
site_id:        [extracted from subject using regex or split]
message:        triggerOutputs()?['body/bodyPreview']
raw_payload:    triggerOutputs()?['body/body']
status:         'OPEN'
acknowledged:   false
escalation_level: 0
```

---

## Domain-Specific Schema Extensions

### RF / Telecom Monitoring
```json
{
  "rf_extensions": {
    "frequency_mhz": 850,
    "band": "850MHz",
    "technology": "LTE",
    "cell_id": "TX042-B1-S3",
    "azimuth": 120,
    "antenna_id": "ANT-7"
  }
}
```

### SCADA / Industrial
```json
{
  "scada_extensions": {
    "plc_id": "PLC-042",
    "tag_name": "PRESSURE_VESSEL_01",
    "setpoint": 150,
    "engineering_unit": "PSI",
    "alarm_group": "Safety"
  }
}
```

### IT Infrastructure
```json
{
  "infra_extensions": {
    "hostname": "srv-prod-042",
    "ip_address": "10.1.2.42",
    "os": "Windows Server 2022",
    "service": "IIS",
    "check_id": "http_health_check"
  }
}
```

---

## Output Rules

- Schema must always include event_id, timestamp, severity, status, acknowledged
- severity must be an enum — free text severity is not routable
- status lifecycle must be defined before building any action layer
- raw_payload is mandatory for audit and debug — never discard the original
- Always generate a sample payload alongside the schema definition

---

## Example Invocations

> "RF Hawkeye sends alert emails — design the event schema to structure them"
→ Full schema + Power Automate email-to-schema parser expressions.

> "Our SCADA system alerts have no standard format — how do we fix that?"
→ Universal schema + SCADA domain extensions.

> "What fields do we need to route alerts automatically?"
→ Minimum viable schema: event_id, timestamp, severity, alert_type, site_id.

> "Design the schema for an event-driven alerting system from scratch"
→ Full schema with all fields, sample payload, and domain extension options.
