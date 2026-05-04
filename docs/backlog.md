# Backlog

## Project

AI Scheduler — n8n Telegram Calendar Automation

This document lists what still needs to be built, tested, cleaned, and improved.

## Current Priority

Finish the **cancel branch**.

The cancel branch is partially built but not complete.

## 1. Finish Cancel Branch

### Current Cancel Structure

Already started:

```text
Main Switch: cancel
→ Code: Normalize cancel request
→ Google Calendar: Get Many
→ Code: Filter cancel matches
→ Switch
```

The cancel Switch should route by:

```text
{{ $json.match_status }}
```

Expected outputs:

```text
single
multiple
none
```

## 2. Cancel Branch — None Path

### Goal

If no matching event is found, tell the user clearly.

### Required Flow

```text
Switch: none
→ Telegram: Send Message
→ STOP
```

### Message Example

```text
I couldn’t find any event matching that request.
```

Or more specific:

```text
I couldn’t find an event matching your request. Please send the event name, date, or time again.
```

### Notes

No Data Table state is needed for this path right now.

## 3. Cancel Branch — Single Match Path

### Goal

If exactly one matching event is found, do not delete immediately. Ask for confirmation first.

### Required Flow

```text
Switch: single
→ Data Table: Delete old waiting rows for this session
→ Data Table: Insert row
→ Telegram: Send confirmation question
→ STOP
```

### Pending Row to Save

Save into:

```text
pending_calendar_actions
```

Fields:

```text
sessionId
pending_action = confirm_cancel
title
start_datetime
end_datetime
target_event_id
status = waiting_for_user
created_at
```

### Confirmation Message Example

```text
I found this event:

April 24, 5:00 PM - 6:00 PM | Dinner

Do you want me to delete it?
```

## 4. Cancel Branch — Confirm Single Delete

### Goal

When the user replies “yes” to a `confirm_cancel` pending action, delete the saved target event.

### Required Flow

```text
Telegram Trigger
→ Edit Fields
→ Data Table: Get row(s)
→ Merge
→ AI Agent
→ Code in JavaScript
→ Main Switch: cancel
→ IF pending_action = confirm_cancel AND confirmed = true
→ Google Calendar: Delete event
→ Telegram: Send deleted confirmation
→ Data Table: Delete row(s)
```

### Delete Event ID

Use saved:

```text
target_event_id
```

### Confirmation Message Example

```text
Deleted: Dinner on April 24 at 5:00 PM.
```

### Cleanup

After successful deletion, delete the exact pending row by Data Table row ID.

Use:

```text
id = pending_row_id
```

## 5. Cancel Branch — Multiple Match Path

### Goal

If multiple events match the cancel request, list them and ask which one to delete.

### Required Flow

```text
Switch: multiple
→ Data Table: Delete old waiting rows for this session
→ Data Table: Insert row
→ Telegram: Send options list
→ STOP
```

### Pending Row to Save

Save into:

```text
pending_calendar_actions
```

Fields:

```text
sessionId
pending_action = select_cancel_target
candidate_event_ids
status = waiting_for_user
created_at
```

Optional fields:

```text
pending_message_type
title
start_datetime
end_datetime
```

### Candidate IDs

Store as stringified JSON:

```json
["event_id_1","event_id_2","event_id_3"]
```

### Telegram Message Example

```text
I found more than one matching event:

1) 5:00 PM - 6:00 PM | Dinner
2) 6:00 PM - 7:00 PM | Family time
3) 8:00 PM - 9:00 PM | Walk

Which one should I delete? Reply with a number or say all.
```

## 6. Cancel Branch — Select Target Reply

### Goal

When pending_action is `select_cancel_target`, the user can reply with:

```text
1
2
3
all
```

### Required Flow

```text
Telegram Trigger
→ Edit Fields
→ Data Table: Get row(s)
→ Merge
→ AI Agent
→ Code in JavaScript
→ Main Switch: cancel
→ IF pending_action = select_cancel_target AND confirmed = true
→ Code: resolve selected event IDs
→ Google Calendar: Delete event(s)
→ Telegram: Send deleted confirmation
→ Data Table: Delete row(s)
```

