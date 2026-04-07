---
name: on-call-rotation-manager
description: >
  Build and manage on-call rotation schedules for operational alert systems.
  Use when alerts need to reach the right engineer based on rotation schedule,
  time of day, or specialty. Activates on: "on-call schedule", "who is on call",
  "rotation schedule", "on-call engineer lookup", "escalation contacts",
  "primary and backup on-call", "after hours contact", or any request
  to manage on-call routing for operational alerts.
---

# On-Call Rotation Manager — Brain Alert Ops

You are an on-call operations specialist. The best alerting system fails if it
calls the wrong person — or the right person at the wrong time. On-call rotation
management is the layer that connects alert severity and timing to the specific
human who should respond right now.

---

## On-Call Data Model

```
ON-CALL SCHEDULE RECORD:
{
  rotation_id:    "rot_noc_primary_2026_w15"
  team:           "NOC RF Engineering"
  role:           "primary" | "secondary" | "manager" | "executive"
  engineer_name:  "John Smith"
  engineer_upn:   "john.smith@company.com"
  phone_mobile:   "+1-555-0142"
  teams_id:       "[Entra Object ID for @mentions]"
  week_start:     "2026-04-06" (Monday)
  week_end:       "2026-04-12" (Sunday)
  coverage_start: "08:00"  (local time — blank = 24/7)
  coverage_end:   "20:00"
  timezone:       "America/Chicago"
  specialties:    ["RF", "SCADA"]  (route by specialty)
}
```

---

## SharePoint List: On-Call Schedule

```
CREATE SHAREPOINT LIST: "OnCallSchedule"
Columns:
  Title (text):              rotation_id
  Team (text):               NOC RF Engineering
  Role (choice):             Primary/Secondary/Manager/Executive
  EngineerName (text)
  EngineerUPN (text):        for Power Automate lookups
  PhoneMobile (text):        for voice call escalation
  TeamsObjectID (text):      for @mention in cards
  WeekStart (date):          start of coverage week
  WeekEnd (date):            end of coverage week
  CoverageStart (text):      "08:00"
  CoverageEnd (text):        "20:00"
  Timezone (text):           "America/Chicago"
  Specialties (text):        comma-separated
  IsActive (yes/no):         default Yes
```

---

## Power Automate: Look Up Current On-Call Engineer

```
POWER AUTOMATE — GET CURRENT ON-CALL:
Action: Get items (SharePoint)
List: OnCallSchedule
Filter:
  Role eq 'Primary'
  and IsActive eq 1
  and WeekStart le '[formatDateTime(utcNow(), yyyy-MM-dd)]'
  and WeekEnd ge '[formatDateTime(utcNow(), yyyy-MM-dd)]'
  and Team eq '[alert routing team — based on alert_type/region]'

Result: engineer's name, phone, UPN, TeamsObjectID

USE RESULT TO:
  - Populate voice call phone number
  - @mention in Teams adaptive card
  - Assign incident in ServiceNow
  - Set AcknowledgedBy expectation in state tracker
```

---

## Business Hours vs After-Hours Logic

```
POWER AUTOMATE EXPRESSION — Is it business hours?

Variables needed:
  currentHour: int(convertTimeZone(utcNow(), 'UTC', variables('EngineerTimezone'), 'HH'))
  currentDay:  dayOfWeek(convertTimeZone(utcNow(), 'UTC', variables('EngineerTimezone')))
  coverageStart: int(first(split(variables('CoverageStart'), ':')))
  coverageEnd:   int(first(split(variables('CoverageEnd'), ':')))

isBusinessHours expression:
and(
  greaterOrEquals(variables('currentHour'), variables('coverageStart')),
  less(variables('currentHour'), variables('coverageEnd')),
  not(or(
    equals(variables('currentDay'), 0),  // Sunday
    equals(variables('currentDay'), 6)   // Saturday
  ))
)

ROUTING BASED ON RESULT:
Business hours → alert goes to Primary on-call
After hours → alert goes to Primary on-call (24/7 for CRITICAL)
             → but escalation goes to Secondary faster (10 min vs 15 min)
```

---

## Escalation Contact Resolution

```
ESCALATION LEVEL LOOKUP:

Level 0 (initial):
  → Primary on-call for alert's team/specialty

Level 1 (15 min, no response):
  → Secondary on-call for alert's team/specialty

Level 2 (30 min, no response):
  → NOC Manager (role = 'Manager')

Level 3 (45 min, no response):
  → Director / Executive (role = 'Executive')
  → SMS all Level 0-2 contacts simultaneously

POWER AUTOMATE — GET CONTACT BY ESCALATION LEVEL:
Action: Get items (SharePoint)
Filter:
  Role eq '[role based on escalation level]'
  and IsActive eq 1
  and WeekStart le '[today]'
  and WeekEnd ge '[today]'
  and Team eq '[team]'
Select: EngineerName, PhoneMobile, EngineerUPN, TeamsObjectID
```

---

## Specialty-Based Routing

```
ALERT TYPE → TEAM ROUTING TABLE:
alert_type                → Team
signal_degradation        → NOC RF Engineering
equipment_failure         → NOC Field Operations
power_failure             → NOC Facilities
connectivity_loss         → NOC Network
scheduled_maintenance     → NOC Operations
security_event            → NOC Security

POWER AUTOMATE — ROUTE BY SPECIALTY:
Switch on alert.alert_type:
  Case "signal_degradation": team = "NOC RF Engineering"
  Case "equipment_failure":  team = "NOC Field Operations"
  Case "power_failure":      team = "NOC Facilities"
  Default:                   team = "NOC Operations"

Then look up on-call engineer for that team.
```

---

## On-Call Schedule Management

### Weekly rotation update template:

```
EMAIL TO OUTGOING ON-CALL ENGINEER (Friday):
Subject: Your on-call rotation ends Sunday night

Hi [Name],

Your on-call rotation ends Sunday at midnight.
[Incoming engineer name] takes over Monday morning.

Before handoff, please:
□ Confirm all open alerts are either resolved or handed off
□ Update the on-call notes in Teams if anything is ongoing
□ Confirm [incoming engineer] has your mobile number for emergencies

On-call notes channel: [Teams link]
Open alerts: [SharePoint link]

Thank you for your service this week.
NOC Operations
```

### Swap request template:

```
EMAIL TO NOC MANAGER:
Subject: On-call swap request — [Date]

Hi [Manager],

I'd like to swap my on-call rotation for [date] with [colleague name].
They have confirmed they can cover.

Current assignment: [my name] — [date range]
Proposed swap: [colleague name] covers [my date], I cover [their date]

Please update the schedule and confirm.

[Your name]
```

---

## Example Invocations

> "How do we make sure the alert calls the right engineer based on who's on call this week?"
→ SharePoint schedule list + Power Automate lookup expression.

> "CRITICAL alerts should go to primary on-call, but escalate to secondary faster after hours"
→ Business hours logic + modified escalation timers for after-hours.

> "Different alert types should go to different teams"
→ Specialty routing table with Switch action in Power Automate.

> "Build the on-call handoff email template"
→ Weekly handoff email for outgoing + incoming engineers.
