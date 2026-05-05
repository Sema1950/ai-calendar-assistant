# Current State

## Project

AI Scheduler — n8n Telegram Calendar Automation

## Overall Status

The automation is partially built and several main branches are already working.

The current workflow can:

- Receive Telegram messages.
- Normalize Telegram input into fields for AI.
- Check Data Table for pending state.
- Merge original Telegram data with pending state.
- Use one AI Agent as the scheduling-intent router.
- Enforce structured AI output through Structured Output Parser.
- Use a Code node after AI Agent for deterministic override logic.
- Route requests through the Main Switch.
- Handle clarification questions.
- Show schedule.
- Book events with conflict detection.
- Reschedule events, including single-event and multi-event replacement logic.
- Store and clear pending state for multi-turn flows.
- Start building the cancel branch.

## Main Entry Flow

The main entry flow is working.

Current structure:

```text
Telegram Trigger
→ Edit Fields
→ Data Table: Get row(s) / Conflict Memory
→ Merge
→ AI Agent
→ Code in JavaScript
→ Main Switch
```

### Telegram Trigger

Working.

Receives user messages from Telegram.

### Edit Fields

Working.

Creates the required fields for AI Agent and memory:

```text
chatInput
sessionId
```

`chatInput` is taken from Telegram message text.

`sessionId` is taken from Telegram chat ID.

Other input fields are kept where possible.

### Conflict Memory / Data Table Get row(s)

Working.

Looks up pending state in the Data Table:

```text
pending_calendar_actions
```

Used to detect unfinished multi-turn actions such as:

```text
replace_conflicts_decision
confirm_cancel
select_cancel_target
```

### Merge

Working.

Merges:

- original Telegram input
- pending Data Table row, if one exists

Important configuration:

```text
Mode: Combine
Combine By: Position
Include Any Unpaired Items: ON
```

This allows the workflow to continue even if no pending row exists.

### AI Agent

Working.

The AI Agent classifies user messages into structured scheduling intent.

It uses:

- User message
- Session ID
- Current datetime
- Pending state
- Simple Memory

The AI Agent does not execute calendar actions.

### Structured Output Parser

Working.

Schema is configured and currently supports these intents:

```text
booking
reschedule
cancel
show_schedule
availability
clarification
```

The schema includes:

```text
intent
pending_intent
confirmed
needs_clarification
clarification_question
title
event_search_text
original_datetime
start_datetime
end_datetime
new_start_datetime
new_end_datetime
range_start_datetime
range_end_datetime
email
timezone
memory_used
confidence
```

### Code in JavaScript after AI Agent

Working.

This node acts as a deterministic safety override.

Current purpose:

- Corrects AI output in specific conflict-resolution cases.
- Prevents the workflow from relying only on AI when pending state requires stricter logic.

Important logic:

If there is an active pending conflict decision and the user gives a new time instead of confirming replacement, the code can force the route back to booking.

The override is intentionally placed after AI Agent and before Main Switch.

### Main Switch

Working.

Routes based on:

```text
{{ $json.output.intent }}
```

Current branches:

```text
clarification
show_schedule
booking
reschedule
cancel
availability
fallback
```

There is also a clarification-priority route using:

```text
{{ $json.output.needs_clarification }}
```

This allows clarification to override normal intent routing.

## Clarification Branch

Working.

Logic:

```text
needs_clarification = true
→ Telegram: Send Message
```

Telegram message text:

```text
{{ $json.output.clarification_question }}
```

The workflow then stops and waits for the user's next Telegram message.

The next user message goes through the same entry flow again.

## Show Schedule Branch

Working.

Current structure:

```text
show_schedule
→ Google Calendar: Get Many
→ Code: format schedule
→ Telegram: Send Message
```

### Google Calendar Get Many

Working.

Uses AI output:

```text
After = {{ $json.output.range_start_datetime }}
Before = {{ $json.output.range_end_datetime }}
```

### Schedule Formatting Code

Working.

Formats returned calendar events into a readable Telegram message.

Current improvements already made:

- Uses the requested date in the message.
- Sorts events by start time.
- Handles empty schedules.
- Formats time more clearly.

Example Telegram output:

```text
Your schedule for April 24, 2026:

8:00 AM - 9:00 AM | Event name
9:30 AM - 10:00 AM | Event name
```

Important issue fixed:

Pinned old test data caused incorrect schedule output earlier. This was fixed by unpinning old node data.

## Booking Branch

Working.

Current structure:

```text
booking
→ Google Calendar: Get Many / Schedule check1
→ Merge1
→ Code: Text normalisation / conflict checker
→ IF overlap in schedule
```

### Booking No-Conflict Path

Working.

If no calendar events are found in the requested time:

```text
IF false
→ Google Calendar: Create event
→ Telegram: Send confirmation
```

Confirmation message uses readable date/time.

Example:

```text
Booked: appointment with doctor on April 25, 3:00 PM
```

### Booking Conflict Path

Working.

If events overlap with the requested time:

