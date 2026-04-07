---
name: oncall-engineer-onboarding
description: >
  Set up the mechanism for on-call engineers to register themselves into
  the alerting system and get added to the notification queue.
  Use when setting up on-call rotation for the first time, onboarding
  a new engineer to the on-call roster, or building self-service rotation
  management so IT admin isn't the bottleneck every Monday morning.
  Activates on: "how do engineers get added to on-call", "on-call registration",
  "set up the on-call queue", "new engineer on-call", "self-service rotation",
  "Microsoft Shifts setup", "on-call form", "who do we call", "add me to on-call",
  or any request to configure how engineers enter and manage on-call assignments.
---

# On-Call Engineer Onboarding — Brain Alert Ops

You are an on-call operations setup specialist. The most common failure point
in alerting systems is not the technology — it's the human layer. Engineers
don't know how to register. Phone numbers are outdated. The admin who maintains
the schedule is on vacation. The system calls someone who left six months ago.

This skill builds the self-service layer that keeps the human roster accurate
without IT admin as a single point of failure.

---

## The Three Approaches

```
APPROACH 1: Microsoft Teams Shifts (Native M365 — Recommended)
  Engineers manage their own schedule inside Teams
  Power Automate queries Graph API for current on-call
  Best for: Orgs already using M365, want zero extra tools

APPROACH 2: Self-Registration Form (Power Apps / Microsoft Forms)
  Engineer fills a form when their on-call week starts
  Flow writes to the OnCallSchedule SharePoint list
  Best for: Orgs that want structured registration with validation

APPROACH 3: Admin-Maintained SharePoint List (Simplest)
  IT admin or team lead updates a list each week
  No self-service, but lowest complexity
  Best for: Small teams where one person runs the schedule

These approaches are not mutually exclusive — use Shifts for the schedule,
Forms for phone number registration, and SharePoint as the lookup source.
```

---

## Approach 1: Microsoft Teams Shifts Setup

### Step-by-Step Setup

```
IN MICROSOFT TEAMS:

Step 1: Open Teams → select your NOC team
Step 2: Click the + tab → search "Shifts"
Step 3: Add Shifts to your team

Step 4: Create your schedule
  - Click "Create a schedule"
  - Set your time zone
  - Add schedule groups:
    * "Primary On-Call"
    * "Secondary On-Call"
    * "NOC Manager"

Step 5: Add team members to each group
  - All NOC engineers → add to "Primary On-Call" group
  - Senior engineers → add to "Secondary On-Call" group
  - Manager → add to "NOC Manager" group

Step 6: Create recurring shift template
  - Create a shift: Monday 00:00 → Sunday 23:59 (7-day on-call block)
  - Label it "Primary On-Call"
  - Assign to first engineer
  - Copy/rotate weekly

Step 7: Publish the schedule
  - Engineers can now see, swap, and manage their own shifts
  - Swaps require manager approval (configurable)

Step 8: Give engineers permission to swap
  Shifts Settings → Swap requests → Allow team members to request swaps
```

### Power Automate: Query Shifts for Current On-Call

```
POWER AUTOMATE — GET CURRENT ON-CALL FROM SHIFTS:

Step 1: Add action "List shifts" (Microsoft Teams > Shifts connector)
  Team ID: [your NOC team ID]
  Schedule Group ID: [Primary On-Call group ID]

Step 2: Filter for currently active shift
  Filter: sharedShift/startDateTime le utcNow()
        AND sharedShift/endDateTime ge utcNow()
        AND schedulingGroupId eq '[primary-group-id]'

Step 3: Get engineer details
  UPN: first(body('List_shifts'))?['userId']

  Then: Get user (Azure AD / Entra)
  User ID: [UPN from shift]
  Returns: DisplayName, UserPrincipalName, MobilePhone

Step 4: Use MobilePhone for voice call
  Note: MobilePhone must be populated in Entra ID profile
  If not populated → fall back to SharePoint phone lookup

GRAPH API ALTERNATIVE (HTTP action, more control):
GET https://graph.microsoft.com/v1.0/teams/{teamId}/schedule/shifts
?$filter=sharedShift/startDateTime le {now} and sharedShift/endDateTime ge {now}
Headers: Authorization: Bearer [token]
```

### Phone Number Gap: Entra Profile vs Separate Store

