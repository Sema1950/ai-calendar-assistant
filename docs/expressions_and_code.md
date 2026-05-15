# Expressions and Code

Note: `docs/Calendar_assist.json` is the current exported n8n workflow and implementation reference. It is maintained manually by the project owner; documentation should be aligned to it, but this JSON file should not be edited as part of documentation cleanup.

## Purpose

This file stores important n8n expressions and Code node scripts used in the workflow.

## Common Expressions

### Session ID from Merge

```text
{{ $('Merge').item.json.sessionId }}
```

### Session ID from current item

```text
{{ $json.sessionId }}
```

### Telegram Chat ID from Telegram Trigger

```text
{{ $('Telegram Trigger').item.json.message.chat.id }}
```

### Current Edmonton Time

```text
{{ $now.setZone('America/Edmonton').toISO() }}
```

### Format ISO Date for Telegram

```text
{{ DateTime.fromISO($json.start.dateTime).setZone('America/Edmonton').toFormat('MMMM d, h:mm a') }}
```

### Format AI datetime

```text
{{ DateTime.fromISO($('Switch').item.json.output.start_datetime).setZone('America/Edmonton').toFormat('MMMM d, h:mm a') }}
```

## AI Override Code

Placement:

```text
AI Agent → Code in JavaScript → Main Switch
```

```javascript
const data = $json.output || {};

const message = ($json.chatInput || "").toLowerCase();

const hasNewTime =
  !!data.start_datetime ||
  !!data.new_start_datetime ||
  /\b(today|tomorrow|next|monday|tuesday|wednesday|thursday|friday|saturday|sunday|morning|afternoon|evening|\d{1,2}(:\d{2})?\s?(am|pm)?)\b/.test(message);

if (
  data.intent === "reschedule" &&
  $json.pending_action === "replace_conflicts_decision" &&
  hasNewTime
) {
  data.intent = "booking";
  data.confirmed = false;
  data.needs_clarification = false;
  data.clarification_question = null;
}

return [
  {
    json: {
      ...$json,
      output: data
    }
  }
];
```

## Build Pending Context

Placement:

```text
Merge -> Build Pending Context -> AI Agent
```

Field:

```text
pending_context
```

Expression:

```javascript
={{
$json.pending_action
  ? `The workflow is waiting for the user to respond to a pending calendar action.

Pending action: ${$json.pending_action}

Saved event/request details:
Title: ${$json.title || 'not provided'}
Start: ${$json.start_datetime || 'not provided'}
End: ${$json.end_datetime || 'not provided'}
Target event ID: ${$json.target_event_id || 'not provided'}
Conflicting event IDs: ${$json.conflicting_event_ids || 'not provided'}
Candidate event IDs: ${$json.candidate_event_ids || 'not provided'}

Latest user reply:
${$json.chatInput || ''}

Classify the latest user reply in relation to this pending action.`
  : null
}}
```

## Show Schedule Formatting Code

```javascript
const events = items
  .filter(item => item.json && Object.keys(item.json).length > 0)
  .sort((a, b) => {
    const aStart = a.json.start?.dateTime || a.json.start?.date || "";
    const bStart = b.json.start?.dateTime || b.json.start?.date || "";
    return aStart.localeCompare(bStart);
  });

const scheduleDate = DateTime
  .fromISO($('Switch').item.json.output.range_start_datetime)
  .setZone('America/Edmonton')
  .toFormat('MMMM d, yyyy');

if (!events.length) {
  return [{
    json: {
      message: `Your schedule for ${scheduleDate} is empty.`
    }
  }];
}

const lines = events.map(event => {
  const start = event.json.start?.dateTime || event.json.start?.date;
  const end = event.json.end?.dateTime || event.json.end?.date;
  const title = event.json.summary || "No title";

  const startTime = start?.includes("T") ? DateTime.fromISO(start).setZone("America/Edmonton").toFormat("h:mm a") : "All day";
  const endTime = end?.includes("T") ? DateTime.fromISO(end).setZone("America/Edmonton").toFormat("h:mm a") : "";

  return endTime
    ? `${startTime} - ${endTime} | ${title}`
    : `${startTime} | ${title}`;
});

return [{
  json: {
    message: `Your schedule for ${scheduleDate}:\n\n${lines.join("\n")}`
  }
}];
```

