---
name: outreach
description: |
  Draft personalized cold outreach to a candidate the hiring manager hasn't contacted yet. Pulls background
  from Ashby, personalizes the hook, checks calendar for a time to propose, and sends.
  Use when the user says "reach out to X", "outreach to X", "email X" for a new candidate,
  or when sourcing passive candidates from WAAS AI Sourcer or LinkedIn.
allowed-tools:
  - Bash
  - Read
  - mcp__google-workspace__gmail_query_emails
  - mcp__google-workspace__gmail_get_email
  - mcp__google-workspace__calendar_get_events
  - mcp__claude_ai_Gmail__gmail_create_draft
  - mcp__gmail-drafts__gmail_send_draft
  - mcp__gmail-drafts__gmail_list_drafts
  - mcp__ashby__candidate_search
  - mcp__ashby__candidate_info
  - mcp__ashby__application_info
  - mcp__ashby__application_change_stage
  - mcp__waas__candidate_show
  - mcp__waas__candidate_status_show
  - mcp__waas__candidate_status_update
  - mcp__waas__candidate_messages_list
  - mcp__waas__candidate_message_send
---

> **Note:** Read `recruit-config/user.md` for the hiring manager's name, email, and preferences. Do not hardcode any user-specific values.


# Outreach — Cold Email to New Candidate

Draft personalized cold outreach to a candidate the hiring manager has never contacted. Starts a new thread.

**IMPORTANT**: Before drafting, read the config files:
- `recruit-config/user.md` — hiring manager email, Zoom link, office, Slack IDs, noise senders
- `recruit-config/ashby.md` — source IDs, archive reasons, email style, draft tool rules
- `recruit-config/waas.md` — WAAS pipeline stages, job IDs, MCP tools` — Slack/Gmail config, source IDs, archive reasons
- `recruit-config/CLAUDE.local.md` — Slack bot token (gitignored)
- `recruit-config/jobs/product-engineer.md` — outreach template, tone rules, candidate bar, Ashby job/stage IDs

These files are the source of truth for all job-specific and tone details. Do not hardcode any of those values in this skill.

## When to use this vs `/reply`

| Situation | Use |
|-----------|-----|
| Candidate has an existing email thread | `/reply` |
| **New candidate, no prior contact** | **`/outreach` — this skill** |

## Input

- `/outreach Jane Doe`
- `/outreach John Smith, Candidate C`
- "reach out to Peter Urda"
- "email Jane" (routed from `/recruit` when no existing thread)

## Procedure

### Step 1: Read config
Read all three config files to get the current template, team facts, tone rules, and Ashby IDs.

### Step 2: Gather candidate background
1. **Ashby**: `candidate_search(name)` → email, company, school, position, LinkedIn, location
2. **If not in Ashby**: Ask the hiring manager to add them first (via Chrome Extension or WAAS export), or provide email manually
3. **Check for existing threads**: `gmail_query_emails` for any prior emails to/from this person. If a thread exists, redirect to `/reply` instead.

### Step 3: Check calendar for a time to propose
Check the hiring manager's calendar for available 30-min slots in the next 7 business days. Find an open slot and propose it. If multiple candidates are being outreached in the same batch, stagger the proposed times.

### Step 4: Personalize the hook
Write one sentence connecting the candidate's specific background to YC's work. Follow the hook pattern and tone rules from `product-engineer.md`.

### Step 5: Draft the outreach
Use the template from the job config file (`jobs/product-engineer.md`). Fill in the personalized hook and proposed time. Follow all style rules from the job config.

### Email formatting rule

**No hard line wrapping.** Write each paragraph as a single continuous line — let the email client handle wrapping. Never insert `\n` within a paragraph. Only use newlines between paragraphs.

### Step 6: Present for approval — DO NOT SEND YET

```
## Outreach: {Name}
**To:** {email}
**Subject:** Reaching out from Y Combinator
**Source:** {WAAS/Applied/J&J/Chrome Extension}
**Background:** {one-line summary}
**Proposed time:** {day and time} PT

---
{draft email}
---

