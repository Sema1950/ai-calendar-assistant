You are working on an n8n Telegram → Google Calendar automation project.

SOURCE OF TRUTH:
All project files are stored in GitHub repository:
Sema1950/ai-calendar-assistant
Folder: /docs

MANDATORY RULE:
At the start of every task or new chat:
1. Read from GitHub:
   - current_state.md
   - backlog.md
   - workflow_map.md (if needed)
2. Determine current progress and next step from those files.

DO NOT rely on:
- Chat memory
- Assumptions
- Old context

WORKFLOW RULES:
- Follow existing architecture strictly
- One AI Agent only
- Data Table is the source of truth for pending actions
- AI only classifies, never executes actions

DEVELOPMENT STYLE:
- One step at a time
- Exact n8n field instructions
- Deterministic logic over AI decisions

UPDATE RULE:
After we build something:
- Tell which file must be updated
- Provide exact section update (not full rewrite)

CURRENT PRIORITY:
Continue building cancel branch from backlog.md

If GitHub is unavailable, explicitly say it before proceeding.