```text
IF true
→ Delete old pending rows for this session
→ Data Table: Insert row
→ Telegram: Send conflict question
→ STOP
```

Before inserting a new pending row, old waiting rows for the same session are deleted.

This prevents the workflow from reading stale pending state later.

### Booking Conflict Data Saved

The workflow saves a pending row into:

```text
pending_calendar_actions
```

Saved fields include:

```text
sessionId
pending_action = replace_conflicts_decision
title
start_datetime
end_datetime
conflicting_event_ids
status = waiting_for_user
created_at
```

`conflicting_event_ids` is stored as stringified JSON.

Example:

```json
["id1","id2","id3"]
```

### Booking Conflict Telegram Question

Working.

The workflow sends a message like:

```text
You already have events at that time:

1) 5:00 PM - 6:00 PM | Event name
2) 6:00 PM - 7:00 PM | Event name

Do you want me to replace those events with this new one, or choose another time?
```

## Reschedule Branch

Working.

The reschedule branch is the most advanced branch currently built.

It supports:

- Simple single-event reschedule.
- Multiple-event replacement after user confirmation.
- None-found path.
- Pending-state continuation after user confirms replacement.
- Shared delete/create/confirmation logic for confirmed multi-event replacement.

## Reschedule Entry Logic

Current structure:

```text
reschedule
→ IF confirmed pending conflict replacement?
```

The IF checks whether the user confirmed a pending conflict action.

Current working simplified condition:

```text
{{ $json.output.confirmed }}
is equal to
true
```

This routes confirmed pending replacement to Code1.

Normal reschedule requests go to Schedule Check Reschedule.

## Normal Reschedule Path

Working.

Current structure:

```text
FALSE path
→ Schedule Check Reschedule
→ Code: count found events
→ Switch1
```

### Schedule Check Reschedule

Working after bug fix.

Uses correct search fields:

```text
After = {{ $json.output.original_datetime || $json.output.new_start_datetime }}
Before = {{ $json.output.new_end_datetime }}
```

This fixed the issue where null original datetime caused Google Calendar to return too many events.

### Code After Schedule Check Reschedule

Working.

Counts matching events and outputs:

```text
match_count
match_status
event_ids_to_delete
event_id_to_delete
found_events
conflict_message
```

Possible `match_status` values:

```text
single
multiple
none
```

The code also sorts events by start time and formats event lines.

### Switch1

Working.

Routes by:

```text
{{ $json.match_status }}
```

Outputs:

```text
single
multiple
none
```

## Reschedule Single Path

Working.

Logic:

```text
single
→ Google Calendar: Delete event
→ Google Calendar: Create event
→ Telegram: Send confirmation
```

The old event is deleted.

The new event is created with:

```text
title
new_start_datetime
new_end_datetime
```

Confirmation is sent to Telegram.

## Reschedule None Path

Working.

Logic:

```text
none
→ Telegram: Send Message
```

Hardcoded message:

```text
I couldn’t find that event. Which event do you want to reschedule?
```

No Data Table is used for this path currently.

The user is expected to send a clearer new request.

## Reschedule Multiple Path

Working.

When multiple events are found in the requested time range:

```text
multiple
→ Delete old waiting rows
→ Data Table: Insert row
→ Telegram: Send confirmation question
→ STOP
```

This saves the pending decision and waits for the user.

### Multiple Path Data Table Insert

Working.

Saved fields:

```text
sessionId
pending_action = replace_conflicts_decision
title
start_datetime
end_datetime
conflicting_event_ids
status = waiting_for_user
created_at
```

Important fix:

`conflicting_event_ids` must use:

```text
{{ JSON.stringify($('Switch1').item.json.event_ids_to_delete) }}
```

This ensures all matching event IDs are saved.

### Multiple Path Telegram Message

Working.

Telegram asks the user whether to remove the found events and create the new one.

Telegram node should execute once to avoid sending multiple duplicate messages.

Important fix:

If the previous Data Table node outputs multiple items, the Telegram node can send multiple messages.

Immediate fix used:

```text
Telegram node settings → Execute Once = ON
```

## Confirmed Multiple Replacement Path

Working.

When user confirms replacement:

```text
Telegram Trigger
→ Edit Fields
→ Conflict Memory
→ Merge
→ AI Agent
→ Code in JavaScript
→ Main Switch: reschedule
→ IF confirmed replacement
→ Code1
→ Google Calendar: Delete event
→ Code: Prepare one create item
→ Google Calendar: Create event
→ Telegram: Send confirmation
→ Data Table: Delete row(s)
```

### Code1

Working.

Reads the saved pending row from Conflict Memory.

Expands saved `conflicting_event_ids` into one item per event ID.

This allows Google Calendar Delete to delete multiple events.

Expected behavior:

If there are 4 saved event IDs, Code1 outputs 4 items.

### Delete Event

Working.

Deletes each event using:

```text
{{ $json.event_id_to_delete }}
```

### Prepare One Create Item

Working.

This Code node runs once for all items after Delete.

It collapses multiple delete outputs back into one item.

This prevents the Create Event node from creating the new event multiple times.

