# Decisions

## Project

AI Scheduler — n8n Telegram Calendar Automation

This document records important design decisions made during the project and the reasoning behind them.

## 1. Use One Main AI Agent, Not Multiple Agents

### Decision

Use one main AI Agent as the scheduling-intent router.

### Reason

The AI Agent is responsible only for:

```text
understanding user language
classifying intent
extracting structured scheduling fields
```

Adding multiple AI agents would increase complexity, latency, and debugging difficulty.

A multi-agent design could create confusion such as:

```text
Agent 1 misunderstands intent
Agent 2 overrides incorrectly
Agent 3 misreads pending state
Agent 4 formats the wrong response
```

### Final Direction

Use:

```text
One AI Agent
+ Structured Output Parser
+ Code nodes
+ Switch/IF nodes
+ Data Table state
```

The AI Agent suggests structure. n8n logic makes the final deterministic decisions.

## 2. AI Agent Must Not Execute Calendar Actions

### Decision

The AI Agent must not create, delete, update, search, or check calendar events.

### Reason

Calendar actions are high-impact operations. They must be handled by deterministic n8n nodes.

The AI Agent is probabilistic and may misinterpret requests.

### Final Direction

AI Agent only outputs structured data.

Google Calendar nodes perform:

```text
Get Many
Create event
Delete event
```

This keeps the workflow safer and easier to debug.

## 3. Use Structured Output Parser

### Decision

Use a Structured Output Parser with a fixed JSON schema.

### Reason

AI output must be predictable for Switch nodes and Code nodes.

Without the parser, AI could return:

```text
extra text
broken JSON
wrong casing
unexpected intent labels
missing fields
```

### Final Direction

The parser enforces the expected output fields:

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

## 4. Do Not Rely on AI Memory for Workflow State

### Decision

Do not rely only on Simple Memory for multi-turn state.

Use n8n Data Table for real workflow state.

### Reason

Simple Memory is useful for conversational context, but it is not reliable enough for workflow state.

Problems with AI memory:

```text
may forget context
may ignore context
may merge old and new context incorrectly
not structured
not deterministic
```

### Final Direction

Use Data Table as the source of truth for pending actions.

Simple Memory is secondary context only.

## 5. Use Data Table for Pending Multi-Turn Actions

### Decision

Use Data Table named:

```text
pending_calendar_actions
```

### Reason

Some actions require multiple turns:

```text
user asks to book/reschedule
workflow finds conflicts
workflow asks confirmation
user replies yes/no/new time
workflow continues
```

This requires persistent state across executions.

### Final Direction

Store pending actions in Data Table with:

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

All fields are Text/String for simplicity and consistency.

## 6. Store Event ID Arrays as Stringified JSON

### Decision

Store multiple event IDs as stringified JSON in one text field.

Examples:

```text
conflicting_event_ids
candidate_event_ids
```

Example value:

```json
["id1","id2","id3"]
```

### Reason

The number of conflicting or candidate events can vary.

Stringified JSON is easier than creating multiple columns like:

```text
event_id_1
event_id_2
event_id_3
```

### Final Direction

Use one Text/String column and parse it inside Code nodes.

## 7. Clean Old Pending Rows Before Saving New Pending State

### Decision

Before inserting a new pending row, delete old `waiting_for_user` rows for the same session.

### Reason

Old pending rows can cause the workflow to read stale state.

This happened during testing: the workflow read an older pending row and tried to delete only one old event.

### Final Direction

Before saving new pending state:

```text
Delete old waiting rows for sessionId
→ Insert new pending row
→ Send question
```

This applies especially to booking/reschedule conflict confirmation.

## 8. Delete Pending Row by Exact Row ID After Completion

### Decision

After a confirmed pending action completes, delete the exact pending row by Data Table row ID.

### Reason

Deleting by `sessionId` is less safe because the same user could have another pending action later.

### Final Direction

Use:

```text
id = pending_row_id
```

This deletes only the exact row used for the current confirmed action.

## 9. Main Source-of-Truth Priority

### Decision

Use this priority order:

```text
1. Current user message
2. Pending state from Data Table
3. Simple Memory
```

### Reason

The current message must remain the strongest signal.

Pending state is authoritative only for unfinished multi-turn flows.

Simple Memory is context only.

### Final Direction

Prompt explicitly states:

```text
If pending_action is null, ignore Pending state entirely.
```

## 10. Use `needs_clarification` as Clarification Routing Flag

### Decision

Route clarification using:

```text
needs_clarification = true
```

rather than confidence.

### Reason

Confidence is useful for logging, but it should not drive core workflow decisions.

The AI must explicitly decide whether clarification is needed.

### Final Direction

Clarification priority route checks:

```text
{{ $json.output.needs_clarification }}
equals true
```

## 11. Confidence Is Logging Only

### Decision

Do not route based on confidence.

### Reason

Confidence is not reliable enough for branch control.

It is useful for debugging AI behavior later.

### Final Direction

Prompt defines confidence like this:

```text
0.9 to 1.0 = clear request
0.7 to 0.89 = mostly clear
below 0.7 = clarification needed
```

