---
name: inbox-archive
description: |
  Suggest inbox emails that are "complete" and safe to archive. Shows the list for
  the hiring manager to review and confirm before archiving anything. Use when the user asks "clean up
  my inbox", "archive noise", "what can I archive", or "clear out the junk".
allowed-tools:
  - Bash
  - Read
  - mcp__google-workspace__gmail_query_emails
  - mcp__google-workspace__gmail_get_email
  - mcp__google-workspace__gmail_archive
  - mcp__google-workspace__gmail_bulk_archive
---

> **OpenClaw note (read this first):** the `allowed-tools` list above uses Claude-Code MCP names. On OpenClaw, substitute: `gog --account ryan@ycombinator.com gmail/calendar/...` for Google Workspace, `curl` against `https://api.ashbyhq.com` (HTTP Basic with `/data/.ashby/api_key`, always set `User-Agent`) for Ashby, `curl` against `https://api.ycombinator.com` (Bearer token from `/data/.yc/waas-credentials.json`) for WAAS. Behavior the skill prescribes is the source of truth, not the literal tool names. Full mapping: `recruiting/.claude/skills/_meta/BEFORE-DRAFTING.md` section 5.

> **Before drafting any candidate-facing content, read `recruiting/.claude/skills/_meta/BEFORE-DRAFTING.md` first.** It tells you which voice/job/playbook files to load and the drift-check to run before showing a draft.

> **Note:** Read `recruit-config/user.md` for the hiring manager's name, email, and preferences. Do not hardcode any user-specific values.


**Config**: Read these files before executing — they contain all IDs, tokens, templates, and rules:
- `recruit-config/user.md` — hiring manager email, Zoom link, office, Slack IDs, noise senders
- `recruit-config/ashby.md` — source IDs, archive reasons, email style, draft tool rules
- `recruit-config/waas.md` — WAAS pipeline stages, job IDs, MCP tools` — Slack/Gmail config, source IDs, archive reasons
- `recruit-config/CLAUDE.local.md` — Slack bot token (gitignored)
- `recruit-config/jobs/product-engineer.md` — job ID, stage IDs, candidate bar, outreach template, tone rules

# Inbox Archive

Suggest emails that are safe to archive, then wait for the hiring manager to confirm.

## Procedure

### Step 1: Pull inbox

```
gmail_query_emails(user_id: "<hiring manager email from user.md>", query: "in:inbox newer_than:14d", max_results: 50)
```

### Step 2: Classify each email as archivable or not

An email is **safe to archive** if it matches ANY of these:

#### Definitely archive (noise)
- GitLab MR notifications (merged, assigned, reviewed)
- Ashby stage change notifications ("[Ashby] Candidate X has changed stage")
- Ashby feedback reminders ("[Ashby] Reminder: Please Submit Your Feedback")
- Mode analytics reports
- Calendar accepts/declines from room resources
- DoorDash, Yelp, Brex expenses, Vercel, TrueUp, Zapier, Handshake
- Marketing emails (CATEGORY_PROMOTIONS)
- Self-emails (from hiring manager to hiring manager)
- Partiful invites (already RSVPd or past events)
- YC Calendar event updates

#### Probably archive (completed)
- Candidate emails where the hiring manager already replied AND no follow-up needed
  - Scheduling confirmed + calendar invite sent
  - Rejection sent
  - Question answered
- Vendor emails that are informational (J&J daily updates, Dover sourcing progress)
- Legal/contracts that are countersigned and done (Coastal agreement)
- Ashby application notifications for candidates auto-archived by bot

#### DO NOT archive
- Any email where the candidate is waiting for the hiring manager's reply
- Unread emails from candidates
- Emails requiring a decision (feedback to review, questions to answer)
- Emails from YC colleagues that need a response

### Step 3: Present the suggestion — DO NOT ARCHIVE YET

```
## Inbox Archive Suggestion

### Safe to archive (noise) — {count} emails
| # | From | Subject | Why |
|---|------|---------|-----|
| 1 | GitLab (teammate) | MR !12345 feature_branch (x5) | Already merged |
| 2 | Ashby | Candidate Z applied + archived | Bot handled |
| 3 | Vercel | Terms of Service update | Marketing |
| 4 | TrueUp | 7,866 new jobs | Digest |
| ... | | | |

### Probably safe (completed) — {count} emails
| # | From | Subject | Why |
|---|------|---------|-----|
| 1 | Vendor Contact | YC / Coastal countersigned | Done — contract signed |
| 2 | Jill (J&J) | Monday update: no new candidates | Informational |
| ... | | | |

### Keeping (needs action) — {count} emails
| # | From | Subject | Why keeping |
|---|------|---------|------------|
| 1 | Candidate M | Post-onsite questions | Needs reply |
| 2 | Candidate K | Asking for update | Needs decision |
| ... | | | |

**Archive all {N} "safe" emails?** Or review the "probably safe" list too?
```

### Step 4: Wait for confirmation

**DO NOT call `gmail_archive` or `gmail_bulk_archive` until the hiring manager confirms.**

The hiring manager may:
- "yes" → archive all "safe to archive" emails
- "yes, and the probably safe ones too" → archive both tiers
- "remove #3 from the list" → exclude specific emails
- "skip" → don't archive anything

### Step 5: Execute (only after approval)

1. Collect all approved message IDs
2. Use `gmail_bulk_archive(message_ids)` for efficiency
3. Report: "Archived {N} emails."

## Rules

1. **NEVER archive without explicit approval.** Suggest first, always.
2. **When in doubt, keep it.** If you're not sure whether an email is done, put it in "Keeping."
3. **Never archive unread candidate emails.** Even if they look like noise, an unread email from a candidate deserves review.
4. **Show what you're keeping and why.** The hiring manager should see both what you'd archive AND what you'd keep, so they can catch mistakes.
5. **Group by tier.** "Safe" vs "Probably safe" vs "Keeping" — let the hiring manager decide how aggressive to be.
