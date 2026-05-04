# Bugs and Fixes

## Project

AI Scheduler — n8n Telegram Calendar Automation

This document records problems found during the build and how they were fixed.

## 1. Telegram Trigger Did Not Provide AI Agent Fields Directly

### Problem

The Telegram Trigger did not provide the exact fields the AI Agent and memory expected.

The AI Agent needed:

```text
chatInput
sessionId
```

but Telegram Trigger output used Telegram-specific nested fields.

### Fix

Added an **Edit Fields** node after Telegram Trigger.

Created:

```text
chatInput = Telegram message text
sessionId = Telegram chat ID
```

This made AI Agent and memory input consistent.

## 2. AI Agent Returned `confirmed = true` for Normal Requests

### Problem

When the user sent a normal request such as:

```text
Send me my schedule.
```

AI Agent returned:

```json
"confirmed": true
```

This was wrong because the user was not confirming a previous action.

### Fix

Added strict confirmation handling to the AI Agent prompt:

```text
Set confirmed = true only if the current user message explicitly confirms a previously proposed action.
Otherwise set confirmed = false.
Never set confirmed = true for a first-time request.
```

Added a final enforcement rule to re-check confirmation before output.

## 3. Switch Routed to Fallback Even When AI Intent Was Correct

### Problem

AI Agent returned:

```json
"intent": "show_schedule"
```

but the Switch routed to fallback.

### Cause

Switch expression handling was not reading the output path correctly or was comparing values inconsistently.

### Fix

Used explicit path:

```text
{{ $json["output"]["intent"] }}
```

or:

```text
{{ $json.output.intent }}
```

and ensured it was in expression mode.

Also enabled:

```text
Convert types where required
```

where needed.

## 4. AI Intent Could Have Case or Spacing Issues

### Problem

There was concern that AI might output:

```text
Reschedule
RESCHEDULE
reschedule 
```

instead of exact enum values.

### Fix

Kept Structured Output Parser enum to enforce valid intent values.

Discussed optional normalization node, but current schema and parser are the primary protection.

Decision:

```text
Prompt + Schema + Parser = primary guard
Code normalization only if real failures appear
```

## 5. Clarification Needed to Override Intent Routing

### Problem

AI could output:

```json
intent = booking
needs_clarification = true
```

If routing only used `intent`, the workflow might continue to booking instead of asking the user a question.

### Fix

Added a clarification-priority rule before normal intent routing.

Rule:

```text
{{ $json.output.needs_clarification }}
equals true
```

This routes to Telegram clarification first.

## 6. Boolean Condition Failed in Switch

### Problem

`needs_clarification` was boolean, but the Switch was comparing it as text.

### Fix

Used one of these:

```text
{{ $json.output.needs_clarification.toString() }}
equals
true
```

or enabled:

```text
Convert types where required
```

Preferred solution:

```text
Convert types where required = ON
```

## 7. Telegram Chat ID Failed After Switch/Data Table Nodes

### Problem

Telegram Send Message node could not always access:

```text
{{ $json.message.chat.id }}
```

because the original Telegram fields were lost after some nodes.

### Fix

Used a stable session ID source.

Examples used:

```text
{{ $('Merge').item.json.sessionId }}
```

or, in some branches:

```text
{{ $('Telegram Trigger').item.json.message.chat.id }}
```

Final recommendation:

Use `sessionId` from normalized/merged data wherever possible.

## 8. Show Schedule Raw Calendar Output Was Too Messy

### Problem

Google Calendar Get Many returned large raw event objects.

Sending them directly to Telegram was unreadable.

### Fix

Added a Code node to format schedule output into readable text.

Example final message:

```text
Your schedule for April 24, 2026:

8:00 AM - 9:00 AM | Event name
9:30 AM - 10:00 AM | Event name
```

## 9. Show Schedule Used Wrong or Old Calendar Data

### Problem

Telegram showed schedule items that did not match the visible Google Calendar day.

### Cause

Pinned test data was still active in n8n.

### Fix

Unpinned old data in affected nodes.

Important lesson:

When test results look impossible, check for:

```text
This data is pinned for test executions
```

and unpin it.

## 10. Show Schedule Events Were Not Sorted

### Problem

Google Calendar events appeared in Telegram out of chronological order.

### Cause

Google Calendar Get Many did not always return events sorted by start time.

### Fix

Sorted events in Code node by:

```javascript
event.start?.dateTime || event.start?.date
```

## 11. Show Schedule Date Header Was Missing

### Problem

Schedule message said:

```text
Your schedule:
```

but did not show which date the schedule was for.

### Fix

Used `range_start_datetime` from AI output to format the date.

Example:

```text
Your schedule for April 24, 2026:
```

## 12. Google Calendar Get Many Returned Empty Output When No Events Found

### Problem