But Switch routes do not use confidence.

## 12. Use Deterministic Code Node After AI Agent

### Decision

Place a Code node after AI Agent and before Main Switch.

### Reason

AI sometimes makes incorrect routing decisions in edge cases.

Example:

When pending conflict exists and user gives a new time, AI may still output `reschedule`.

The Code node can override safely.

### Final Direction

Flow:

```text
AI Agent
→ Code in JavaScript
→ Main Switch
```

This gives the workflow a deterministic correction layer.

## 13. Keep Pending Sub-States Out of `pending_intent`

### Decision

Do not put workflow sub-states into AI output field `pending_intent`.

### Reason

`pending_intent` is restricted by schema to main intent values:

```text
booking
reschedule
cancel
show_schedule
availability
clarification
null
```

Workflow sub-states such as:

```text
replace_conflicts_decision
confirm_cancel
select_cancel_target
```

belong only in Data Table field:

```text
pending_action
```

### Final Direction

Prompt explicitly says:

```text
Never put workflow sub-states into pending_intent.
```

## 14. Booking Uses `start_datetime` and `end_datetime`

### Decision

Booking uses:

```text
start_datetime
end_datetime
```

### Reason

This separates booking fields from reschedule fields.

### Final Direction

For booking:

```text
title
start_datetime
end_datetime
```

## 15. Reschedule Uses `new_start_datetime` and `new_end_datetime`

### Decision

Reschedule uses:

```text
new_start_datetime
new_end_datetime
```

for the new event.

### Reason

Reschedule has two time concepts:

```text
old event time
new event time
```

So we use:

```text
original_datetime = old event start time
new_start_datetime = new event start time
new_end_datetime = new event end time
```

### Final Direction

For reschedule:

```text
event_search_text
original_datetime
title
new_start_datetime
new_end_datetime
```

## 16. Reschedule Means Delete + Create, Not Update

### Decision

Do not use Google Calendar Update for reschedule.

Use:

```text
Delete old event
→ Create new event
```

### Reason

Update was considered less reliable and harder to control.

Delete + Create is clearer and more predictable in this workflow.

### Final Direction

All reschedule operations use delete/create logic.

## 17. Reschedule Is One Intent Even Though It Deletes and Creates

### Decision

Do not treat reschedule as multi-intent.

### Reason

Reschedule naturally includes:

```text
delete old event
create new event
```

But for the user, it is one scheduling action.

### Final Direction

Prompt states:

```text
Reschedule is ONE intent even though it deletes + creates.
```

## 18. Separate Cancel from Reschedule

### Decision

Cancel must be its own intent.

### Reason

Earlier prompt wording incorrectly treated remove/delete as reschedule.

That caused messages like:

```text
delete block between 17:00 and 18:00
```

to route to reschedule.

### Final Direction

Cancel means:

```text
delete/remove/cancel an event without creating a new one
```

Reschedule means:

```text
change/replace/clear/overwrite existing events and create a new one
```

## 19. Ask Confirmation Before Destructive Cancel

### Decision

Cancel should not immediately delete events.

### Reason

Deleting calendar events is destructive.

If a single event is found, the workflow should still ask for confirmation before deleting.

### Final Direction

Cancel planned flow:

```text
single match
→ ask confirm delete
→ save confirm_cancel pending row
→ user confirms
→ delete
→ send confirmation
→ clear pending row
```

## 20. For Multiple Cancel Matches, Ask User to Choose

### Decision

If multiple possible cancel targets are found, list them and ask the user to choose.

### Reason

The workflow should not guess which event to delete.

### Final Direction

Cancel multiple flow:

```text
multiple matches
→ list events
→ ask user to reply with number or all
→ save select_cancel_target pending row
→ wait for reply
```

## 21. Booking Conflict Requires Confirmation

### Decision

If a new booking overlaps existing events, do not book immediately.

### Reason

The system should not silently overwrite or delete existing events.

### Final Direction

Booking conflict flow:

```text
find overlapping events
→ save pending row
→ ask user
→ wait for confirmation
```

## 22. Multiple Reschedule Conflicts Require Confirmation

### Decision

If reschedule/replace time range finds multiple events, ask before deleting them.

### Reason

Deleting multiple events without confirmation is unsafe.

### Final Direction

Reschedule multiple flow:

```text
multiple
→ save pending row
→ ask confirmation
→ confirmed
→ delete saved event IDs
→ create new event
```

## 23. Single Reschedule Can Proceed Without Extra Confirmation

### Decision

If exactly one matching event is found for reschedule, proceed directly.

### Reason

The user already asked to reschedule and the workflow found exactly one matching event.

### Final Direction

Single path:

```text
single
→ delete old event
→ create new event
→ send confirmation
```

## 24. None Found Should Ask Clarification, Not Save State

### Decision

If no event is found for reschedule, send a simple clarification and stop.

### Reason

No Data Table is needed for this simple failure case.

### Final Direction

None path:

```text
none
→ Telegram: I couldn’t find that event. Which event do you want to reschedule?
→ STOP
```

## 25. Use Merge Nodes Carefully

### Decision

