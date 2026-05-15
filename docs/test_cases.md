# Test Cases

Note: `docs/Calendar_assist.json` is the current exported n8n workflow and implementation reference. It is maintained manually by the project owner; documentation should be aligned to it, but this JSON file should not be edited as part of documentation cleanup.

## Purpose

This file contains Telegram messages to test the AI Scheduler workflow.

Each test should verify:

```text
AI output
routing
calendar action
Telegram response
Data Table state
```

## Before Testing

Check:

```text
No old pinned data in n8n nodes
No stale pending rows unless intentionally testing pending state
Calendar selected = Learn
Timezone = America/Edmonton
```

## 1. Show Schedule — Today

### Message

```text
Show my schedule today.
```

### Expected AI

```text
intent = show_schedule
needs_clarification = false
range_start_datetime = today 00:00:00
range_end_datetime = today 23:59:59
```

### Expected Route

```text
show_schedule
→ Google Calendar Get Many
→ Format schedule
→ Telegram message
```

### Expected Result

Telegram shows sorted readable schedule for today.

## 2. Show Schedule — Specific Date

### Message

```text
Show my schedule next Friday.
```

### Expected AI

```text
intent = show_schedule
range_start_datetime = next Friday 00:00:00
range_end_datetime = next Friday 23:59:59
```

### Expected Result

Telegram shows events for the correct Friday.

## 3. Booking — No Conflict

### Message

```text
Book appointment tomorrow at 3 PM.
```

### Expected AI

```text
intent = booking
title = appointment
start_datetime = tomorrow 15:00
end_datetime = tomorrow 15:30
```

### Expected Route

```text
booking
→ Schedule check
→ no conflict
→ Create event
→ Telegram confirmation
```

### Expected Result

One event created.

One confirmation message sent.

## 4. Booking — Time Range

### Message

```text
Book doctor appointment tomorrow from 3 to 4 PM.
```

### Expected AI

```text
intent = booking
title = doctor appointment
start_datetime = tomorrow 15:00
end_datetime = tomorrow 16:00
```

### Expected Result

Default 30-minute duration must not be applied.

## 5. Booking — Conflict

### Message

```text
Book doctor appointment today from 5 to 6 PM.
```

### Setup

Calendar must already contain an event overlapping 5–6 PM.

### Expected Route

```text
booking
→ Schedule check
→ conflict found
→ delete old pending rows
→ insert pending row
→ Telegram question
→ STOP
```

### Expected Data Table

```text
pending_action = replace_conflicts_decision
title = doctor appointment
conflicting_event_ids contains all overlapping IDs
status = waiting_for_user
```

### Expected Telegram

```text
You already have events at that time...
Do you want me to replace those events with this new one, or choose another time?
```

## 6. Booking Conflict — Confirm Replacement

### Message

```text
yes, replace them
```

### Expected AI

```text
intent = reschedule
confirmed = true
memory_used = true
```

### Expected Route

```text
reschedule
→ confirmed pending replacement path
→ Code1
→ Delete all saved events
→ Prepare one create item
→ Create event once
→ Telegram confirmation
→ Delete pending row
```

### Expected Result

All saved conflicting events deleted.

New event created once.

Pending row deleted.

## 7. Booking Conflict — Try New Time

### Message

```text
Try 7 PM instead.
```

### Expected AI

```text
intent = booking
confirmed = false
memory_used = true
```

### Expected Route

```text
booking
→ Schedule check with new time
```

### Expected Result

The old pending replacement is abandoned logically.

The workflow tries a new booking time.

## 8. Reschedule — Single Event

### Message

```text
I have dinner today from 6 to 7. Reschedule it to walk time from 7 to 8.
```

### Expected AI

```text
intent = reschedule
event_search_text = dinner
title = walk time
original_datetime = today 18:00
new_start_datetime = today 19:00
new_end_datetime = today 20:00
```

### Expected Route

```text
reschedule
→ Schedule Check Reschedule
→ Code count events
→ Switch1 single
→ Delete old event
→ Create new event
→ Telegram confirmation
```

## 9. Reschedule — Multiple Events in Range

### Message

```text
I want to reschedule my events today. I need a new event between 5 and 11 PM named Relax.
```

### Expected AI

```text
intent = reschedule
title = Relax
new_start_datetime = today 17:00
new_end_datetime = today 23:00
```

### Expected Route

```text
reschedule
→ Schedule Check Reschedule
→ multiple
→ delete old pending rows
→ insert pending row
→ Telegram question
→ STOP
```

### Expected Data Table

```text
conflicting_event_ids contains all events between 5 and 11 PM
```

## 10. Reschedule Multiple — Confirm

### Message

```text
yes, remove them
```

### Expected Route

```text
confirmed pending replacement
→ Code1 outputs one item per saved ID
→ Delete event runs for each item
→ Prepare one create item
→ Create Relax once
→ Telegram confirmation
→ Delete pending row
```

## 11. Reschedule — None Found

### Message

```text
Reschedule fake meeting today from 2 to 3 to test from 4 to 5.
```

### Expected Route

```text
reschedule
→ Schedule Check Reschedule
→ none
→ Telegram message
```

### Expected Telegram

