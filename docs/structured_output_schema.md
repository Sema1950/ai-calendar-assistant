# Structured Output Schema

## Purpose

This schema is used in the n8n Structured Output Parser connected to the AI Agent.

The goal is to force the AI Agent to output predictable fields that can be safely used by Switch, IF, Code, Google Calendar, Telegram, and Data Table nodes.

## Current Schema

```json
{
  "type": "object",
  "properties": {
    "intent": {
      "type": "string",
      "enum": [
        "booking",
        "reschedule",
        "cancel",
        "show_schedule",
        "availability",
        "clarification"
      ]
    },
    "pending_intent": {
      "type": ["string", "null"],
      "enum": [
        "booking",
        "reschedule",
        "cancel",
        "show_schedule",
        "availability",
        "clarification",
        null
      ]
    },
    "confirmed": {
      "type": "boolean"
    },
    "needs_clarification": {
      "type": "boolean"
    },
    "clarification_question": {
      "type": ["string", "null"]
    },
    "title": {
      "type": ["string", "null"]
    },
    "event_search_text": {
      "type": ["string", "null"]
    },
    "original_datetime": {
      "type": ["string", "null"]
    },
    "start_datetime": {
      "type": ["string", "null"]
    },
    "end_datetime": {
      "type": ["string", "null"]
    },
    "new_start_datetime": {
      "type": ["string", "null"]
    },
    "new_end_datetime": {
      "type": ["string", "null"]
    },
    "range_start_datetime": {
      "type": ["string", "null"]
    },
    "range_end_datetime": {
      "type": ["string", "null"]
    },
    "email": {
      "type": ["string", "null"]
    },
    "timezone": {
      "type": "string",
      "enum": ["America/Edmonton"]
    },
    "memory_used": {
      "type": "boolean"
    },
    "confidence": {
      "type": "number",
      "minimum": 0,
      "maximum": 1
    }
  },
  "required": [
    "intent",
    "pending_intent",
    "confirmed",
    "needs_clarification",
    "clarification_question",
    "title",
    "event_search_text",
    "original_datetime",
    "start_datetime",
    "end_datetime",
    "new_start_datetime",
    "new_end_datetime",
    "range_start_datetime",
    "range_end_datetime",
    "email",
    "timezone",
    "memory_used",
    "confidence"
  ],
  "additionalProperties": false
}
```

## Why This Schema Is Intentionally Simple

A stricter schema was considered, using:

```json
"format": "date-time"
"format": "email"
"anyOf"
"maxLength"
```

The decision was to avoid that for now.

Reason:

n8n / LangChain parser behavior is usually more stable with simpler schemas.

Strict `format: date-time` could reject valid AI output if formatting is slightly different.

Strict `format: email` could fail the whole parser on imperfect email input.

## Important Rules

Do not add workflow sub-states to this schema.

These are Data Table `pending_action` values only:

```text
replace_conflicts_decision
confirm_cancel
select_cancel_target
```

They must not be added to:

```text
intent
pending_intent
```

## Field Meaning

### intent

Main route for the Main Switch.

Allowed values:

```text
booking
reschedule
cancel
show_schedule
availability
clarification
```

### pending_intent

The main intent still waiting for clarification or confirmation.

Allowed values are the same as `intent`, plus `null`.

### confirmed

True only when the current message confirms a previous proposed action.

### needs_clarification

Primary flag for clarification routing.

Use this for clarification priority routing, not `confidence`.

### title

New event title.

Used mainly for:

```text
booking
reschedule
```

### event_search_text

Description of existing event to find.

Used mainly for:

```text
reschedule Type A
cancel
```

### original_datetime

Old/existing event start datetime.

Used mainly for:

```text
reschedule
cancel
```

### start_datetime / end_datetime

Booking start and end time.

Used only for:

```text
booking
```

### new_start_datetime / new_end_datetime

New event time for reschedule.

Used only for:

```text
reschedule
```

### range_start_datetime / range_end_datetime

Used for:

```text
show_schedule
availability
```

### memory_used

Logging/debug field.

Do not route on this.

### confidence

Logging/debug field.

Do not route on this.

## Routing Notes

Main Switch should route on:

```text
{{ $json.output.intent }}
```

Clarification priority should route on:

```text
{{ $json.output.needs_clarification }}
```

Do not route on:

```text
confidence
memory_used
```


## Latest Usage Note — Cancel Confirmation
No schema change was needed for the completed cancel confirmation path.

The existing schema already supports the confirmed cancel reply using:

```text
intent = cancel
pending_intent = cancel
confirmed = true
memory_used = true
```

Important reminder:

```text
confirm_cancel
select_cancel_target
replace_conflicts_decision
```

remain Data Table `pending_action` values only. They must not be added to `intent` or `pending_intent`.