When checking availability or conflicts, Google Calendar Get Many returned no items when no events existed.

This could stop downstream nodes.

### Fix

Enabled:

```text
Always Output Data = ON
```

where needed.

Then Code nodes were updated to handle empty `{}` output correctly.

## 13. Empty `{}` Was Treated as a Conflict

### Problem

When Google Calendar returned empty output with Always Output Data enabled, downstream Code saw one item and treated it as a real event.

### Fix

Code nodes were updated to count only real Google Calendar events.

Pattern:

```javascript
event.id && event.start && event.end
```

Only items with these fields are treated as calendar events.

## 14. Merge Caused False Booking Conflicts

### Problem

In the booking branch, Merge added original booking data back into the stream.

The conflict checker treated this non-calendar item as a calendar event.

### Fix

Updated conflict checker code to filter only real calendar events using:

```javascript
event.id && event.start && event.end
```

This ignored booking data and empty items.

## 15. Booking Merge Was Wired Incorrectly

### Problem

Booking conflict path ran both true and false branches.

### Cause

The Merge node was wired incorrectly.

Both booking and schedule-check paths were connected in a way that produced duplicate or mixed items.

### Fix

Correct wiring:

```text
booking output → Merge input 1
Schedule check output → Merge input 2
Merge → conflict checker
```

After fixing wiring, the IF route worked correctly.

## 16. Booking Conflict Telegram Sent Multiple Messages

### Problem

Telegram sent multiple conflict messages.

### Cause

Previous node output multiple items, and Telegram executed once per item.

### Fix

For Telegram nodes after multi-item outputs, enabled:

```text
Execute Once = ON
```

Used only where a single message should be sent.

## 17. Booking Conflict Needed Pending State

### Problem

The system asked the user if they wanted to replace conflicting events, but did not save what the question was about.

The next user reply could not reliably continue the workflow.

### Fix

Added Data Table pending state.

Conflict flow became:

```text
Conflict true
→ Data Table: Insert row
→ Telegram: Send question
→ STOP
```

Saved fields include:

```text
sessionId
pending_action
title
start_datetime
end_datetime
conflicting_event_ids
status
created_at
```

## 18. AI Memory Alone Was Not Reliable for Conflict Continuation

### Problem

After conflict question, user replied:

```text
I want to replace it.
```

AI did not reliably know what “it” referred to.

### Fix

Used Data Table state as authoritative workflow memory.

AI prompt receives pending state and can classify replies correctly.

Data Table, not AI memory, became the source of truth for multi-turn actions.

## 19. AI Routed “Replace It” to Wrong Intent

### Problem

When user replied to a conflict question with:

```text
I want to replace it.
```

AI first returned `reschedule` in a confusing way, or previously tried to use unsupported `replace_conflicts`.

### Fix

Project decision:

Do not add a new public intent called `replace_conflicts`.

Use:

```text
intent = reschedule
```

for confirmed replacement.

Keep workflow sub-state in Data Table:

```text
pending_action = replace_conflicts_decision
```

## 20. `replace_conflicts_decision` Appeared in `pending_intent`

### Problem

AI tried to output:

```text
pending_intent = replace_conflicts_decision
```

Structured Output Parser rejected this because `pending_intent` only allows main intents.

### Fix

Prompt updated:

```text
Never put workflow sub-states like replace_conflicts_decision into pending_intent.
```

Workflow sub-states are stored only in Data Table `pending_action`.

## 21. AI Kept Sending New Time Replies to Reschedule

### Problem

After a conflict question, if user gave a new time like:

```text
next Friday at 8pm
```

AI still routed to `reschedule`.

### Fix

Added deterministic Code node after AI Agent.

Logic:

If pending conflict exists and user gives a new time, override intent to booking.

Later prompt was tightened to reduce this error, but Code node remains a safety layer.

## 22. Code Override Was Too Broad

### Problem

The Code override changed normal reschedule requests to booking just because the message had words like:

```text
today
tomorrow
6 PM
```

### Fix

Narrowed override condition so it only runs when:

```text
pending_action = replace_conflicts_decision
```

This prevented normal reschedule requests from being incorrectly routed to booking.

## 23. Data Table Get Row Returned Old Pending Row

### Problem

The workflow read an old pending row with stale event IDs.

As a result, Code1 deleted only one old event instead of all current conflicts.

### Fix

Before inserting a new pending row, delete old waiting rows for the same session.

Pattern:

```text
Delete old waiting rows for sessionId
→ Insert new pending row
→ Send question
```

## 24. Confirmed Multiple Replacement Deleted Only One Event

### Problem

After user confirmed replacing multiple events, only the first event was deleted.

### Cause

The workflow was going through normal reschedule path again, not using saved `conflicting_event_ids`.

### Fix

Added IF before Schedule Check Reschedule.

