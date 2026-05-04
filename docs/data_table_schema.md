# Data Table Schema

## Table Name

```text
pending_calendar_actions
```

## Purpose

This Data Table stores unfinished multi-turn workflow state.

It is the source of truth when the workflow asks the user a question and waits for a Telegram reply.

Examples:

- Booking conflict confirmation
- Reschedule multiple-event confirmation
- Cancel confirmation
- Cancel target selection

## Why Data Table Is Needed

AI memory is not reliable enough for workflow state.

The workflow needs structured state across separate executions.

Data Table allows the system to remember:

```text
what question was asked
which event IDs are involved
what new event should be created
which user/session the pending action belongs to
```

## Columns

All custom columns should be Text/String.

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

## Column Details

### sessionId

Telegram chat ID.

Used to find pending actions for the current user/chat.

Example:

```text
123456789
```

### pending_action

Workflow sub-state.

Current or planned values:

```text
replace_conflicts_decision
confirm_cancel
select_cancel_target
```

Important:

These values are not AI `intent` values and must not be added to `pending_intent`.

### title

Saved new event title.

Used for booking/reschedule replacement.

Example:

```text
Relax
```

### start_datetime

Saved start time for a pending event.

Usually used for:

```text
booking conflict replacement
reschedule multiple replacement
```

Example:

```text
2026-04-24T17:00:00-06:00
```

### end_datetime

Saved end time for a pending event.

Example:

```text
2026-04-24T23:00:00-06:00
```

### conflicting_event_ids

Stringified JSON array of event IDs to delete.

Used for:

```text
replace_conflicts_decision
```

Example:

```json
["id1","id2","id3"]
```

### status

Pending row status.

Common value:

```text
waiting_for_user
```

### created_at

ISO timestamp when the pending row was created.

Use Edmonton timezone if possible.

Example:

```text
2026-04-24T18:12:30-06:00
```

### candidate_event_ids

Stringified JSON array of candidate event IDs for cancel multiple-choice flow.

Used for:

```text
select_cancel_target
```

Example:

```json
["id1","id2","id3"]
```

### target_event_id

Single event ID selected or waiting for confirmation.

Used for:

```text
confirm_cancel
```

Example:

```text
abc123eventid
```

### pending_message_type

Optional helper label.

Can be used to record what kind of Telegram message was sent.

Examples:

```text
confirm_delete_single
select_delete_target
confirm_replace_events
```

## Pending Action Types

### replace_conflicts_decision

Used when booking or reschedule found conflicting events and needs user confirmation before deleting/replacing.

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

### confirm_cancel

Used when one cancel target was found and user must confirm deletion.

Saved fields:

```text
sessionId
pending_action = confirm_cancel
target_event_id
title
start_datetime
end_datetime
status = waiting_for_user
created_at
```

### select_cancel_target

Used when multiple cancel candidates were found and user must choose one or all.

Saved fields:

```text
sessionId
pending_action = select_cancel_target
candidate_event_ids
status = waiting_for_user
created_at
```

Optional:

```text
pending_message_type
```

## Lifecycle Pattern

### Create Pending State

```text
Delete old waiting rows for this session
→ Insert new pending row
→ Send Telegram question
→ STOP
```

### Resume Pending State

```text
Telegram Trigger
→ Edit Fields
→ Data Table: Get row(s)
→ Merge
→ AI Agent
→ Switch
→ Confirmed branch
```

### Complete Pending State

```text
Perform action
→ Send confirmation
→ Delete exact pending row by id
```

## Important Cleanup Rule

Before creating a new pending row, delete old `waiting_for_user` rows for the same `sessionId`.

This prevents stale pending rows from being reused.

## Preferred Delete After Completion

After confirmed completion, delete by exact row ID:

```text
id = pending_row_id
```

This is safer than deleting by sessionId.

## Type Decision

All columns are Text/String.

Reason:

- Avoid n8n type mismatch problems.
- Telegram IDs, event IDs, ISO dates, and JSON arrays work safely as strings.
- Code nodes can parse JSON strings when needed.

## Example Row: Replace Conflicts

```json
{
  "sessionId": "123456789",
  "pending_action": "replace_conflicts_decision",
  "title": "Relax",
  "start_datetime": "2026-04-24T17:00:00-06:00",
  "end_datetime": "2026-04-24T23:00:00-06:00",
  "conflicting_event_ids": "[\"id1\",\"id2\",\"id3\"]",
  "status": "waiting_for_user",
  "created_at": "2026-04-24T16:35:00-06:00"
}
```

## Example Row: Confirm Cancel

```json
{
  "sessionId": "123456789",
  "pending_action": "confirm_cancel",
  "target_event_id": "eventid123",
  "title": "Dinner",
  "start_datetime": "2026-04-24T18:00:00-06:00",
  "end_datetime": "2026-04-24T19:00:00-06:00",
  "status": "waiting_for_user",
  "created_at": "2026-04-24T16:40:00-06:00"
}
```

## Example Row: Select Cancel Target

```json
{
  "sessionId": "123456789",
  "pending_action": "select_cancel_target",
  "candidate_event_ids": "[\"id1\",\"id2\",\"id3\"]",
  "status": "waiting_for_user",
  "created_at": "2026-04-24T16:45:00-06:00"
}
```


## Latest Usage Update — confirm_cancel Tested
The `confirm_cancel` pending action is now implemented and tested.

Working saved fields:

```text
sessionId
pending_action = confirm_cancel
title
start_datetime
end_datetime
target_event_id
status = waiting_for_user
created_at
pending_message_type = confirm_delete_single
```

Example real tested row values:

```json
{
  "sessionId": "994586134",
  "pending_action": "confirm_cancel",
  "title": "Evening walk",
  "start_datetime": "2026-05-04T20:00:00-06:00",
  "end_datetime": "2026-05-04T20:45:00-06:00",
  "target_event_id": "_dlnmsc9i81nn0t0_20260505T020000Z",
  "status": "waiting_for_user",
  "pending_message_type": "confirm_delete_single"
}
```

After confirmation, the workflow deletes the pending row by exact Data Table row ID:

```text
id = pending_row_id
```

Do not delete by `sessionId` after completion unless intentionally clearing all old waiting rows before creating new pending state.
