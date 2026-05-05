# Architecture

## Project

AI Scheduler — n8n Telegram Calendar Automation

## Architecture Summary

This project uses n8n as the workflow engine, Telegram as the user interface, Google Calendar as the calendar backend, an AI Agent as the language router, and n8n Data Table as persistent workflow state.

The architecture is intentionally hybrid:

```text
AI Agent = language understanding and structured extraction
n8n nodes = deterministic execution, routing, safety, state, and calendar actions
```

The AI Agent does not directly create, delete, update, or search calendar events. It only outputs structured scheduling data. All real actions are performed by n8n nodes.

## Core Architecture Pattern

The system follows this pattern:

```text
Telegram Trigger
→ Normalize Telegram input
→ Load pending state from Data Table
→ Merge message + pending state
→ AI Agent
→ Structured Output Parser
→ Code override / validation
→ Main Switch
→ Specialist workflow branch
```

The specialist branches currently include:

```text
clarification
show_schedule
booking
reschedule
cancel
availability
fallback
```

## Main Entry Flow

### 1. Telegram Trigger

Purpose:

Receives the user's Telegram message.

Typical input data includes:

```text
message.text
message.chat.id
message.from
message.date
```

The Telegram chat ID is used as the session identifier.

### 2. Edit Fields

Purpose:

Creates clean fields needed by the AI Agent and memory.

Required fields:

```text
chatInput = Telegram message text
sessionId = Telegram chat ID
```

Important setting:

```text
Include Other Input Fields = ON
```

This helps keep original Telegram data available later in the workflow.

### 3. Data Table: Get row(s) / Conflict Memory

Purpose:

Looks for an active pending action for the same Telegram session.

Data table:

```text
pending_calendar_actions
```

Typical lookup filters:

```text
sessionId equals current sessionId
status equals waiting_for_user
```

Limit:

```text
1
```

This node retrieves unfinished multi-turn actions, such as confirmation requests.

### 4. Merge

Purpose:

Combines the current Telegram message with pending state from the Data Table.

Important configuration:

```text
Mode: Combine
Combine By: Position
Include Any Unpaired Items: ON
```

This is necessary because if no pending row exists, the workflow must still continue.

### 5. AI Agent

Purpose:

Classifies the user's message and extracts structured scheduling data.

The AI Agent receives:

```text
chatInput
sessionId
current datetime
timezone
pending state
Simple Memory context
```

The AI Agent outputs structured fields only.

It does not execute actions.

### 6. Structured Output Parser

Purpose:

Forces the AI Agent output into a predictable schema.

Supported intents:

```text
booking
reschedule
cancel
show_schedule
availability
clarification
```

Core output fields:

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

### 7. Code in JavaScript

Purpose:

Applies deterministic corrections after AI output and before routing.

This node acts as a safety layer.

Example responsibility:

If there is an active pending conflict decision and the user gives a new date or time instead of confirming replacement, force route back to booking.

Pattern:

```text
AI Agent suggests
Code node validates/corrects
Switch routes
```

### 8. Main Switch

Purpose:

Routes the request based on AI output.

Main route field:

```text
{{ $json.output.intent }}
```

Routes:

```text
clarification
show_schedule
booking
reschedule
cancel
availability
fallback
```

There is also a priority route for clarification based on:

```text
{{ $json.output.needs_clarification }}
```

Clarification should override normal intent routing.

## Data Table Architecture

### Table Name

```text
pending_calendar_actions
```

### Purpose

Stores workflow state across multiple Telegram messages.

This is required because AI memory is not reliable enough for deterministic workflow state.

### Columns

All columns are Text/String.

Current columns:

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

### Stored JSON Arrays

Some fields store arrays as stringified JSON.

Example:

```json
["event_id_1","event_id_2","event_id_3"]
```

Fields that use this pattern:

```text
conflicting_event_ids
candidate_event_ids
```

### Current Pending Actions

Supported or planned pending actions:

```text
replace_conflicts_decision
confirm_cancel
select_cancel_target
```

Important:

These values exist only in Data Table `pending_action`.

They must not be placed into AI output field `pending_intent`.

### Pending Row Lifecycle

