---
name: candidate-status
description: |
  Look up the full status of a recruiting candidate by name. Cross-references Ashby, Gmail,
  Calendar, and Drafts to return a unified view. Use when the user asks "status of X",
  "where is X", "what's happening with X", or any variation of checking on a candidate.
allowed-tools:
  - mcp__google-workspace__gmail_query_emails
  - mcp__google-workspace__gmail_get_email
  - mcp__google-workspace__calendar_get_events
  - mcp__gmail-drafts__gmail_list_drafts
  - mcp__ashby__candidate_search
  - mcp__ashby__candidate_info
  - mcp__ashby__application_info
  - mcp__ashby__application_feedback_list
  - mcp__ashby__interview_schedule_list
  - mcp__ashby__interview_event_list
  - mcp__waas__candidate_show
  - mcp__waas__candidate_status_show
  - mcp__waas__candidate_messages_list
  - mcp__waas__candidate_notes_list
  - Read
  - Bash
---

> **Note:** Read `recruit-config/user.md` for the hiring manager's name, email, and preferences. Do not hardcode any user-specific values.


**Config**: Read these files before executing — they contain all IDs, tokens, templates, and rules:
- `recruit-config/user.md` — hiring manager email, Zoom link, office, Slack IDs, noise senders
- `recruit-config/ashby.md` — source IDs, archive reasons, email style, draft tool rules
- `recruit-config/waas.md` — WAAS pipeline stages, job IDs, MCP tools` — Slack/Gmail config, source IDs, archive reasons
- `recruit-config/CLAUDE.local.md` — Slack bot token (gitignored)
- `recruit-config/jobs/product-engineer.md` — job ID, stage IDs, candidate bar, outreach template, tone rules

# Candidate Status Lookup

Look up the full recruiting status of a candidate by name. Returns a unified view from all sources.

## Unified Resolver

Read `recruit-config/resolve-candidate.md` for the shared procedure to merge data from all 4 sources (WAAS MCP, Ashby, Gmail, Calendar). Follow that procedure and present the unified view format defined there.

## Input

The user provides a candidate name (or multiple names). Examples:
- `/candidate-status Jane Doe`
- `/candidate-status John Smith, Candidate C`
- "what's the status of Jane?"

## Procedure

For EACH candidate name provided, run ALL of the following queries. Run independent queries in parallel.

### Step 1: Ashby lookup

```
candidate_search(name: "<candidate name>")
```

From the result, extract:
- `name`, `primaryEmailAddress`, `position`, `company`, `school`
- `socialLinks` → find LinkedIn URL
- `applicationIds` → take the first one
- `customFields` → look for "YC Bio Summary", "YC Bio Stage", "YC Bio Source"
- `profileUrl` → Ashby link

If no results, note "Not in Ashby" and continue with email-only data.

### Step 2: Ashby application detail

If applicationIds exist:
```
application_info(applicationId: "<first application ID>")
```

From the result, extract:
- `currentInterviewStage.title` → the pipeline stage
- `status` → Active / Archived / Hired
- `archiveReason.text` → if archived, why
- `applicationHistory` → list of stage transitions with dates and actors
- `source.title` → how they entered (WAAS, Jack and Jill, Ashby Chrome Extension, Applied, etc.)

### Step 3: Gmail — latest from candidate

```
gmail_query_emails(user_id: "<hiring manager email from user.md>", query: "from:<candidate email> newer_than:14d", max_results: 3)
```

Extract: most recent email subject, snippet, date.

### Step 4: Gmail — latest from the hiring manager

```
gmail_query_emails(user_id: "<hiring manager email from user.md>", query: "in:sent to:<candidate email> newer_than:14d", max_results: 3)
```

Extract: most recent sent email subject, snippet, date.

**IMPORTANT**: Scan the sent emails for any request from the candidate to use a different email address.

### Step 5: Gmail drafts check

```
gmail_list_drafts(maxResults: 10)
```

Cross-reference the draft `threadId` values against the candidate's email thread IDs (from steps 3 and 4). Report whether a draft exists for this candidate.

### Step 6: Calendar lookup

```
calendar_get_events(user_id: "<hiring manager email from user.md>", calendar_id: "primary", time_min: <today>, time_max: <4 weeks from today>, timezone: "America/Los_Angeles")
```

Search the results for events containing the candidate's name (first name, last name, or full name). For each match, extract:
- Event summary (title)
- Start/end time
- Location
- `attendees[]` → find the candidate's email
- `attendees[].responseStatus` → **this is the key field**:
  - `accepted` → confirmed
  - `declined` → rejected the invite
  - `tentative` → tentatively accepted
  - `needsAction` → invite sent, no response yet

**If the calendar result is too large**, use Bash + python to parse the saved file and filter for the candidate name.

### Step 7: WAAS lookup (if candidate has a short_id)

If the candidate was found in WAAS (via the unified resolver), also pull:
```
candidate_notes_list(short_id)
```
Include WAAS notes in the output alongside Ashby notes.

### GATE — Do not present results until this is complete

For EACH candidate, verify you have queried ALL of:
- [ ] Ashby search + application info (or confirmed "not in Ashby")
- [ ] Gmail inbox (14d)
- [ ] Gmail sent (14d)
- [ ] Gmail drafts
- [ ] Calendar (4 weeks)
- [ ] WAAS profile + status + notes (if candidate has short_id)

Do not present partial results. If a query failed, note the failure — do not silently skip it.

## Output Format

Present the result as a structured summary:

```
Checked: Ashby, Gmail inbox (14d), Gmail sent (14d), Drafts, Calendar (4 weeks), WAAS ({yes/no/not applicable})
Skipped: [none / list with reason]

## <Candidate Name>

| Field | Value |
|-------|-------|
| **Email** | <email> (note if they requested a different one) |
| **LinkedIn** | <url> |
| **Ashby** | <profile url> |
| **Background** | <position @ company, school — from Ashby or email> |
| **Ashby Stage** | <stage> (<Active/Archived>) |
| **Source** | <how they entered: WAAS, J&J, Applied, Sourced, etc.> |
| **Calendar** | <event details + responseStatus, or "No upcoming events"> |
| **Last email (them)** | <date — snippet> |
| **Last email (hiring manager)** | <date — snippet> |
| **Draft exists** | <Yes (threadId) / No> |
| **Days waiting** | <days since candidate's last email, if they're waiting on the hiring manager> |
| **Action needed** | <what the hiring manager should do next, or "None — ball is in candidate's court"> |

Sources checked: Ashby, Gmail inbox (14d), Gmail sent (14d), Gmail drafts, Calendar (4 weeks)
```

For multiple candidates, show one block per candidate.

## Rules

1. **Query everything before presenting.** Never skip a source.
2. **Never say "scheduled" unless responseStatus is "accepted".** Use "invite sent, not yet accepted" for needsAction.
3. **Check Ashby for interview schedules.** Use `interview_schedule_list(applicationId)` to see scheduled interviews in Ashby, and cross-reference with Google Calendar for responseStatus.
4. **Check for email address changes** in the email thread before reporting the email.
5. **Flag ambiguous situations** — don't guess, tell the hiring manager what you're unsure about.
6. **Compute days of week correctly.** March 20, 2026 is a Friday. Always verify.
7. **If Ashby has no background detail**, infer from email threads (J&J intro descriptions, WAAS profiles, the hiring manager's outreach text).