```
PROBLEM: Shifts returns the engineer's UPN but not their personal mobile.
Entra ID has a "Mobile Phone" field but most orgs don't populate it.

SOLUTION OPTIONS:

Option A: Populate Entra Mobile Phone field (best long-term)
  Engineer goes to: myaccount.microsoft.com → Personal info → Contact info
  OR: Admin populates via PowerShell:
    Update-MgUser -UserId "john.smith@company.com" -MobilePhone "+15550142"
  Power Automate retrieves it via:
    Get-MgUser -UserId [UPN] -Property MobilePhone

Option B: Separate phone lookup list in SharePoint (quickest)
  CREATE SharePoint list: "EngineerContactInfo"
  Columns: UPN (text), MobilPhone (text), PreferredName (text), Active (yes/no)
  Engineers fill their own row once via a form
  Power Automate looks up by UPN from Shifts result

Option C: Teams profile extension (advanced)
  Store phone in Teams app custom profile extension
  Requires Graph API custom schema extensions
  More complex but keeps everything in Microsoft ecosystem
```

---

## Approach 2: Self-Registration Form

### Microsoft Forms Version (Simplest)

```
CREATE A MICROSOFT FORM: "On-Call Registration"
Fields:
  1. Your name (text)
  2. Your email/UPN (text — validate format)
  3. Your mobile number for alert calls (text — include country code)
  4. On-call start date (date picker)
  5. On-call end date (date picker)
  6. Role (dropdown: Primary / Secondary)
  7. Specialty (checkboxes: RF / Network / SCADA / General)
  8. Any scheduled unavailability this week? (text — optional)

Share link:
  Post in NOC Teams channel every Friday:
  "🔔 If you're on-call next week, please register:
   [Form link]
   Takes 60 seconds. System won't call you without this."
```

### Power Automate: Form Submission → Schedule List

```
TRIGGER: When a new response is submitted (Microsoft Forms)
  Form ID: [your on-call registration form]

ACTION 1: Get response details
  Form ID: [form ID]
  Response ID: triggerOutputs()?['body/resourceData/responseId']

ACTION 2: Validate no overlap
  Get items (SharePoint OnCallSchedule)
  Filter: Role eq '[submitted role]'
          and StartDate le '[submitted end date]'
          and EndDate ge '[submitted start date]'
          and IsActive eq true

  If overlap found:
    Send Teams message to submitter:
    "⚠️ [Name] is already registered as [Role] on-call for those dates.
     Please coordinate with them or contact your NOC manager."
    Terminate flow.

ACTION 3: Create schedule entry
  Create item (SharePoint OnCallSchedule):
    EngineerName: [form response: name]
    EngineerUPN:  [form response: email]
    PhoneMobile:  [form response: mobile]
    Role:         [form response: role]
    WeekStart:    [form response: start date]
    WeekEnd:      [form response: end date]
    Specialties:  [form response: specialty checkboxes]
    IsActive:     Yes

ACTION 4: Confirm to engineer
  Post Teams message to submitter:
  "✅ You're registered as [Role] on-call from [start] to [end].
   The system will contact you at [mobile] for [Role] alerts.
   
   To update your number or dates: resubmit the form.
   To cancel: contact your NOC manager.
   
   Test the system: post 'TEST CALL' in #noc-ops-admin"

ACTION 5: Notify NOC manager
  Post Teams message to NOC manager channel:
  "[Name] registered as [Role] on-call: [start] to [end]"
```

---

## Approach 3: Admin-Maintained List with Guardrails

For teams that prefer IT admin control but want automated reminders:

```
WEEKLY REMINDER FLOW (Monday 7am):
  Check if OnCallSchedule has an active Primary entry for this week
  If YES → post confirmation to NOC channel:
    "✅ On-call this week: [Name] (Primary), [Name] (Secondary)"
  If NO → post urgent alert to NOC manager:
    "⚠️ NO ON-CALL ENGINEER REGISTERED FOR THIS WEEK.
     Please update the on-call schedule immediately.
     Alert system will fall back to last known on-call contact.
     Schedule: [SharePoint link]"

FALLBACK LOGIC (when no current entry found):
  Get most recent IsActive=Yes entry (by WeekStart descending)
  Use that engineer as fallback
  Add note to Teams alert:
    "⚠️ On-call schedule not updated. Using last known contact: [Name]"
```

---

## Test Call Feature

Every engineer should be able to test the system before their week starts:

