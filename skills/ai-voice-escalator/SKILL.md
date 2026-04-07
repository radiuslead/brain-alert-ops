---
name: ai-voice-escalator
description: >
  Build AI voice call escalation for critical operational alerts.
  Use when a Teams notification is not enough and on-call engineers need
  to be reached by phone with spoken alert details. Covers Azure Communication
  Services, Vapi, and Synthflow as implementation options based on what
  the organization can get approved.
  Activates on: "call the on-call engineer", "voice escalation", "AI voice alert",
  "phone call for critical alerts", "ACS calling", "Vapi outbound call",
  "automated phone alert", "on-call phone notification", or any request
  to trigger a voice call as part of an alerting workflow.
---

# AI Voice Escalator — Brain Alert Ops

You are an operational voice escalation specialist. You know that a Teams
notification is a passive medium — engineers have to be watching. A phone call
is an active interrupt. For CRITICAL alerts, the difference between passive
and active notification is often the difference between a 15-minute response
and a 3am outage that nobody knew about until morning.

Modern AI voice platforms have replaced old TTS dial-out systems. The on-call
engineer no longer hears a robotic read-out — they have a real conversation
with an AI agent that can take acknowledgment commands, transfer to colleagues,
or repeat alert details on request.

---

## The Three Implementation Paths

Enterprise alerting always has the same tension: the ideal solution vs the
approved solution. This skill covers all three paths.

```
PATH A: Azure Communication Services (ACS)
  Best for:   Orgs fully in Azure ecosystem
  Approval:   Requires Azure subscription + ACS resource provisioning
  Cost:       ~$0.0085/min outbound PSTN + Azure AI Speech costs
  Timeline:   2-4 weeks to provision and get approved
  Complexity: High — requires developer and Azure admin

PATH B: Vapi (or Retell AI / Bland AI)
  Best for:   Orgs that need it working this week, have a developer
  Approval:   Third-party SaaS — needs security/procurement review
  Cost:       ~$0.05-0.15/min all-in
  Timeline:   1-3 days to set up and integrate
  Complexity: Medium — API-first, needs coding

PATH C: Synthflow (no-code)
  Best for:   Orgs without a developer, need fastest path
  Approval:   Third-party SaaS — simpler procurement than ACS
  Cost:       Subscription-based (~$500/month for business tier)
  Timeline:   Same day to 2 days
  Complexity: Low — no-code flow builder

PRAGMATIC RECOMMENDATION:
  If ACS is in procurement → use Vapi/Synthflow now, migrate to ACS later
  If budget is constrained → Vapi at $0.15/min is cheapest at low volume
  If no developer available → Synthflow no-code
  If full Azure shop and approved → ACS is the enterprise-native choice
```

---

## Path A: Azure Communication Services Implementation

```
ARCHITECTURE:
Power Automate (CRITICAL alert) 
  → HTTP POST to Azure Logic App or Function
  → ACS Call Automation API
  → Outbound PSTN call to on-call engineer's phone
  → Azure AI Speech reads alert summary
  → DTMF or voice input: press 1 = acknowledge, press 2 = escalate
  → Result posted back to Teams + ServiceNow updated

POWER AUTOMATE → ACS CALL:
HTTP Action:
  Method: POST
  URI: https://[your-acs-resource].communication.azure.com/calling/callConnections
  Headers:
    Authorization: Bearer [ACS token]
    Content-Type: application/json
  Body:
    {
      "targets": [{
        "kind": "phoneNumber",
        "phoneNumber": { "value": "[on-call phone number]" }
      }],
      "sourceCallerIdNumber": { "value": "[your ACS phone number]" },
      "callbackUri": "[your webhook URL to receive call events]",
      "operationContext": "[alert event_id]"
    }

ONCE CALL CONNECTS — play TTS via ACS:
POST /callConnections/{callConnectionId}/playText
{
  "textToPlay": "Critical alert: [site_name]. [metric] value of [value] [unit] has breached threshold of [threshold]. This is alert [event_id]. Press 1 to acknowledge. Press 2 to escalate to your manager.",
  "voiceName": "en-US-JennyNeural",
  "loop": false
}

HANDLE DTMF INPUT:
POST /callConnections/{callConnectionId}/startRecognizing
{
  "recognizeInputType": "dtmf",
  "dtmfConfigurations": {
    "interToneTimeoutInSeconds": 5,
    "maxTonesToCollect": 1
  }
}

ON RESULT:
  "1" → mark alert acknowledged in state tracker, update Teams card
  "2" → trigger escalation to next on-call, update Teams card
  No input → retry call after 5 minutes, then escalate
```

---

## Path B: Vapi Implementation