```text
I couldn’t find that event. Which event do you want to reschedule?
```

## 12. Cancel — Missing Date

### Message

```text
delete block between 17:00 and 18:00
```

### Expected AI

```text
intent = clarification
pending_intent = cancel
needs_clarification = true
clarification_question = What date should I delete this from?
```

## 13. Cancel — Single Match

### Message

```text
Cancel dinner today at 6 PM.
```

### Expected AI

```text
intent = cancel
event_search_text = dinner
original_datetime = today 18:00
```

### Expected Route

```text
cancel
→ Normalize cancel request
→ Google Calendar Get Many
→ Filter cancel matches
→ single
→ Ask confirm delete
→ Save pending confirm_cancel
→ STOP
```

## 14. Cancel — Confirm Single Delete

### Message

```text
yes
```

### Expected Route

```text
pending_action = confirm_cancel
→ intent = cancel
→ confirmed = true
→ Delete target_event_id
→ Telegram deleted confirmation
→ Delete pending row
```

## 15. Cancel — Multiple Matches

### Message

```text
Cancel my events today between 5 and 9 PM.
```

### Expected Route

```text
cancel
→ Filter matches
→ multiple
→ list options
→ Save pending select_cancel_target
→ STOP
```

### Expected Telegram

```text
I found more than one matching event:

1) ...
2) ...

Which one should I delete? Reply with a number or say all.
```

## 16. Cancel Multiple — Select Number

### Message

```text
2
```

### Expected Route

```text
pending_action = select_cancel_target
→ resolve selected event ID
→ delete selected event
→ Telegram confirmation
→ delete pending row
```

## 17. Cancel Multiple — Select All

### Message

```text
all
```

### Expected Route

```text
pending_action = select_cancel_target
→ resolve all candidate IDs
→ delete all selected events
→ Telegram confirmation
→ delete pending row
```

## 18. Availability — Free

### Message

```text
Am I free tomorrow at 3 PM?
```

### Expected AI

```text
intent = availability
range_start_datetime = tomorrow 15:00
range_end_datetime = tomorrow 15:30
```

### Expected Result

Telegram says user is free if no events overlap.

## 19. Availability — Busy

### Message

```text
Am I free today from 5 to 6 PM?
```

### Expected Result

If events overlap, Telegram lists the busy events.

## 20. Multi-Intent

### Message

```text
Cancel my 3 PM and book dinner at 7.
```

### Expected AI

```text
intent = clarification
needs_clarification = true
```

### Expected Telegram

```text
Which action should I handle first?
```

## 21. Garbage / Off Topic

### Message

```text
hi
```

### Expected AI

```text
intent = clarification
clarification_question = How can I help with your schedule?
```

## 22. Past Booking

### Message

```text
Book meeting yesterday at 3 PM.
```

### Expected AI

```text
intent = clarification
needs_clarification = true
```

## 23. 24-Hour Time

### Message

```text
Book work block tomorrow from 14:00 to 16:00.
```

### Expected AI

```text
intent = booking
start_datetime = tomorrow 14:00
end_datetime = tomorrow 16:00
```

No AM/PM clarification needed.

## 24. Bare Hour Ambiguity

### Message

```text
Book meeting today at 5.
```

### Expected

Depending on current time:

- If 5 PM is still future and context is clear, AI may use 5 PM.
- If ambiguous or already past, AI should ask clarification.

## 25. Weekend

### Message

```text
Show my schedule this weekend.
```

### Expected

If today is Monday-Friday:

```text
Saturday 00:00:00 to Sunday 23:59:59
```

If today is Saturday or Sunday:

```text
current weekend
```


## Latest Tested Result — Cancel Single Confirm
## Cancel Single — Tested Working

### Initial Message

```text
Delete walk today at 8 PM.
```

### AI Output

```text
intent = cancel
confirmed = false
event_search_text = walk
original_datetime = 2026-05-04T20:00:00-06:00
```

### Filter Cancel Matches Output

```text
match_status = single
match_count = 1
event_id_to_delete = _dlnmsc9i81nn0t0_20260505T020000Z
cancel_options_message = 1) 8:00 PM - 8:45 PM | Evening walk
```

### Pending Row Saved

```text
pending_action = confirm_cancel
title = Evening walk
start_datetime = 2026-05-04T20:00:00-06:00
end_datetime = 2026-05-04T20:45:00-06:00
target_event_id = _dlnmsc9i81nn0t0_20260505T020000Z
status = waiting_for_user
```

### Confirmation Message Sent

```text
I found this event:

May 4, 8:00 PM - 8:45 PM | Evening walk

Do you want me to delete it?
```

### User Reply

```text
yes
```

### Confirm Reply AI Output

```text
intent = cancel
pending_intent = cancel
confirmed = true
memory_used = true
```

### Confirmed Delete Route

```text
Main Switch: cancel
→ IF Confirmed Cancel TRUE
→ Prepare Confirm Cancel Delete
→ Google Calendar Delete Event
→ Data Table Delete Row(s)
→ Telegram deleted confirmation
```

### Final Result

```text
Google Calendar event deleted successfully.
Pending Data Table row deleted.
Telegram confirmation sent.
```
