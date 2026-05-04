# Workflow Map

## Main Flow

```text
Telegram Trigger
→ Edit Fields
→ Data Table: Get row(s) / Conflict Memory
→ Merge
→ AI Agent
→ Structured Output Parser
→ Code in JavaScript
→ Main Switch
```

## Main Switch Outputs

```text
clarification
show_schedule
booking
reschedule
cancel
availability
fallback
```

## Clarification

```text
needs_clarification = true
→ Telegram: Send clarification_question
→ STOP
```

## Show Schedule

```text
show_schedule
→ Google Calendar: Get Many
→ Code: Format schedule
→ Telegram: Send schedule
→ STOP
```

## Booking

```text
booking
→ Google Calendar: Schedule check1
→ Merge1
→ Code: Conflict checker
→ IF hasConflict
```

### Booking — No Conflict

```text
IF false
→ Google Calendar: Create event
→ Telegram: Booking confirmation
→ STOP
```

### Booking — Conflict

```text
IF true
→ Data Table: Delete old waiting rows
→ Data Table: Insert row pending_action = replace_conflicts_decision
→ Telegram: Ask replace or choose new time
→ STOP
```

### Booking Conflict — User Confirms

```text
Telegram Trigger
→ Edit Fields
→ Conflict Memory
→ Merge
→ AI Agent
→ Code in JavaScript
→ Main Switch: reschedule
→ IF confirmed pending replacement
→ Code1
→ Google Calendar: Delete event(s)
→ Code: Prepare one create item
→ Google Calendar: Create event
→ Telegram: Confirmation
→ Data Table: Delete pending row
→ STOP
```

### Booking Conflict — User Gives New Time

```text
Telegram Trigger
→ AI Agent / Code override
→ Main Switch: booking
→ Booking flow again
```

## Reschedule

```text
reschedule
→ IF confirmed pending replacement
```

### Reschedule — Confirmed Pending Replacement

```text
TRUE
→ Code1
→ Google Calendar: Delete event(s)
→ Code: Prepare one create item
→ Google Calendar: Create event
→ Telegram: Confirmation
→ Data Table: Delete pending row
→ STOP
```

### Reschedule — Normal Path

```text
FALSE
→ Google Calendar: Schedule Check Reschedule
→ Code: Count found events
→ Switch1
```

## Switch1

Outputs:

```text
single
multiple
none
```

### Reschedule Single

```text
single
→ Google Calendar: Delete event
→ Google Calendar: Create event
→ Telegram: Confirmation
→ STOP
```

### Reschedule Multiple

```text
multiple
→ Data Table: Delete old waiting rows
→ Data Table: Insert row pending_action = replace_conflicts_decision
→ Telegram: Ask confirmation
→ STOP
```

### Reschedule None

```text
none
→ Telegram: I couldn’t find that event. Which event do you want to reschedule?
→ STOP
```

## Cancel

Current target map:

```text
cancel
→ Code: Normalize cancel request
→ Google Calendar: Get Many
→ Code: Filter cancel matches
→ Cancel Switch
```

### Cancel Switch Outputs

```text
single
multiple
none
```

### Cancel None

```text
none
→ Telegram: No events found
→ STOP
```

### Cancel Single

```text
single
→ Data Table: Delete old waiting rows
→ Data Table: Insert row pending_action = confirm_cancel
→ Telegram: Confirm delete?
→ STOP
```

### Cancel Multiple

```text
multiple
→ Data Table: Delete old waiting rows
→ Data Table: Insert row pending_action = select_cancel_target
→ Telegram: List events, ask number/all
→ STOP
```

### Cancel Confirm Single

```text
Telegram reply yes
→ Conflict Memory finds confirm_cancel
→ AI Agent returns cancel confirmed true
→ Delete target_event_id
→ Telegram: Deleted
→ Data Table: Delete pending row
→ STOP
```

### Cancel Select Target

```text
Telegram reply number/all
→ Conflict Memory finds select_cancel_target
→ AI Agent returns cancel confirmed true
→ Code: Resolve selected IDs
→ Google Calendar: Delete event(s)
→ Telegram: Deleted
→ Data Table: Delete pending row
→ STOP
```

## Availability

Target map:

```text
availability
→ Google Calendar: Get Many
→ Code: Determine free/busy
→ Telegram: Send result
→ STOP
```

## Fallback

Target map:

```text
fallback
→ Telegram: I couldn’t understand that scheduling request. Please rephrase it.
→ STOP
```

## Data Table Cleanup Pattern

Before saving a new pending row:

```text
Data Table: Delete old waiting rows for same sessionId
→ Data Table: Insert new pending row
```

After completing a pending action:

```text
Telegram confirmation
→ Data Table: Delete exact pending row by id
```


## Latest Workflow Update — Cancel Single Confirm Path Built
The cancel branch now has a split immediately after Main Switch: cancel.

Updated cancel entry:

```text
Main Switch: cancel
→ IF Confirmed Cancel
```

### IF Confirmed Cancel

Condition:

```text
{{ $json.output.confirmed }} is equal to true
```

TRUE output:

```text
confirmed_cancel
```

FALSE path — new cancel request:

```text
IF false
→ Code: Normalize Cancel Request
→ Google Calendar: Get Many
→ Code: Filter Cancel Matches
→ Cancel Switch
```

TRUE path — user confirmed existing cancel request:

```text
IF true
→ Code: Prepare Confirm Cancel Delete
→ Google Calendar: Delete Event
→ Data Table: Delete pending row by exact row ID
→ Telegram: Deleted confirmation
→ STOP
```

### Cancel Single — completed

```text
Cancel Switch: single
→ Data Table: Delete old waiting rows
→ Data Table: Insert row pending_action = confirm_cancel
→ Telegram: Confirm delete?
→ STOP
```

### Cancel Confirm Single — completed

```text
Telegram reply yes
→ Conflict Memory finds confirm_cancel
→ AI Agent returns intent = cancel, confirmed = true
→ Main Switch: cancel
→ IF Confirmed Cancel TRUE
→ Prepare Confirm Cancel Delete
→ Delete target_event_id
→ Delete pending row by id
→ Telegram: Deleted confirmation
→ STOP
```