Confirmed pending replacement routes to Code1 instead of normal schedule search.

Code1 reads saved Data Table row and expands `conflicting_event_ids` into one item per event ID.

## 25. IF for Confirmed Pending Replacement Used Wrong Fields

### Problem

The IF node checked:

```text
$json.pending_action
```

but this field was missing in that execution item.

### Fix

Simplified the IF condition to route based on:

```text
{{ $json.output.confirmed }}
equals true
```

Then Code1 reads the actual pending row from Conflict Memory.

## 26. Code1 Output Was Empty

### Problem

Code1 tried to read:

```javascript
$json.conflicting_event_ids
```

but the input item only contained AI output fields.

### Fix

Changed Code1 to read from the Data Table node:

```javascript
const pending = $('Conflict Memory').item.json;
```

Then parsed:

```javascript
pending.conflicting_event_ids
```

## 27. Delete Multiple Events Could Create New Event Multiple Times

### Problem

When Code1 output 4 items, Delete Event ran 4 times.

If Create Event was connected directly after Delete Event, it could create the new event 4 times.

### Fix

Added Code node after Delete Event:

```text
Prepare one create item
```

This collapses multiple delete outputs back to one item before Create Event.

## 28. Telegram Confirmation After Create Event Had Invalid Time

### Problem

Telegram confirmation used:

```text
$json.new_start_datetime
```

after Create Event.

But after Google Calendar Create Event, that field was gone.

### Fix

Use Google Calendar Create output instead:

```text
$json.summary
$json.start.dateTime
```

Formatted with:

```text
DateTime.fromISO($json.start.dateTime).setZone('America/Edmonton')
```

## 29. Reschedule Schedule Check Returned Too Many Events

### Problem

Schedule Check Reschedule returned many events instead of only the target window.

### Cause

It used:

```text
After = original_datetime
```

but for Type B reschedule, `original_datetime` was null.

### Fix

Changed After field to:

```text
{{ $json.output.original_datetime || $json.output.new_start_datetime }}
```

Before field:

```text
{{ $json.output.new_end_datetime }}
```

## 30. Reschedule Code Event Order Was Wrong

### Problem

Multiple events were listed out of order.

### Fix

Sorted events in the Code node by start time.

## 31. Reschedule Prompt Put Old Title in `title`

### Problem

For reschedule request:

```text
I have dinner from 6 to 7. Reschedule it for walk time from 6 to 7.
```

AI put:

```text
title = dinner
```

But `title` should be the new event title.

### Fix

Prompt updated:

```text
event_search_text = old event to find/delete
title = new event title to create
```

This fixed reschedule field meaning.

## 32. Reschedule Example Was Semantically Weak

### Problem

Example used same old and new time:

```text
dinner from 6 to 7 → walk time from 6 to 7
```

This is closer to rename/replace than reschedule.

### Fix

Example changed to:

```text
dinner today from 6 to 7 → walk time from 7 to 8
```

## 33. Prompt Was Too Bloated and Repetitive

### Problem

Prompt contained repeated rules and scattered field definitions.

This made AI behavior less consistent.

### Fix

Prompt was rewritten with cleaner structure:

```text
Role
Runtime context
Source-of-truth priority
Output schema
Hard rules
Intent definitions
Multi-intent
Pending conflict decision
Pending cancel decisions
Date and time inference
Edge cases
Output behavior
```

## 34. `pending_intent` Was Undefined

### Problem

Prompt used `pending_intent` but did not clearly define it.

### Fix

Prompt now defines:

```text
pending_intent = original intent still waiting for clarification or confirmation
```

It does not store workflow sub-states.

## 35. `memory_used` and `confidence` Were Undefined

### Problem

Prompt had `memory_used` and `confidence`, but did not define how to set them.

### Fix

Prompt now defines:

```text
memory_used = true only if Pending state or memory affected the output
confidence = logging only, not used for routing
```

## 36. Delete Request Routed to Reschedule

### Problem

User message:

```text
delete block between 17:00 and 18:00
```

routed to `reschedule`.

### Cause

Prompt said remove/clear/delete could be reschedule.

### Fix

Prompt clarified:

```text
cancel = delete/remove/cancel without creating a new event
reschedule = change/replace/clear/overwrite and create a new event
```

Cancel branch should handle pure delete requests.

## 37. Delete Request Without Date Did Not Ask Date

### Problem

Message:

```text
delete block between 17:00 and 18:00
```

gave a time range but no date.

AI did not ask for date.

### Fix

Prompt cancel section updated:

```text
If only a time is given without a date, ask:
"What date should I delete this from?"
```

## 38. Google Calendar Event Timezone Showed America/New_York

### Problem

Some Google Calendar returned event objects had:

```text
timeZone = America/New_York
```

even though workflow uses America/Edmonton.

### Explanation

