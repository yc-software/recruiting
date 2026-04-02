# Recruiting Skills — Architecture

## Overview

A suite of Claude Code skills that act as a recruiting co-pilot. Manages the full hiring loop: sourcing, outreach, scheduling, interviewing, and pipeline management — across Ashby (ATS), Gmail, Google Calendar, Slack, and WAAS.

## Prerequisites

### MCP Servers

These skills rely on MCP (Model Context Protocol) servers for external API access. Each must be installed and configured in your Claude Code MCP settings.

| MCP Server | Purpose | Source | Notes |
|------------|---------|--------|-------|
| **Google Workspace** (`google-workspace`) | Gmail read/query/archive, Calendar read | [j3k0/mcp-google-workspace](https://github.com/j3k0/mcp-google-workspace) | Handles Gmail search, email reading, archiving, and calendar queries. Requires Google OAuth. |
| **Gmail Drafts** (`gmail-drafts`) | Create/send/delete email drafts | Claude AI managed MCP | Write access to Gmail — creating drafts, sending emails, managing labels. Configured via Claude Code MCP settings. |
| **Google Calendar** (`claude_ai_Google_Calendar`) | Create/update/delete calendar events, find free time | Claude AI managed MCP | Write access to Google Calendar. Configured via Claude Code MCP settings. |
| **Slack** | Read channels, send messages | Copilot bot token + curl (NOT the Claude AI Slack MCP) | The Claude AI Slack MCP sends as the user (no notifications to the hiring manager). We use curl + the Copilot bot token instead so messages come from "Copilot" and trigger notifications. Bot token stored in `CLAUDE.local.md`. |
| **Ashby** (`ashby`) | ATS — candidates, applications, pipeline, interview scheduling | [ryankicks/mcp-ashby](https://github.com/ryankicks/mcp-ashby) — fork of [PlenishAI/mcp-ashby](https://github.com/PlenishAI/mcp-ashby) | This fork adds: `interview_schedule_create/update/cancel/list`, `candidate_update`, `candidate_get_resume`. Install: `uv tool install git+https://github.com/ryankicks/mcp-ashby`. Requires `ASHBY_API_KEY` env var. |
| **WAAS API** (`waas`) | Work at a Startup candidate data | [yc-software/waas-mcp](https://github.com/yc-software/waas-mcp) | MCP server for WAAS candidate profiles, messages, notes, and status updates. Install: `uv tool install git+https://github.com/yc-software/waas-mcp`. Uses OAuth2 PKCE — run `waas login` to authenticate. |

### Claude Code Setup

1. **Install Claude Code** — https://claude.com/claude-code
2. **Configure MCP servers** — Add Ashby, Google Workspace, Gmail Drafts, and Google Calendar MCPs
3. **Create secrets file** — `.claude/skills/recruit-config/CLAUDE.local.md` with the Slack Copilot bot token (this file is gitignored)
4. **Verify Ashby access** — Run `/candidate-status` on a known candidate to confirm the Ashby MCP is working

### Slack Setup

1. **Create a channel** — e.g., `#recruit-bot` for the bot to post to
2. **Invite the bot** — `/invite @Copilot` in the channel (the bot must be a member to read/write)
3. **Update config** — Set the channel ID in `ashby.md`

### Ashby Setup

1. **Get API key** — Ashby Settings → API Keys → create a key with read/write access
2. **Set env var** — `ASHBY_API_KEY=your-key` (or configure in MCP server settings)
3. **Get job/stage IDs** — Use the Ashby MCP tools (`job_list`, `interview_stage_list`) to find IDs for your job, or copy from an existing job playbook

### Gmail / Calendar

These use Claude AI's managed MCP integrations. When first invoked, Claude Code will prompt for OAuth authorization. No manual setup needed beyond clicking "Allow."

## Config / Data Separation

Skills contain **process** (how to do things). Config contains **data** (IDs, templates, rules).

```
recruit-config/                   <- Shared config (not a skill)
├── user.md                       <- User-specific: email, Zoom link, office, Slack IDs, noise senders
├── ashby.md                      <- Org-wide: source IDs, archive reasons, email style, draft tool rules
├── waas.md                       <- WAAS pipeline stages, job IDs, MCP tools
├── CLAUDE.local.md               <- Secrets: bot token (GITIGNORED — not checked in)
├── README.md                     <- This file
└── jobs/
    └── product-engineer.md       <- Complete role playbook:
                                     • Ashby job ID + pipeline stage IDs
                                     • Outreach email template
                                     • Tone & style rules
                                     • Candidate evaluation bar
                                     • Slack recommendation format
                                     • Exception candidate rules
```

## Skills

| Skill | Purpose | Invocation |
|-------|---------|------------|
| `recruit` | Router — dispatches to the right skill | "recruit", "candidates" |
| `recruit-watch` | Background monitor — polls Slack, Gmail, Ashby on a loop | `/loop 10m /recruit-watch` |
| `outreach` | Draft cold outreach to new candidates | `/outreach Jane Doe` |
| `reply` | Reply to candidates in existing threads | `/reply John Smith` |
| `reject` | Draft rejection email, archive in Ashby | `/reject Candidate C` |
| `triage` | Start-of-day view — inbox, pipeline, what needs action | `/triage` |
| `pipeline` | Full pipeline status by stage | `/pipeline` |
| `schedule` | Find calendar availability, batch schedule screens | `/schedule` |
| `review-applicants` | Review all new inbound applicants | `/review-applicants` |
| `candidate-status` | Look up a single candidate's full state | `/candidate-status Jane Doe` |
| `candidate-brief` | Quick one-paragraph background brief | `/candidate-brief Jane Doe` |
| `inbox-archive` | Suggest safe-to-archive emails | `/inbox-archive` |

## Adding a new role

1. Create `recruit-config/jobs/your-role.md` with:
   - Ashby job ID and pipeline stage IDs
   - Outreach email template (with team facts baked in)
   - Tone rules and hook examples
   - Candidate evaluation bar (hard reqs, signals, auto-archive criteria)
   - Slack recommendation format
2. All skills will pick it up — they read from config at runtime

## Handing to someone else

1. Update `ashby.md` with their Slack user ID, email, and channel
2. Create `CLAUDE.local.md` with their bot token
3. Update the job playbook template with their intro line and team details
4. Skills stay the same — all process, no personal data

## Security

- **Bot token**: Only in `CLAUDE.local.md` which is gitignored (pattern: `**/*CLAUDE.local.md`)
- **Email addresses**: The hiring manager's work email is in `ashby.md` — semi-public, low risk
- **Ashby UUIDs**: Internal IDs, useless without API credentials (managed by MCP server)
- **No candidate PII** is stored in config — all candidate data lives in Ashby/Gmail

## Known remaining hardcoded values

Some skills still have hardcoded email addresses and Ashby IDs inline (in code examples). These should reference `user.md` instead — the config block at the top of each skill tells sessions to read from config. Full deduplication is a future cleanup task.