**d)** draft — create in Gmail drafts
**s)** send
**e)** edit — tell me what to change
**?)** something else
```

For multiple candidates, present all drafts first, then ask "send all?" or let the hiring manager approve one by one.

### Step 7: Wait for approval

| User says | Action |
|-----------|--------|
| **d** | Create Gmail draft |
| **s** | Send email |
| **e** | Ask what to change, update, re-present |
| **?** | Ask the user what they want |

### Choosing the channel: Gmail vs WAAS messaging

Before drafting, determine the right channel:

1. **Check if the candidate has a WAAS short_id** — look in Ashby custom fields, or check if they came from a WAAS application email.
2. **Check for existing WAAS messages** via `candidate_messages_list(short_id)`.

| Situation | Channel |
|-----------|---------|
| Candidate has WAAS short_id and no prior email thread | **WAAS messaging** — messages appear as emails to the candidate |
| Candidate is in Ashby but not WAAS (sourced via Chrome Extension, J&J, etc.) | **Gmail** — new thread |
| Candidate has both WAAS and email threads | **Gmail** — use the existing email thread via `/reply` |

When using WAAS messaging:
- No draft concept — options become **s/e/?** (no "d")
- After sending, update WAAS status: `candidate_status_update(short_id, state: "reviewing", pipeline_stage: "Reached Out")`

### Step 8: Execute (only after approval)

1. **Send email**: `mcp__claude_ai_Gmail__gmail_create_draft(to, subject, body, contentType: "text/html")` → `gmail_send_draft(draftId)`
2. **Move in Ashby** (if applicable): `application_change_stage` to "Reached Out" stage (get ID from job config)
3. **Verify (MANDATORY)**: DO NOT report success until you have re-queried sent mail to confirm delivery. If you moved Ashby stage, check `application_info` to confirm. Report what you verified.

#### WAAS messaging channel
1. **Send message**: `candidate_message_send(short_id, message)`
2. **Update WAAS stage**: `candidate_status_update(short_id, state: "reviewing", pipeline_stage: "Reached Out")`
3. **Move in Ashby** (if also in Ashby): `application_change_stage` to "Reached Out" stage

### Step 9: After sending

Report what was done:
```
Sent outreach to {Name} ({email}) via {Gmail/WAAS}. Proposed {day/time} PT.
Ashby: moved to Reached Out.
WAAS: moved to Reached Out.
```

## Rules

1. **NEVER send without approval.** Always present first with d/s/e/? options.
2. **Always personalize the hook.** Never send the template verbatim without a personalized opening.
3. **Always read config files first.** Never hardcode team size, template, tone rules, or Ashby IDs.
4. **Always propose a specific time.** Check the hiring manager's calendar first.
5. **Stagger times for batch outreach.** Don't propose the same slot to two candidates.
6. **Check for existing threads first.** If there's already an email thread, use `/reply` instead.
7. **New email, not a thread reply.** This creates a new thread — no threadId.
8. **Move to Reached Out in Ashby after sending** (get stage ID from job config).

## Gmail & Calendar Rules

- **ASCII only in emails.** No em dashes, curly quotes, or unicode. Use `--` instead of em dashes and straight quotes. The plain-text Gmail MCP tool mangles unicode into garbage.
- **Phone screen calendar invites** use the title `<Candidate First Name> and Ryan` -- no "Phone Screen" label. Later stages (tech screen, onsite, partner) include the stage name.
- **Use `mcp__claude_ai_Gmail__gmail_create_draft`** (not `mcp__gmail-drafts__gmail_create_draft`) when creating email drafts. Set `contentType: "text/html"` and use `<p>` tags. The `gmail-drafts` tool only supports `text/plain` which hard-wraps lines at 78 characters, making emails look broken.

## Follow-Up Email Rules

When drafting follow-up emails to candidates who haven't responded to initial outreach:

### Tone
- **Don't say "following up," "circling back," "bumping this," "last note from me," or "I know I've tried a couple times."** The recipient doesn't care about your outreach cadence. Get to the strongest lead immediately.
- **No hyperbole.** Don't use "exactly," "perfect fit," "ideal candidate," or superlatives about the candidate's background. Keep it grounded.
- **Don't flatter.** Don't say things like "exactly the depth we need" about their experience. It reads as recruiter-speak.

### Content
- **Follow the tone and content rules in the job playbook** (`recruit-config/jobs/`) for role-specific framing, hooks, and what to emphasize.
- **Personalize based on their background** — reference specific things they've built or done, not generic bullet points.

### Structure
- **Two paragraphs max.** First paragraph: the hook + why their background is relevant. Second paragraph: the ask.
- **The call-to-action must be on its own line**, separated from the body. Don't bury it in the middle of a paragraph.
- **Combine the call time and coffee offer in one line:** "Would Thursday at 9am or Friday at 11am work for a call? Also happy to grab coffee at our Dogpatch office if you're nearby."

### Process
- **Always check Gmail for prior outreach** before presenting candidates as "never contacted." Search by both email and name.
- **Use `contentType: "text/html"`** when creating drafts to avoid Gmail's plain text line wrapping.