### Code Needed

A Code node should:

1. Read candidate event IDs from Data Table.
2. Read user reply from `chatInput`.
3. If user says a number, select that event ID.
4. If user says `all`, select all candidate event IDs.
5. Output one item per selected event ID.

Example output fields:

```text
event_id_to_delete
sessionId
pending_row_id
```

## 7. Cancel Branch — Reject Delete

### Goal

If user rejects deletion after confirmation, do not delete anything.

### Example User Replies

```text
no
cancel
do not delete
never mind
```

### Required Flow

```text
pending_action = confirm_cancel OR select_cancel_target
→ user rejects
→ Telegram: Send Message
→ Data Table: Delete row(s)
```

### Message Example

```text
Okay, I did not delete anything.
```

### Prompt Status

The prompt currently has a small pending cancel section.

It may need to be improved later to classify negative replies properly.

## 8. Cancel Branch — Calendar Error Handling

### Goal

Every Google Calendar delete/search node should have an error path.

### Required Pattern

```text
Google Calendar node error
→ Telegram: Send Message
→ Data Table: Delete row(s) if pending action exists
```

### Message Example

```text
Calendar error. Please try again.
```

## 9. Availability Branch

### Goal

Build the availability branch.

### Target Flow

```text
Main Switch: availability
→ Google Calendar: Get Many
→ Code: determine free/busy
→ Telegram: Send result
```

### Expected Behavior

If no events overlap:

```text
You are free at that time.
```

If events overlap:

```text
You are busy at that time:

5:00 PM - 6:00 PM | Dinner
```

### Required AI Fields

```text
range_start_datetime
range_end_datetime
```

## 10. Fallback Branch

### Goal

Handle unexpected outputs or unsupported requests.

### Target Flow

```text
Main Switch fallback
→ Telegram: Send Message
```

### Message Example

```text
I couldn’t understand that scheduling request. Please rephrase it.
```

## 11. Improve Pending Cancel Prompt Rules

### Goal

Prompt should better support:

```text
confirm_cancel
select_cancel_target
negative replies
number selection
all selection
```

### Important Rule

These pending actions are Data Table states only:

```text
confirm_cancel
select_cancel_target
replace_conflicts_decision
```

They must never appear in:

```text
pending_intent
```

## 12. Add Cancel-Specific Code Override If Needed

### Goal

If AI Agent incorrectly routes pending cancel replies, add deterministic Code node logic after AI Agent.

Possible cases:

```text
pending_action = confirm_cancel + user says yes → force intent = cancel, confirmed = true
pending_action = select_cancel_target + user replies number/all → force intent = cancel, confirmed = true
pending_action = confirm_cancel + user says no → force clarification or cancel cleanup path
```

### Current Status

Not built yet.

Build only if real testing shows AI is unreliable.

## 13. Test Full Workflow End-to-End

### Required Test Cases

#### Show Schedule

```text
Show my schedule today.
```

Expected:

```text
Correct date, sorted events, readable Telegram message.
```

#### Booking No Conflict

```text
Book appointment tomorrow at 3 PM.
```

Expected:

```text
Event created once.
Confirmation sent once.
```

#### Booking Conflict

```text
Book doctor appointment today from 5 to 6 PM.
```

Expected:

```text
Conflict detected.
Pending row inserted.
Question sent.
No event created until confirmation.
```

#### Booking Conflict Confirm

```text
Yes, replace them.
```

Expected:

```text
Saved conflicting events deleted.
New event created once.
Pending row deleted.
```

#### Booking Conflict New Time

```text
Try 7 PM instead.
```

Expected:

```text
Reroutes to booking with saved title and new time.
```

#### Reschedule Single

```text
Reschedule dinner today from 6 to 7 to walk time from 7 to 8.
```

Expected:

```text
One old event deleted.
One new event created.
Confirmation sent.
```

#### Reschedule Multiple

```text
I want to reschedule my events today. I need a new event between 5 and 11 PM named Relax.
```

Expected:

```text
Multiple events found.
Pending row inserted.
Question sent.
After confirmation, all saved events deleted and Relax created once.
```

#### Reschedule None