## Booking Conflict Checker Code

```javascript
const allItems = items;

const events = allItems.filter(item => {
  const event = item.json || {};
  return Boolean(
    event.id &&
    event.start &&
    event.end &&
    (event.start.dateTime || event.start.date) &&
    (event.end.dateTime || event.end.date)
  );
});

const hasConflict = events.length > 0;

const formatTime = (dateString) => {
  if (!dateString) return "unknown time";

  if (dateString.includes("T")) {
    return DateTime
      .fromISO(dateString)
      .setZone("America/Edmonton")
      .toFormat("h:mm a");
  }

  return dateString;
};

const conflictLines = events.map((item, index) => {
  const event = item.json;

  const title = event.summary || "No title";
  const start = formatTime(event.start?.dateTime || event.start?.date);
  const end = formatTime(event.end?.dateTime || event.end?.date);

  return `${index + 1}) ${start} - ${end} | ${title}`;
});

const conflictMessage = hasConflict
  ? `You already have events at that time:\n\n${conflictLines.join("\n")}\n\nDo you want me to replace those events with this new one, or choose another time?`
  : null;

return [
  {
    json: {
      ...($json || {}),
      hasConflict,
      conflictMessage,
      conflictingEvents: events.map(item => item.json)
    }
  }
];
```

## Reschedule Count Found Events Code

```javascript
const events = items
  .filter(item => {
    const event = item.json || {};
    return event.id && event.start && event.end;
  })
  .sort((a, b) => {
    const aStart = a.json.start?.dateTime || a.json.start?.date || "";
    const bStart = b.json.start?.dateTime || b.json.start?.date || "";
    return aStart.localeCompare(bStart);
  });

const eventIds = events.map(item => item.json.id);

let matchStatus = "none";

if (events.length === 1) {
  matchStatus = "single";
}

if (events.length > 1) {
  matchStatus = "multiple";
}

const formatTime = (dateString) => {
  if (!dateString) return "unknown time";

  return DateTime
    .fromISO(dateString)
    .setZone("America/Edmonton")
    .toFormat("h:mm a");
};

const conflictLines = events.map((item, index) => {
  const event = item.json;
  const title = event.summary || "No title";

  const start = event.start?.dateTime || event.start?.date || "";
  const end = event.end?.dateTime || event.end?.date || "";

  const startTime = start.includes("T") ? formatTime(start) : "All day";
  const endTime = end.includes("T") ? formatTime(end) : "";

  return endTime
    ? `${index + 1}) ${startTime} - ${endTime} | ${title}`
    : `${index + 1}) ${startTime} | ${title}`;
});

return [{
  json: {
    ...$('Switch').item.json.output,

    match_count: events.length,
    match_status: matchStatus,
    event_ids_to_delete: eventIds,
    event_id_to_delete: eventIds[0] || null,
    found_events: events.map(item => item.json),

    conflict_message: events.length > 1
      ? `I found more than one event in that time:\n\n${conflictLines.join("\n")}\n\nDo you want me to remove these events and create the new one?`
      : null
  }
}];
```

## Code1 — Prepare Pending Replacement Event IDs

Placement:

```text
confirmed pending replacement TRUE path
```

```javascript
const pending = $('Conflict Memory').item.json;

const eventIds = JSON.parse(pending.conflicting_event_ids || "[]");

return eventIds.map(eventId => ({
  json: {
    event_id_to_delete: eventId,

    title: pending.title,
    start_datetime: pending.start_datetime,
    end_datetime: pending.end_datetime,

    new_start_datetime: pending.start_datetime,
    new_end_datetime: pending.end_datetime,

    pending_action: pending.pending_action,
    sessionId: pending.sessionId,
    pending_row_id: pending.id
  }
}));
```