Google Calendar can store each event with its own event timezone.

The ISO `dateTime` value includes the offset, so formatting with:

```text
DateTime.fromISO(...).setZone("America/Edmonton")
```

displays correctly in Edmonton time.

### Fix / Prevention

When creating events, ensure Create Event nodes use:

```text
America/Edmonton
```

where timezone setting is available.

## 39. Stricter JSON Schema Was Considered but Rejected

### Problem

A stricter schema using:

```text
anyOf
format: date-time
format: email
maxLength
```

was proposed.

### Decision

Do not use it yet.

### Reason

n8n / LangChain parser may behave more reliably with simpler schema:

```json
"type": ["string", "null"]
```

Strict format validation could reject useful AI output unnecessarily.

### Fix

Keep the simpler schema for now.

## 40. Cancel Branch Needed New Data Table Columns

### Problem

Cancel confirmation and multi-choice selection need pending state.

### Fix

Added Data Table columns:

```text
candidate_event_ids
target_event_id
pending_message_type
```

All are Text/String.

## 41. Prompt Needed Pending Cancel Decisions

### Problem

AI would not understand replies to cancel confirmation unless prompt mentioned new pending actions.

### Fix

Added pending cancel decision block for:

```text
confirm_cancel
select_cancel_target
```

Important:

These are Data Table pending actions only.

They must not be added to `pending_intent`.

## 42. Node Instructions Were Too Broad

### Problem

Multi-step instructions caused confusion during n8n build.

### Fix

Project rule established:

Give one step at a time unless a full list is explicitly requested.

For Switch/IF setup, always specify:

```text
First field / value 1
Condition
Second field / value 2
Output Name
```

## 43. Current Known Incomplete Area

### Problem

Cancel branch is started but not finished.

### Current Completed Pieces

```text
Normalize cancel request
Google Calendar Get Many
Filter cancel matches
```

### Still Needed

```text
Cancel Switch
none path
single confirmation path
multiple selection path
confirm_cancel continuation
select_cancel_target continuation
delete action
confirmation message
pending row cleanup
```


## Latest Bugs and Fixes — Cancel Branch
## 44. Filter Cancel Matches Lost `cancel_search_text`

### Problem

After Google Calendar Get Many, the current `$json` was a calendar event object. It no longer contained:

```text
cancel_search_text
cancel_start_datetime
sessionId
```

As a result, this logic failed:

```javascript
const searchText = ($json.cancel_search_text || "").toLowerCase();
```

`searchText` became empty, so the filter returned all events in the searched time range.

### Fix

Read the source data from the previous Normalize Cancel Request node:

```javascript
const source = $('Normalize Cancel Request').first().json;
const searchText = (source.cancel_search_text || "").toLowerCase();
```

Then return source fields together with match results.

## 45. Delete Old Pending Rows Stopped Insert Row

### Problem

After `Data Table: Delete Row(s)`, the next Insert Row node did not execute when there were no rows to delete.

### Fix

Turn on:

```text
Settings → Always Output Data = ON
```

on the Delete Row(s) cleanup node.

## 46. Insert Row Expressions Turned Red After Cleanup Node

### Problem

After Delete Row(s), the current item no longer contained:

```text
found_events
sessionId
event_id_to_delete
```

So expressions like this failed:

```text
{{ $json.found_events[0].summary }}
```

### Fix

Reference the Filter Cancel Matches node directly:

```text
title = {{ $('Filter cancel matches').first().json.found_events[0].summary }}
start_datetime = {{ $('Filter cancel matches').first().json.found_events[0].start.dateTime }}
end_datetime = {{ $('Filter cancel matches').first().json.found_events[0].end.dateTime }}
sessionId = {{ $('Filter cancel matches').first().json.sessionId }}
target_event_id = {{ $('Filter cancel matches').first().json.event_id_to_delete }}
```

## 47. Confirmed Cancel Should Not Search Calendar Again

### Problem

When the user replied `yes`, the workflow routed to cancel and would have gone through Normalize Cancel Request and Calendar search again.

This is wrong because the pending row already stores the exact event ID to delete.

### Fix

Added an IF node immediately after Main Switch: cancel:

```text
IF Confirmed Cancel
{{ $json.output.confirmed }} is equal to true
```

TRUE path:

```text
Prepare Confirm Cancel Delete
→ Google Calendar Delete Event
→ Data Table cleanup
→ Telegram confirmation
```

FALSE path:

```text
Normalize Cancel Request
```

## 48. Confirmed Cancel Delete Path Built and Tested

### Result

The workflow successfully deleted:

```text
Evening walk
2026-05-04T20:00:00-06:00 to 2026-05-04T20:45:00-06:00
```

using saved:

```text
target_event_id
```

from Data Table pending state.

The pending row was then deleted by exact row ID.