General lifecycle:

```text
Detect situation requiring confirmation
→ delete old waiting rows for same session
→ insert new pending row
→ send Telegram question
→ wait for user reply
→ retrieve pending row
→ execute confirmed action
→ delete pending row
```

## Pending Context Architecture

When a pending Data Table row exists, the workflow should build a clear `pending_context` field before the AI Agent.

Purpose:

Make the previous workflow question easy for the AI Agent to understand.

Raw Data Table fields remain the source of truth for Code nodes, but `pending_context` helps the AI interpret the latest user reply.

Example `pending_context`:

```text
The user was asked to confirm deleting this event:
Title: Dinner
Time: 8:00 PM - 8:45 PM
Pending action: confirm_cancel

The latest user reply is:
no

Classify the latest reply in relation to this pending action.
```

Pattern:

```text
Telegram Trigger
→ Edit Fields
→ Data Table: Get pending row
→ Merge
→ Code/Edit Fields: Build pending_context
→ AI Agent
→ Structured Output Parser
→ Code override / safety validation
→ Main Switch
```

Applies to:

```text
replace_conflicts_decision
confirm_cancel
select_cancel_target
```

Core rule:

```text
AI understands the reply.
Code protects destructive actions.
Data Table remains the source of truth.
```

## Clarification Branch

### Purpose

Ask the user for missing required information.

### Flow

```text
Main Switch / clarification priority
→ Telegram: Send Message
→ STOP
```

### Telegram Text

```text
{{ $json.output.clarification_question }}
```

### Behavior

The workflow stops after asking the clarification question.

The next Telegram reply starts a new execution.

Memory and/or Data Table state helps the AI understand whether the reply continues a previous request.

## Show Schedule Branch

### Purpose

Display the user's calendar events for a requested date or date range.

### Flow

```text
Main Switch: show_schedule
→ Google Calendar: Get Many
→ Code: format schedule
→ Telegram: Send Message
```

### Google Calendar Get Many

Uses AI output:

```text
After = {{ $json.output.range_start_datetime }}
Before = {{ $json.output.range_end_datetime }}
```

### Code: Format Schedule

Responsibilities:

```text
filter valid events
sort events by start time
format readable date
format readable event times
handle empty schedule
return one Telegram message
```

### Example Output

```text
Your schedule for April 24, 2026:

8:00 AM - 9:00 AM | Event name
9:30 AM - 10:00 AM | Event name
```

## Booking Branch

### Purpose

Create a new event if the requested time is free.

If the requested time overlaps existing events, ask before replacing anything.

### Flow

```text
Main Switch: booking
→ Google Calendar: Schedule check1 / Get Many
→ Merge1
→ Code: Text normalisation / conflict checker
→ IF overlap in schedule
```

### Google Calendar Schedule Check

Checks requested booking window.

Uses:

```text
After = {{ $json.output.start_datetime }}
Before = {{ $json.output.end_datetime }}
```

### Merge1

Purpose:

Preserves both original booking data and calendar search results.

Correct wiring:

```text
booking output → Merge1 input 1
Schedule check1 output → Merge1 input 2
```

### Conflict Checker Code

Responsibilities:

```text
ignore non-calendar items
detect real Google Calendar event conflicts
format conflict list
return hasConflict boolean
return conflictingEvents array
```

### IF Overlap in Schedule

Condition:

```text
{{ $json.hasConflict }}
equals true
```

### No-Conflict Path

```text
IF false
→ Google Calendar: Create event
→ Telegram: Send booking confirmation
```

Create event uses:

```text
title
start_datetime
end_datetime
```

### Conflict Path

```text
IF true
→ Data Table: Delete old waiting rows for this session
→ Data Table: Insert row
→ Telegram: Send conflict question
→ STOP
```

### Data Saved for Booking Conflict

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

### Conflict Question

The user receives a message similar to:

```text
You already have events at that time:

1) 5:00 PM - 6:00 PM | Event name
2) 6:00 PM - 7:00 PM | Event name

Do you want me to replace those events with this new one, or choose another time?
```

## Reschedule Branch

### Purpose

Handle changing existing event(s) and creating a new replacement event.