## Prepare One Create Item Code

Mode:

```text
Run Once for All Items
```

```javascript
const first = $('Code1').first().json;

return [
  {
    json: {
      title: first.title,
      start_datetime: first.start_datetime,
      end_datetime: first.end_datetime,
      new_start_datetime: first.new_start_datetime,
      new_end_datetime: first.new_end_datetime,
      sessionId: first.sessionId,
      pending_action: first.pending_action,
      pending_row_id: first.pending_row_id
    }
  }
];
```

## Normalize Cancel Request Code

```javascript
const output = $json.output || {};

return [
  {
    json: {
      ...$json,
      cancel_search_text: output.event_search_text || null,
      cancel_start_datetime: output.original_datetime || null,
      sessionId: $json.sessionId || $('Merge').item.json.sessionId
    }
  }
];
```

## Filter Cancel Matches Code

```javascript
const searchText = ($json.cancel_search_text || "").toLowerCase();

const events = items
  .filter(item => {
    const event = item.json || {};
    return event.id && event.start && event.end;
  })
  .filter(item => {
    if (!searchText) return true;

    const title = (item.json.summary || "").toLowerCase();
    return title.includes(searchText);
  })
  .sort((a, b) => {
    const aStart = a.json.start?.dateTime || a.json.start?.date || "";
    const bStart = b.json.start?.dateTime || b.json.start?.date || "";
    return aStart.localeCompare(bStart);
  });

const eventIds = events.map(item => item.json.id);

let matchStatus = "none";
if (events.length === 1) matchStatus = "single";
if (events.length > 1) matchStatus = "multiple";

const lines = events.map((item, index) => {
  const event = item.json;
  const title = event.summary || "No title";
  const start = event.start?.dateTime || event.start?.date || "";
  const end = event.end?.dateTime || event.end?.date || "";

  const startTime = start.includes("T")
    ? DateTime.fromISO(start).setZone("America/Edmonton").toFormat("h:mm a")
    : "All day";

  const endTime = end.includes("T")
    ? DateTime.fromISO(end).setZone("America/Edmonton").toFormat("h:mm a")
    : "";

  return `${index + 1}) ${startTime} - ${endTime} | ${title}`;
});

return [{
  json: {
    ...$json,
    match_count: events.length,
    match_status: matchStatus,
    event_ids_to_delete: eventIds,
    event_id_to_delete: eventIds[0] || null,
    found_events: events.map(item => item.json),
    cancel_options_message: lines.join("\n")
  }
}];
```

## Google Calendar Expressions

### Booking Create

Summary:

```text
{{ $('Switch').item.json.output.title }}
```

Start:

```text
{{ $('Switch').item.json.output.start_datetime }}
```

End:

```text
{{ $('Switch').item.json.output.end_datetime }}
```

### Reschedule Single Create

Summary:

```text
{{ $('Switch1').item.json.title }}
```

Start:

```text
{{ $('Switch1').item.json.new_start_datetime }}
```

End:

```text
{{ $('Switch1').item.json.new_end_datetime }}
```

### Multiple Replacement Create

Summary:

```text
{{ $json.title }}
```

Start:

```text
{{ $json.new_start_datetime }}
```

End:

```text
{{ $json.new_end_datetime }}
```

### Delete Event

Event ID:

```text
{{ $json.event_id_to_delete }}
```

## Telegram Confirmation Examples

### Booking

```text
Booked: {{ $('Switch').item.json.output.title }} on {{ DateTime.fromISO($('Switch').item.json.output.start_datetime).setZone('America/Edmonton').toFormat('MMMM d, h:mm a') }}
```

### Reschedule After Create Event

