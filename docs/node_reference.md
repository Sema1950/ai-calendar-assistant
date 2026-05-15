# Node Reference

Note: `docs/Calendar_assist.json` is the current exported n8n workflow and implementation reference. It is maintained manually by the project owner; documentation should be aligned to it, but this JSON file should not be edited as part of documentation cleanup.

## Main Entry Nodes

```text
Telegram Trigger
Edit Fields
Conflict Memory
Merge
Build Pending Context
AI Agent
Structured Output Parser
Code in JavaScript
Switch
```

`Switch` is the current exported node name for the Main Switch.

## Current Entry Flow

```text
Telegram Trigger
-> Edit Fields
-> Conflict Memory / Data Table
-> Merge
-> Build Pending Context
-> AI Agent
-> Structured Output Parser
-> Code in JavaScript
-> Switch / Main Switch
```

## Cancel Nodes

```text
IF Confirmed Cancel
Normalize Cancel Request
Schedule check2
Filter cancel matches
Delete row(s)2
Insert row
Send a text message5
Prepare Confirm Cancel Delete
Delete an event2
Delete row(s)3
Send a text message6
```

## Cancel Single Confirmation Path

Built and tested:

```text
Main Switch: cancel
-> IF Confirmed Cancel
```

FALSE path:

```text
Normalize Cancel Request
-> Schedule check2
-> Filter cancel matches
-> Delete row(s)2
-> Insert row pending_action = confirm_cancel
-> Send a text message5
```

TRUE path:

```text
Prepare Confirm Cancel Delete
-> Delete Event
-> Delete pending row
-> Telegram deleted confirmation
```

## Planned Cancel Switch

The real Cancel Switch node is not built yet in the JSON export.

Current condition:

```text
Filter cancel matches
-> Delete row(s)2
-> Insert row
-> Send a text message5
```

Next build step:

```text
Add real Cancel Switch with single / multiple / none.
Then connect the none path first.
```

## Cancel Work Not Fully Built

```text
Cancel none path
Cancel multiple match path
select_cancel_target reply handling
reject delete handling
calendar error handling
```

## Design Rules

```text
One AI Agent only.
AI only classifies and extracts structured data.
Deterministic n8n nodes perform real actions.
Data Table is the source of truth for pending actions.
Do not add pending_action values to Structured Output Parser enums.
```

Data Table-only pending actions:

```text
replace_conflicts_decision
confirm_cancel
select_cancel_target
```