The reschedule branch supports:

```text
simple single-event reschedule
multiple-event replacement after confirmation
none-found path
confirmed pending conflict replacement
```

### Reschedule Entry Flow

```text
Main Switch: reschedule
→ IF confirmed pending replacement?
```

### Confirmed Pending Replacement IF

Purpose:

Detect whether the user is confirming a previously saved conflict replacement.

Current routing uses confirmed state.

Expected logic:

```text
if confirmed replacement
→ Code1
else
→ Schedule Check Reschedule
```

### Normal Reschedule Search

```text
FALSE path
→ Google Calendar: Schedule Check Reschedule
→ Code: count found events
→ Switch1
```

### Schedule Check Reschedule

Uses:

```text
After = {{ $json.output.original_datetime || $json.output.new_start_datetime }}
Before = {{ $json.output.new_end_datetime }}
```

This handles both:

```text
simple reschedule with original_datetime
replace-events-in-range reschedule with new_start_datetime
```

### Code: Count Found Events

Responsibilities:

```text
filter valid Google Calendar events
sort by start time
count matches
return match_status
return event IDs
format conflict message
```

Outputs:

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

### Switch1

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

### Purpose

One matching event found.

### Flow

```text
Switch1: single
→ Google Calendar: Delete event
→ Google Calendar: Create event
→ Telegram: Send confirmation
```

### Delete Event

Uses:

```text
{{ $json.event_id_to_delete }}
```

### Create Event

Uses:

```text
title
new_start_datetime
new_end_datetime
```

### Confirmation

Uses created Google Calendar event data.

Example:

```text
Rescheduled: walk time on April 24, 7:00 PM
```

## Reschedule None Path

### Purpose

No matching event found.

### Flow

```text
Switch1: none
→ Telegram: Send Message
→ STOP
```

Message:

```text
I couldn’t find that event. Which event do you want to reschedule?
```

No Data Table state is currently used for this path.

## Reschedule Multiple Path

### Purpose

Multiple events were found in the requested time range.

No deletion should happen until the user confirms.

### Flow

```text
Switch1: multiple
→ Data Table: Delete old waiting rows
→ Data Table: Insert row
→ Telegram: Send confirmation question
→ STOP
```

### Data Saved

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

Important:

`conflicting_event_ids` must contain all event IDs from:

```text
{{ JSON.stringify($('Switch1').item.json.event_ids_to_delete) }}
```

### Telegram Confirmation Question

The user receives a list of events and a confirmation question.

Example:

```text
I found more than one event in that time:

1) 5:00 PM - 6:00 PM | Event name
2) 6:00 PM - 7:00 PM | Event name

Do you want me to remove these events and create the new one?
```

## Confirmed Multiple Replacement Path

### Purpose

After user confirms deleting multiple events and creating the new event.