```text
Reschedule fake meeting today from 2 to 3 to test from 4 to 5.
```

Expected:

```text
No event found.
Telegram asks which event to reschedule.
```

#### Cancel Single

```text
Cancel dinner today at 6 PM.
```

Expected:

```text
One event found.
Confirmation requested.
No deletion until yes.
```

#### Cancel Multiple

```text
Cancel my events today between 5 and 9 PM.
```

Expected:

```text
Multiple events listed.
User asked to choose number or all.
```

#### Cancel Confirm

```text
Yes.
```

Expected:

```text
Target event deleted.
Pending row deleted.
Confirmation sent.
```

#### Availability

```text
Am I free tomorrow at 3 PM?
```

Expected:

```text
Correct busy/free response.
```

## 14. Check All Telegram Nodes for Duplicate Messages

### Goal

Prevent duplicate Telegram messages.

### Known Issue

Telegram sent multiple messages when previous node output multiple items.

### Fix Pattern

For message nodes after multi-item outputs:

```text
Settings → Execute Once = ON
```

Use only where appropriate.

## 15. Check All Google Calendar Create Nodes for Duplicate Creation

### Goal

Prevent duplicate calendar creation after multiple deletes.

### Fix Pattern

After multi-event delete:

```text
Code: Prepare one create item
→ Create Event
```

Create Event should receive exactly one item.

## 16. Check All Data Table Cleanup Nodes

### Goal

Avoid stale pending rows.

### Required cleanup points:

```text
after confirmed booking/reschedule replacement
after confirmed cancel
after rejected cancel
after calendar error
```

### Preferred Delete Method

Delete exact pending row by:

```text
id
```

not by sessionId, unless intentionally clearing old pending rows before insert.

## 17. Review Error Handling

### Goal

Add error handling to:

```text
Google Calendar Get Many
Google Calendar Delete
Google Calendar Create
Data Table Insert
Data Table Delete
Telegram Send Message
```

### Error Message Example

```text
Something went wrong with the calendar action. Please try again.
```

## 18. Add Logging Later

### Goal

Optionally add logging for debugging.

Possible log fields:

```text
timestamp
sessionId
chatInput
intent
pending_action
workflow_branch
success/failure
error_message
```

This can be stored in a Data Table later.

## 19. Consider Telegram Buttons Later

### Goal

Later replace text confirmations with Telegram inline buttons.

Examples:

```text
Yes, delete
No, cancel
Delete all
Choose 1
Choose 2
```

### Current Decision

Do not add buttons now.

Plain text is easier to build, debug, and test.

## 20. Final Cleanup Before Production

### Tasks

```text
Rename unclear nodes
Remove unused nodes
Remove pinned test data
Check all calendars are set to Learn
Check all Create Event nodes use America/Edmonton timezone
Check all Telegram Chat ID fields use stable sessionId source
Check all Data Table fields are Text/String
Check all Switch rules have correct output names
Check all IF nodes use correct field/value placement
```

## Current Next Action

Continue from the cancel branch.

Next node likely needed:

```text
Cancel Switch routing:
single
multiple
none
```

Then build:

```text
none path
single confirm path
multiple selection path
pending confirm delete path
pending selection delete path
```


## Latest Backlog Update
Completed now:

```text
Cancel Switch single path
Cancel single confirmation question
Data Table insert pending_action = confirm_cancel
Confirmed cancel IF before Normalize Cancel Request
Prepare Confirm Cancel Delete Code node
Google Calendar Delete Event using saved target_event_id
Data Table cleanup by exact row ID after confirmed cancel
Telegram deleted confirmation after cleanup
```

Current remaining cancel work:

```text
1. Cancel none path
2. Cancel multiple match path
3. select_cancel_target reply handling
4. reject delete handling for no/cancel/never mind
5. calendar error handling
```

Recommended next action:

```text
Build Cancel none path first because it is simple:
Cancel Switch: none
→ Telegram: I couldn’t find any event matching that request.
→ STOP
```

Then build:

```text
Cancel Switch: multiple
→ Delete old waiting rows
→ Insert pending_action = select_cancel_target
→ Telegram list options and ask number/all
→ STOP
```