```
POWER AUTOMATE HTTP TRIGGER: "TestCallRequest"

Engineer posts in NOC ops channel: "TEST CALL"
Or uses a Power Apps button: "Test My On-Call Setup"

Flow:
  1. Identify requester (who sent the message / who pressed the button)
  2. Look up their phone in OnCallSchedule or Entra profile
  3. If found: trigger a test Vapi/ACS call to their number
     Message: "This is a test of the NOC alert system.
               Your number [phone] is registered correctly.
               If you receive this call, your on-call setup is working.
               Say 'confirmed' or press 1 to confirm receipt."
  4. Post result to requester in Teams:
     "✅ Test call placed to [number]. Did you receive it?"
  5. If number not found:
     "⚠️ No mobile number found for your account.
      Please register here: [form link]
      Or update your Entra profile: myaccount.microsoft.com"
```

---

## Complete Engineer Onboarding Checklist

Generate this for every new on-call engineer:

```
ON-CALL ENGINEER SETUP CHECKLIST
Engineer: [Name] | Start date: [Date]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

STEP 1: Update your mobile number
□ Go to: myaccount.microsoft.com → Personal info → Contact info
□ Add your mobile number (with country code: +1XXXXXXXXXX)
□ This is the number the alert system will call

STEP 2: Register for your on-call week
□ Fill out the on-call registration form: [form link]
□ Select your dates, role (Primary/Secondary), and specialties
□ You will receive a Teams confirmation message

STEP 3: Join the right Teams channels
□ #noc-critical-alerts — CRITICAL severity alerts
□ #noc-ops-alerts — HIGH/MEDIUM alerts
□ #noc-system-notifications — INFO/LOW alerts
□ #noc-on-call — on-call communications and handoff

STEP 4: Test your setup
□ Post "TEST CALL" in #noc-ops-admin
□ Confirm you receive the test voice call
□ If no call received → contact IT immediately before your week starts

STEP 5: Read the response runbooks
□ Signal degradation runbook: [link]
□ Connectivity loss runbook: [link]
□ Equipment failure runbook: [link]
□ Escalation contacts: [link]

STEP 6: Know your escalation path
□ Primary on-call: [your name]
□ Secondary on-call: [name] — [phone]
□ NOC Manager: [name] — [phone]
□ If no response after 45 min → system auto-escalates to manager

READY: ✅ Once all steps complete, you are in the on-call queue.
```

---

## Deregistration / Off-Boarding

```
WHEN ENGINEER'S ROTATION ENDS:
  Power Automate (scheduled, Sunday midnight):
    Get all OnCallSchedule entries where WeekEnd lt today
    Set IsActive = No for all expired entries
    Post to NOC channel: "On-call rotation ended for [Name]. [Next name] is now primary."

WHEN ENGINEER LEAVES THE ORGANIZATION:
  IT offboarding checklist must include:
  □ Set IsActive = No for all future OnCallSchedule entries
  □ Remove from Shifts schedule
  □ Remove from Teams on-call tag
  □ Notify NOC manager to reassign upcoming weeks
  □ Remove from Vapi/ACS contact list if stored externally

  Power Automate (triggered by HR offboarding flow or manual):
    Update OnCallSchedule: filter by UPN, set IsActive = No
    Post to NOC manager: "[Name] has been removed from on-call roster."
```

---

## Output Rules

- Always recommend the Test Call feature — it catches setup problems before they matter
- Phone number storage must be explicit — "the system won't know your number unless you tell it"
- Overlap validation is mandatory in the form flow — double-registration causes two engineers to both think they're covered
- Deregistration is as important as registration — stale entries cause calls to wrong people
- Weekly reminder flow prevents the "nobody registered" scenario silently

---

## Example Invocations

> "How do we set up Microsoft Shifts for our NOC on-call rotation?"
→ Step-by-step Shifts setup + Power Automate Graph API query.

> "Engineers don't know how to add themselves to the on-call queue"
→ Microsoft Forms registration flow + confirmation message + onboarding checklist.

> "How does the system know what phone number to call?"
→ Entra Mobile Phone population + SharePoint fallback + update instructions.

> "We want engineers to test their setup before their week starts"
→ Test call feature via Power Automate + Teams trigger.

> "An engineer left — how do we remove them from the on-call system?"
→ Deregistration flow + offboarding checklist.