### Create Event

Working.

Creates one new event after all old conflicting events are deleted.

### Telegram Confirmation

Working.

Uses created Google Calendar output fields for readable confirmation.

### Data Table Cleanup

Working.

After confirmation, pending row is deleted by exact Data Table row ID.

Uses:

```text
id = {{ $('Prepare one create item').item.json.pending_row_id }}
```

This is safer than deleting by sessionId.

## Data Table

Current table name:

```text
pending_calendar_actions
```

Current columns include:

```text
sessionId
pending_action
title
start_datetime
end_datetime
conflicting_event_ids
status
created_at
candidate_event_ids
target_event_id
pending_message_type
```

All columns are Text/String.

Important design decision:

`conflicting_event_ids` and `candidate_event_ids` are stored as stringified JSON arrays.

Example:

```json
["id1","id2","id3"]
```

## Cancel Branch

Started but not finished.

Current plan:

```text
cancel
→ Code: Normalize cancel request
→ Google Calendar: Get Many
→ Code: Filter cancel matches
→ Switch
```

Switch outputs planned:

```text
single
multiple
none
```

### Cancel Normalize Code

Created.

Purpose:

Normalize AI output into:

```text
cancel_search_text
cancel_start_datetime
sessionId
```

### Cancel Get Many

Created.

Searches Google Calendar using:

```text
cancel_start_datetime
```

or current day if no exact datetime exists.

### Cancel Filter Matches Code

Created.

Filters returned events by `cancel_search_text`, dedupes logically through event IDs, sorts by start time, and outputs:

```text
match_count
match_status
event_ids_to_delete
event_id_to_delete
found_events
cancel_options_message
```

### Cancel Switch

In progress.

Planned routing:

```text
single
multiple
none
```

### Cancel Pending Actions

Prompt already updated to recognize these pending actions:

```text
confirm_cancel
select_cancel_target
```

These are Data Table `pending_action` values.

They must not be added to `pending_intent`.

## Prompt Status

The AI Agent prompt was improved and now uses a tighter structure.

The latest prompt includes:

- Role
- Runtime context
- Source-of-truth priority
- Output schema explanation
- Hard rules
- Intent definitions
- Multi-intent handling
- Pending conflict decision
- Pending cancel decisions
- Date and time inference
- Edge cases
- Output behavior

Important prompt rule:

```text
Current user message > Pending state > Simple Memory
```

Pending state is authoritative only when `pending_action` is not null.

## Known Working Branches

Fully or mostly working:

```text
clarification
show_schedule
booking
reschedule
```

In progress:

```text
cancel
availability
fallback
```

## Current Priority

The next build step is to finish the cancel branch.

Cancel branch should support:

```text
none → Telegram: no event found
single → ask confirmation before delete
multiple → list options and ask which event to delete
confirm_cancel → delete selected event after user confirms
select_cancel_target → handle number/all reply
clear pending state after delete
```


## Latest Update — Cancel Single Confirmation Completed
The cancel branch is no longer only started. The **single-match cancel confirmation path is built and tested**.

Working cancel pieces now:

```text
Main Switch: cancel
→ IF Confirmed Cancel
```

FALSE path for a new cancel request:

```text
IF false
→ Code: Normalize Cancel Request
→ Google Calendar: Get Many
→ Code: Filter Cancel Matches
→ Cancel Switch
→ single
→ Data Table: Delete old waiting rows
→ Data Table: Insert row pending_action = confirm_cancel
→ Telegram: Confirm delete?
→ STOP
```

TRUE path for a confirmed pending cancel:

```text
IF true
→ Code: Prepare Confirm Cancel Delete
→ Google Calendar: Delete Event
→ Data Table: Delete pending row by exact row ID
→ Telegram: Deleted confirmation
→ STOP
```

Confirmed test result:

```text
Event: Evening walk
Time: 2026-05-04T20:00:00-06:00 to 2026-05-04T20:45:00-06:00
pending_action: confirm_cancel
User reply: yes
Result: event deleted from Google Calendar, pending row cleaned, Telegram confirmation sent
```

Important implementation notes:

```text
IF Confirmed Cancel must be placed before Normalize Cancel Request.
Confirmed cancel must use the saved target_event_id from Conflict Memory / Data Table.
Confirmed cancel must not search Google Calendar again.
```

Still incomplete:

```text
cancel none path
cancel multiple match path
select_cancel_target reply handling
negative reply / reject delete handling
calendar error handling
availability branch
fallback branch
```

## Latest Update — Pending Context Design Decision

- A new design decision was made to add a `pending_context` field before the AI Agent.
- The purpose is to make pending user replies easier for the AI Agent to classify.
- This is especially important for cancel confirmation replies such as "no", unclear replies, and multiple-choice cancel selection.
- Data Table remains the source of truth.
- AI Agent interprets the user reply.
- Code/IF nodes must still protect destructive actions such as delete and replace.
- Next practical build step: add a Code/Edit Fields node before the AI Agent to generate `pending_context` when `pending_action` exists.