```text
Rescheduled: {{ $json.summary }} on {{ DateTime.fromISO($json.start.dateTime).setZone('America/Edmonton').toFormat('MMMM d, h:mm a') }}
```

### Generic Chat ID

```text
{{ $('Merge').item.json.sessionId }}
```


## Latest Added Code — Cancel Confirm Delete
## Filter Cancel Matches Code — corrected version

Use this version when Google Calendar output no longer carries `cancel_search_text`.

```javascript
const source = $('Normalize Cancel Request').first().json;

const searchText = (source.cancel_search_text || "").toLowerCase();
const sessionId = source.sessionId;
const cancelStartDatetime = source.cancel_start_datetime;

const events = items
  .filter(item => {
    const event = item.json || {};
    return event.id && event.start && event.end;
  })
  .filter(item => {
    if (!searchText) return true;

    const title = (item.json.summary || "").toLowerCase();
    return title.includes(searchText);
  })
  .sort((a, b) => {
    const aStart = a.json.start?.dateTime || a.json.start?.date || "";
    const bStart = b.json.start?.dateTime || b.json.start?.date || "";
    return aStart.localeCompare(bStart);
  });

const eventIds = events.map(item => item.json.id);

let matchStatus = "none";
if (events.length === 1) matchStatus = "single";
if (events.length > 1) matchStatus = "multiple";

const lines = events.map((item, index) => {
  const event = item.json;
  const title = event.summary || "No title";
  const start = event.start?.dateTime || event.start?.date || "";
  const end = event.end?.dateTime || event.end?.date || "";

  const startTime = start.includes("T")
    ? DateTime.fromISO(start).setZone("America/Edmonton").toFormat("h:mm a")
    : "All day";

  const endTime = end.includes("T")
    ? DateTime.fromISO(end).setZone("America/Edmonton").toFormat("h:mm a")
    : "";

  return `${index + 1}) ${startTime} - ${endTime} | ${title}`;
});

return [{
  json: {
    ...source,
    match_count: events.length,
    match_status: matchStatus,
    event_ids_to_delete: eventIds,
    event_id_to_delete: eventIds[0] || null,
    found_events: events.map(item => item.json),
    cancel_options_message: lines.join("
"),
    sessionId,
    cancel_start_datetime: cancelStartDatetime
  }
}];
```

## IF Confirmed Cancel

Placement:

```text
Main Switch: cancel output → IF Confirmed Cancel → FALSE to Normalize Cancel Request
```

Condition:

- First field / value 1:

```text
{{ $json.output.confirmed }}
```

- Condition:

```text
is equal to
```

- Second field / value 2:

```text
true
```

- Output Name:

```text
confirmed_cancel
```

## Prepare Confirm Cancel Delete Code

Placement:

```text
IF Confirmed Cancel TRUE path
```

```javascript
const pending = $('Conflict Memory').item.json;

return [
  {
    json: {
      event_id_to_delete: pending.target_event_id,
      title: pending.title,
      start_datetime: pending.start_datetime,
      end_datetime: pending.end_datetime,
      sessionId: pending.sessionId,
      pending_action: pending.pending_action,
      pending_row_id: pending.id
    }
  }
];
```

## Google Calendar Delete Event — confirmed cancel

Event ID:

```text
{{ $json.event_id_to_delete }}
```

## Data Table Cleanup — confirmed cancel

Delete exact pending row by ID:

```text
id = {{ $('Prepare Confirm Cancel Delete').first().json.pending_row_id }}
```

Settings:

```text
Always Output Data = ON
```

## Telegram Deleted Confirmation

Chat ID:

```text
{{ $('Prepare Confirm Cancel Delete').first().json.sessionId }}
```

Text:

```text
Deleted: {{ $('Prepare Confirm Cancel Delete').first().json.title }} on {{ DateTime.fromISO($('Prepare Confirm Cancel Delete').first().json.start_datetime).setZone('America/Edmonton').toFormat('MMMM d, h:mm a') }}.
```
