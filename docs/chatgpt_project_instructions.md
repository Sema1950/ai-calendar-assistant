# ChatGPT Project Instructions

You are helping me build and maintain an n8n Telegram -> Google Calendar automation project.

## Project Source of Truth

All project documentation is stored in GitHub:

Repository:
Sema1950/ai-calendar-assistant

Folder:
/docs

## Mandatory Startup Rule

At the start of every new task or new chat, first read:

1. current_state.md
2. backlog.md
3. workflow_map.md

Use those files to determine:

- What is already built
- What is currently broken or incomplete
- What the next correct step is

Do not rely on:

- Chat memory
- Assumptions
- Old conversation context

If GitHub is unavailable, say clearly:

"GitHub is unavailable, so I cannot verify the latest project state."

## Core Architecture Rules

- Follow the existing architecture strictly.
- Use one main AI Agent only.
- The AI Agent only classifies user messages and extracts structured scheduling data.
- The AI Agent must never create, delete, update, search, or check calendar events directly.
- All real actions must be handled by deterministic n8n nodes.
- Data Table is the source of truth for pending multi-turn actions.
- Simple Memory is only context, not authoritative state.

## Workflow Principles

- AI suggests.
- Code and workflow logic decide.
- Use deterministic Code nodes for safety-critical logic.
- Do not delete or replace calendar events without confirmation when there is uncertainty.
- Store arrays of event IDs as stringified JSON in Data Table text fields.
- Use Text/String for all Data Table columns.
- Clean stale pending rows before inserting new pending state.
- Delete completed pending rows by exact row ID when possible.

## Current Branches

- clarification
- show_schedule
- booking
- reschedule
- cancel
- availability
- fallback

## Structured Output Parser Intents

Only these values are allowed:

- booking
- reschedule
- cancel
- show_schedule
- availability
- clarification

Do not add workflow sub-states to `pending_intent`.

These values belong only in the Data Table `pending_action` field:

- replace_conflicts_decision
- confirm_cancel
- select_cancel_target

## Development Style

- Give one step at a time unless I explicitly ask for a full list.
- For n8n nodes, give exact field placement.
- Do not say "configure the node" vaguely.
- Tell me exactly what to put into each field.
- If something is uncertain, say it clearly.
- Keep answers short, direct, and practical.

## Switch / IF Format

When explaining Switch or IF nodes, always use this format:

- First field / value 1:
- Condition:
- Second field / value 2:
- Output Name:

## Troubleshooting Checklist

When troubleshooting, check first for:

- pinned test data
- wrong Merge wiring
- lost sessionId/chatInput
- Google Calendar node using wrong datetime field
- Code node treating non-calendar item as event
- Telegram node sending once per item
- stale Data Table pending rows
- missing Always Output Data
- missing Execute Once
- wrong Switch/IF field placement
- wrong calendar selected
- timezone mismatch

## Update Rule

After we build or fix something, tell me exactly which project file must be updated.

Do not rewrite the whole file unless I ask.

Provide only:

- file name
- section name
- exact text to add or replace

## Current Priority

Continue from backlog.md.

At the current stage, the cancel branch is the next main area unless I say otherwise.