```
ARCHITECTURE:
Power Automate (CRITICAL alert)
  → HTTP POST to Vapi API
  → Vapi makes outbound call
  → AI voice agent speaks alert + takes spoken acknowledgment
  → Webhook posts result back to Power Automate
  → Teams + ITSM updated

POWER AUTOMATE HTTP ACTION — TRIGGER VAPI CALL:
Method: POST
URI: https://api.vapi.ai/call/phone
Headers:
  Authorization: Bearer [VAPI_API_KEY]
  Content-Type: application/json
Body:
{
  "assistantId": "[your-on-call-assistant-id]",
  "customer": {
    "number": "[on-call engineer phone number]",
    "name": "[engineer name]"
  },
  "assistantOverrides": {
    "firstMessage": "Hi [engineer name], this is the NOC alert system. There is a CRITICAL alert at [site_name]. [metric] has dropped to [value] [unit], breaching the [threshold] threshold. Say 'acknowledged' to confirm you are responding, or say 'escalate' to page your backup.",
    "variableValues": {
      "site_name": "[site_name from event]",
      "metric": "[metric from event]",
      "value": "[value from event]",
      "unit": "[unit from event]",
      "threshold": "[threshold from event]",
      "event_id": "[event_id]"
    }
  },
  "metadata": {
    "event_id": "[event_id]",
    "alert_type": "[alert_type]"
  }
}

VAPI ASSISTANT CONFIGURATION:
{
  "name": "NOC Alert Agent",
  "model": {
    "provider": "anthropic",
    "model": "claude-sonnet-4-20250514",
    "systemPrompt": "You are a NOC alert notification agent. Your job is to notify on-call engineers about critical alerts, confirm they have received and understood the alert, and record their acknowledgment or escalation request. Be brief and professional. The engineer is likely being woken up or interrupted. Get to the point. After they acknowledge, thank them and end the call. If they say anything other than acknowledge or escalate, repeat the alert details once and ask again."
  },
  "voice": {
    "provider": "11labs",
    "voiceId": "rachel"
  },
  "endCallFunctionEnabled": true,
  "endCallMessage": "Alert has been logged. Stay safe.",
  "serverUrl": "[your webhook URL for call results]"
}

VAPI WEBHOOK RESULT HANDLING (Power Automate HTTP trigger):
When Vapi posts result:
  "status": "ended"
  "endedReason": "assistant-ended-call"
  "transcript": "[full call transcript]"

Extract acknowledgment from transcript:
  if transcript contains "acknowledged" → update alert status
  if transcript contains "escalate" → trigger escalation sequence
  if endedReason = "no-answer" → retry in 5 minutes
```

---

## Path C: Synthflow (No-Code)

```
SETUP IN SYNTHFLOW:
1. Create account at synthflow.ai
2. Create a new "Outbound" agent
3. Configure:
   - Voice: choose from realistic voices
   - First message: [use template below]
   - Goal: "Notify engineer of critical alert and record acknowledgment"
   - End condition: engineer says "acknowledged" or "escalate"
4. Get the Synthflow API endpoint for your agent
5. In Power Automate: HTTP POST to Synthflow endpoint when CRITICAL alert fires

SYNTHFLOW FIRST MESSAGE TEMPLATE:
"Hello [name]. This is an automated alert from your NOC system.
A CRITICAL alert has been triggered at [site_name].
The alert type is [alert_type].
[metric] has reached [value] [unit].
Please say 'acknowledged' if you are responding to this alert,
or say 'escalate' if you need to pass this to your backup."

POWER AUTOMATE → SYNTHFLOW:
HTTP POST to Synthflow API with:
{
  "agentId": "[your-synthflow-agent-id]",
  "phoneNumber": "[on-call number]",
  "variables": {
    "name": "[engineer name]",
    "site_name": "[site]",
    "alert_type": "[type]",
    "metric": "[metric]",
    "value": "[value]",
    "unit": "[unit]"
  }
}
```

---

## Escalation Chain Logic

```
ESCALATION SEQUENCE (applies to all paths):

T+0:    CRITICAL alert fires
T+0:    Teams adaptive card posted + voice call to Primary on-call
T+15:   If no acknowledgment → call Secondary on-call
T+30:   If no acknowledgment → call NOC Manager
T+45:   If no acknowledgment → call Director of Infrastructure Ops
T+60:   If no acknowledgment → send SMS to all + post @channel in Teams

POWER AUTOMATE ESCALATION TIMER:
After triggering voice call:
  Do Until: alert_acknowledged = true OR escalation_level > 3
    Delay: 15 minutes
    If not acknowledged:
      Increment escalation_level
      Get next on-call contact for this level
      Trigger new voice call
      Post escalation update to Teams
```

---

## Example Invocations

> "We need the on-call engineer to get a phone call for CRITICAL alerts"
→ Recommend path based on their environment + full implementation for that path.

> "We can't get ACS approved — what's the fastest alternative?"
→ Path B (Vapi) with complete Power Automate integration.

> "We have no developer — how do we build voice escalation?"
→ Path C (Synthflow no-code) with step-by-step setup.

> "How do we escalate if the on-call engineer doesn't pick up?"
→ Escalation chain logic with Power Automate timer and level tracking.

> "We want the AI to have a real conversation, not just read a message"
→ Vapi with Claude-powered assistant that handles natural language acknowledgment.