Use Merge only when required to preserve data from two branches.

### Reason

Incorrect Merge wiring caused false conflicts and duplicate branch execution.

Known issue fixed:

```text
Booking output and Schedule Check output were both connected incorrectly to the same Merge input.
```

Correct pattern:

```text
original request data → Merge input 1
calendar search results → Merge input 2
```

## 26. Filter Only Real Google Calendar Events in Code

### Decision

Code nodes must filter real Google Calendar event items.

### Reason

Merge nodes can pass non-calendar data, and empty output items can appear.

Without filtering, Code nodes may treat booking data or empty `{}` as a conflict.

### Final Direction

Use checks like:

```javascript
event.id && event.start && event.end
```

Only those count as real calendar events.

## 27. Sort Calendar Events by Start Time

### Decision

Code nodes should sort calendar events by start time before formatting or selecting.

### Reason

Google Calendar Get Many does not always return events in the expected order.

### Final Direction

Sort by:

```javascript
event.start.dateTime || event.start.date
```

## 28. Collapse Multiple Delete Items Before Creating New Event

### Decision

After deleting multiple events, collapse the output back to one item before Create Event.

### Reason

If Delete outputs 4 items and Create Event runs directly after it, the workflow may create the new event 4 times.

### Final Direction

Use a Code node:

```text
Prepare one create item
```

This converts multiple delete results into one item for Create Event.

## 29. Telegram Confirmation Should Use Created Event Output

### Decision

After Create Event, confirmation message should use the Google Calendar output.

### Reason

Previous fields may be gone after Create Event.

### Final Direction

Use created event fields:

```text
summary
start.dateTime
```

and format with:

```text
DateTime.fromISO(...).setZone('America/Edmonton')
```

## 30. Keep Node Count Reasonable

### Decision

Avoid unnecessary extra agents and unnecessary duplicated workflow branches.

### Reason

More nodes make debugging harder.

### Final Direction

Reuse shared endings where possible.

Example:

Single reschedule and confirmed multi-event replacement both reuse:

```text
delete event(s)
→ create event
→ send confirmation
```

with preparation Code nodes as needed.

## 31. Build Branches Incrementally

### Decision

Build and test one branch at a time.

### Reason

Large workflows become difficult to debug if too many parts are added at once.

### Final Direction

Order followed:

```text
entry flow
clarification
show_schedule
booking
reschedule
cancel
availability
fallback
```

Current next priority is finishing cancel.

## 32. Use Simple Telegram Text Before Buttons

### Decision

Use plain text replies first, not Telegram buttons.

### Reason

Plain text is easier to debug and works with AI intent parsing.

Buttons can be added later.

### Final Direction

Example confirmation:

```text
Do you want me to remove these events and create the new one?
```

## 33. All Data Table Columns Are Text/String

### Decision

Use Text/String type for all custom Data Table columns.

### Reason

This avoids type mismatch issues in n8n.

Dates, IDs, JSON arrays, and statuses are all handled consistently.

### Final Direction

Use strings for:

```text
sessionId
datetime values
event IDs
pending actions
JSON arrays
status
```

## 34. Use Exact Field Placement in Node Instructions

### Decision

When documenting or configuring nodes, specify:

```text
First field / value 1
Condition
Second field / value 2
Output Name
```

### Reason

This avoids confusion in n8n Switch and IF node setup.

### Final Direction

Future setup instructions should always be precise about exact field placement.

## 35. One Step at a Time During Build

### Decision

For troubleshooting/building, give one step at a time unless the user asks for a full list.

### Reason

The workflow is complex. Multi-step instructions create confusion when errors happen mid-way.

### Final Direction

Default style:

```text
one node or one configuration change
wait for confirmation/screenshot
continue
```


## Latest Decisions — Cancel Confirmation Handling
## 36. Confirmed Cancel Must Use Saved `target_event_id`

### Decision

When the user confirms a pending cancel request, the workflow must delete the saved `target_event_id` from Data Table.

It must not search Google Calendar again.

### Reason

The confirmation reply usually contains only:

```text
yes
```

Searching the calendar again at that point could match the wrong event or fail if the user reply lacks date/title context.

### Final Direction

Use:

```text
Conflict Memory.target_event_id
```

Then:

```text
Google Calendar Delete Event
→ Data Table Delete Row(s) by exact id
→ Telegram confirmation
```

## 37. Put IF Confirmed Cancel Before Normalize Cancel Request

### Decision

The cancel branch must split before Normalize Cancel Request:

```text
Main Switch: cancel
→ IF Confirmed Cancel
```

TRUE path handles confirmed pending cancel.

FALSE path handles a new cancel request.

### Reason

Normalize Cancel Request is only for first-time cancel requests. It is not appropriate for a confirmation reply.

## 38. Keep Verification Node Optional for Now

### Decision

A verification node can be added later after Data Table cleanup, but not now.

### Reason

The current priority is to keep the workflow simple and finish core cancel behavior first.

Future optional pattern:

```text
Data Table Delete Row(s)
→ Data Table Get Row(s) by id
→ IF row still exists
→ Telegram/internal error message
```
