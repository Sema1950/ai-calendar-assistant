# Project Overview

Note: `docs/Calendar_assist.json` is the current exported n8n workflow and implementation reference. It is maintained manually by the project owner; documentation should be aligned to it, but this JSON file should not be edited as part of documentation cleanup.

## Project Name

AI Scheduler — n8n Telegram Calendar Automation

## Purpose

This project is an n8n automation that lets a user manage Google Calendar through Telegram messages using an AI Agent for language understanding and deterministic n8n workflow logic for safe execution.

The goal is to allow natural-language scheduling commands such as:

- Show my schedule for today
- Book an appointment tomorrow at 3 PM
- Reschedule dinner to walk time
- Replace existing events in a time range with a new event
- Cancel an event
- Check whether a time is available

The system is designed to feel conversational while keeping all calendar actions controlled, traceable, and safe.

## ChatGPT Project Instructions

This project includes a dedicated instruction file for ChatGPT project behavior:

/docs/chatgpt_project_instructions.md

Purpose:

Defines how ChatGPT should work inside this project.
Requires reading current project state before giving build instructions.
Enforces one-step-at-a-time guidance.
Enforces exact n8n node field instructions.
Keeps AI Agent logic separate from deterministic workflow execution.

This file is for ChatGPT project operation only.

It does not change the n8n workflow architecture, AI Agent prompt, Structured Output Parser schema, or Data Table schema.

## Main Goal

The main goal is to build a reliable AI-assisted scheduling assistant that can:

1. Understand user intent from Telegram messages.
2. Extract structured scheduling data.
3. Route the request to the correct workflow branch.
4. Check Google Calendar before making changes.
5. Prevent unsafe actions such as deleting or replacing events without confirmation.
6. Use Data Table state to handle multi-turn decisions.
7. Send clear Telegram responses back to the user.

## Core Principle

The AI Agent does not perform calendar actions directly.

The AI Agent only classifies the message and extracts structured data.

Actual execution is handled by deterministic n8n nodes:

- Switch nodes
- Code nodes
- Google Calendar nodes
- Telegram nodes
- Data Table nodes
- Merge nodes
- IF nodes

This keeps the system safer and easier to debug.

## High-Level Workflow

The general workflow starts with a Telegram message.

The message goes through normalization, pending-state lookup, AI classification, code-based correction, and routing.

High-level flow:

```text
Telegram Trigger
→ Edit Fields
→ Data Table: Get row(s)
→ Merge
→ Build Pending Context
→ AI Agent
→ Structured Output Parser
→ Code in JavaScript
→ Main Switch
→ Specialist branch
```

Main Switch routes requests into:

```text
clarification
show_schedule
booking
reschedule
cancel
availability
fallback
```

## AI Agent Role

The AI Agent acts as a scheduling-intent router.

It receives:

- Current Telegram message
- Session ID
- Current datetime
- Pending state from Data Table
- Pending context from `Build Pending Context`
- Simple Memory context

It outputs structured data with fields such as:

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

The Structured Output Parser enforces the schema.

## n8n Execution Role

n8n nodes perform the real work:

- Google Calendar nodes search, create, and delete events.
- Code nodes normalize, filter, sort, and prepare data.
- Switch and IF nodes route based on deterministic values.
- Telegram nodes send questions and confirmations.
- Data Table nodes store and retrieve pending decisions.

## Key Safety Design

Calendar deletion and replacement require confirmation when there is uncertainty or when multiple events may be affected.

For example:

If a new event overlaps with existing events, the workflow does not immediately overwrite the calendar.

Instead, it:

1. Finds the conflicting events.
2. Saves the pending state in Data Table.
3. Sends a Telegram question to the user.
4. Waits for confirmation.
5. Deletes and creates events only after confirmation.

## Pending State

The workflow uses an n8n Data Table named:

```text
pending_calendar_actions
```

This table stores unfinished multi-turn actions, such as:

```text
replace_conflicts_decision
confirm_cancel
select_cancel_target
```

Pending state is necessary because AI memory alone is not reliable enough for workflow state management.

The Data Table makes the workflow deterministic across multiple Telegram messages.

## Current Supported Capabilities

The project supports or is being built to support:

```text
show_schedule
booking
reschedule
cancel
availability
clarification
```

### Show Schedule

The workflow can retrieve and format calendar events for a requested day or range.

### Booking

The workflow can create a new event if the requested time is free.

If the requested time overlaps existing events, it asks for confirmation before replacing anything.

### Reschedule

The workflow supports:

- Simple reschedule: one existing event is replaced with one new event.
- Multi-event replacement: several events in a time range can be removed and replaced by one new event after confirmation.
- None found path: if no matching event is found, the user is asked which event to reschedule.

### Cancel

The cancel branch is being designed to safely find matching events, confirm deletion, handle multiple matches, and delete only after confirmation.

### Clarification

If required information is missing, the workflow asks one short clarification question and waits for the next Telegram reply.

## Design Philosophy

The workflow follows this rule:

```text
AI suggests. Code and workflow logic decide.
```

AI is useful for understanding messy human language.

Code, Switch, IF, and Data Table logic are used for safety-critical decisions such as:

- Whether to delete events
- Which events to delete
- Whether the user confirmed
- Whether to clear pending state
- Whether to route to booking, reschedule, cancel, or clarification

## Long-Term Goal

The long-term goal is to turn this into a reliable personal scheduling assistant that can manage calendar operations through Telegram with:

- Natural language input
- Safe confirmations
- Multi-turn memory through Data Table state
- Clear Telegram feedback
- Predictable routing
- Minimal manual calendar management


## Latest Project Update
Date: 2026-05-04

The cancel branch was advanced significantly.

Completed and tested:

```text
cancel single match
→ ask user for confirmation
→ save pending_action = confirm_cancel
→ user replies yes
→ IF Confirmed Cancel TRUE
→ Prepare Confirm Cancel Delete
→ Google Calendar Delete Event
→ Data Table cleanup by exact pending row ID
→ Telegram deleted confirmation
```

Confirmed working result:

```text
User requested deletion of Evening walk.
Workflow found one matching event.
Workflow saved confirm_cancel pending row.
User replied yes.
Workflow deleted the correct Google Calendar event.
Pending Data Table row was deleted.
Telegram confirmation was sent.
```