### Flow

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
→ Google Calendar: Delete event
→ Code: Prepare one create item
→ Google Calendar: Create event
→ Telegram: Send confirmation
→ Data Table: Delete row(s)
```

### Code1

Purpose:

Reads the saved pending row from Conflict Memory.

Expands `conflicting_event_ids` into one item per event ID.

Example:

If there are 4 saved event IDs, Code1 outputs 4 items.

Each item contains:

```text
event_id_to_delete
title
start_datetime
end_datetime
new_start_datetime
new_end_datetime
sessionId
pending_action
pending_row_id
```

### Delete Events

Google Calendar Delete runs once per item.

Uses:

```text
{{ $json.event_id_to_delete }}
```

### Prepare One Create Item

Purpose:

Collapse multiple delete outputs back into one item.

This prevents the Create Event node from creating the new event multiple times.

### Create Event

Creates the replacement event once.

### Telegram Confirmation

Sends the final result to the user.

### Data Table Cleanup

Deletes the exact pending row that was used.

Uses:

```text
id = {{ $('Prepare one create item').item.json.pending_row_id }}
```

## Cancel Branch

### Purpose

Safely delete calendar events.

The cancel branch is in progress.

Target behavior:

```text
cancel
→ Normalize cancel request
→ Google Calendar: Get Many
→ Code: Filter cancel matches
→ Switch
```

### Normalize Cancel Request

Purpose:

Prepare clean fields for calendar search.

Outputs:

```text
cancel_search_text
cancel_start_datetime
sessionId
```

### Google Calendar Get Many

Searches either:

```text
around cancel_start_datetime
```

or

```text
current day
```

if no exact datetime exists.

### Filter Cancel Matches

Purpose:

```text
filter valid Google Calendar events
match by cancel_search_text if provided
sort by start time
count matches
format candidate options
```

Outputs:

```text
match_count
match_status
event_ids_to_delete
event_id_to_delete
found_events
cancel_options_message
```

Possible statuses:

```text
single
multiple
none
```

### Planned Cancel Switch

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

### Planned Cancel None Path

```text
none
→ Telegram: Send Message
```

Example:

```text
No events found matching that request.
```

### Planned Cancel Single Path

Safer version:

```text
single
→ Telegram: Confirm delete?
→ Data Table: Insert pending row with pending_action = confirm_cancel
→ STOP
```

After user confirms:

```text
pending_action = confirm_cancel
→ Google Calendar: Delete event
→ Telegram: Deleted
→ Data Table: Delete row
```

### Planned Cancel Multiple Path

```text
multiple
→ Telegram: list matching events
→ Data Table: Insert pending row with pending_action = select_cancel_target
→ STOP
```

After user replies with number or all:

```text
pending_action = select_cancel_target
→ determine selected event IDs
→ delete selected event(s)
→ Telegram confirmation
→ Data Table cleanup
```

## Availability Branch

Not fully built yet.

Target behavior:

```text
availability
→ Google Calendar: Get Many / Availability check
→ Code: determine free or busy
→ Telegram: Send result
```

## Fallback Branch

Not fully built yet.

Target behavior:

```text
fallback
→ Telegram: Send generic error or clarification
```

Example:

```text
I couldn’t understand that scheduling request. Could you rephrase it?
```

## Node Responsibility Summary

### AI Agent

```text
understand user text
classify intent
extract structured fields
ask for missing info
```

### Structured Output Parser

```text
force predictable JSON structure
restrict intent values
ensure all fields exist
```

### Code Nodes

```text
normalize AI output
override unsafe AI decisions
filter calendar events
sort events
prepare event IDs
collapse multiple items
format messages
```

### Switch Nodes

```text
route by intent
route by clarification status
route by match_count / match_status
```

### IF Nodes

```text
handle boolean decisions
detect conflict
detect confirmed pending replacement
```

### Google Calendar Nodes

```text
Get Many
Create event
Delete event
```

### Telegram Nodes

```text
send clarification
send schedule
send conflict question
send confirmation
send cancel messages
```

### Data Table Nodes

```text
get pending state
insert pending action
delete old waiting rows
delete completed pending row
```

## Safety Model

The workflow protects against unsafe deletion/replacement by using:

```text
calendar search before create/delete
confirmation before multi-event deletion
pending state stored in Data Table
exact row cleanup by row ID
deterministic Code nodes before Switch routing
```

## Current Build Priority

The next branch to finish is:

```text
cancel
```

Cancel should support:

```text
single match confirmation
multiple match selection
pending confirm_cancel
pending select_cancel_target
safe deletion
pending row cleanup
```


## Latest Architecture Update — Confirmed Cancel Split
The cancel branch now uses a safety split before normal cancel search logic.

Updated cancel branch entry:

```text
Main Switch: cancel
→ IF Confirmed Cancel
```

Reason:

```text
A first-time cancel request must search Google Calendar.
A confirmed cancel reply must use the saved Data Table target_event_id.
```

TRUE path:

```text
pending_action = confirm_cancel
confirmed = true
→ Prepare Confirm Cancel Delete
→ Google Calendar Delete Event
→ Data Table cleanup by exact row ID
→ Telegram confirmation
```

FALSE path:

```text
new cancel request
→ Normalize Cancel Request
→ Google Calendar Get Many
→ Filter Cancel Matches
→ Cancel Switch
```

This prevents the confirmed reply `yes` from searching Google Calendar again and accidentally matching the wrong event.
