---
name: schedule
description: |
  Find open blocks on the hiring manager's calendar and batch schedule phone screens or tech screens.
  Two modes: "when am I free?" (show availability) and "schedule my calls" (match candidates
  to blocks and create events). Use when the user says "schedule", "when am I free", "find me
  time", "batch phone screens", "line up candidates", or any variation.
allowed-tools:
  - Bash
  - Read
  - Skill
  - mcp__google-workspace__gmail_query_emails
  - mcp__google-workspace__gmail_get_email
  - mcp__google-workspace__calendar_get_events
  - mcp__google-workspace__calendar_create_event
  - mcp__google-workspace__calendar_list
  - mcp__claude_ai_Gmail__gmail_create_draft
  - mcp__gmail-drafts__gmail_send_draft
  - mcp__gmail-drafts__gmail_list_drafts
  - mcp__ashby__candidate_search
  - mcp__ashby__candidate_info
  - mcp__ashby__application_info
  - mcp__ashby__application_list
  - mcp__ashby__interview_schedule_create
  - mcp__ashby__interview_schedule_list
  - mcp__ashby__interview_event_list
  - mcp__ashby__lookup
  - mcp__waas__candidate_show
  - mcp__waas__candidate_status_show
  - mcp__waas__candidate_status_update
---

> **Note:** Read `recruit-config/user.md` for the hiring manager's name, email, and preferences. Do not hardcode any user-specific values.


**Config**: Read these files before executing — they contain all IDs, tokens, templates, and rules:
- `recruit-config/user.md` — hiring manager email, Zoom link, office, Slack IDs, noise senders
- `recruit-config/ashby.md` — source IDs, archive reasons, email style, draft tool rules
- `recruit-config/waas.md` — WAAS pipeline stages, job IDs, MCP tools` — Slack/Gmail config, source IDs, archive reasons
- `recruit-config/CLAUDE.local.md` — Slack bot token (gitignored)
- `recruit-config/jobs/product-engineer.md` — job ID, stage IDs, candidate bar, outreach template, tone rules

# Schedule

Find open time and batch schedule candidate calls. Combines calendar availability with pipeline needs.

## Input

- `/schedule` → show availability + suggest who to schedule
- `/schedule when am I free` → just show open blocks
- `/schedule Monday` → show just Monday's availability
- `/schedule 4 calls Thursday` → fit 4 phone screens into Thursday
- `/schedule tech screens` → find 1-hour blocks for tech screens

## Part 1: Find Open Blocks

### Step 1: Pull calendar events

```
calendar_get_events(
  user_id: "<hiring manager email from user.md>",
  calendar_id: "primary",
  time_min: <start of range>,
  time_max: <end of range>,
  timezone: "America/Los_Angeles"
)
```

If the result is too large, save to file and parse with python.

### Step 2: Find open blocks

Working hours: **9:00 AM - 5:00 PM PT**, Monday through Friday.

For each business day in the range:
1. List all non-cancelled events with their start/end times
2. Find gaps between events that are >= minimum block size
3. Only count gaps within working hours (9am-5pm)
4. Ignore all-day events (they don't block time slots) — BUT note them (Demo Day, batch events, onsites)

### Step 3: Calculate capacity

For each open block:
- Phone screens = floor(duration / 30 min)
- Tech screens = floor(duration / 60 min)

### Step 4: Present availability

```
## Available Blocks (Mar 23-28, 2026)
Working hours: 9am-5pm PT

| Day | Block | Duration | Fits |
|-----|-------|----------|------|
| Mon 3/23 | 1:45-4:00pm | 2h 15m | 4 phone screens or 2 tech screens |
| Tue 3/24 | — | (Demo Day) | — |
| Wed 3/25 | 2:00-5:00pm | 3h | 6 phone screens or 3 tech screens |
| Thu 3/26 | 2:00-5:00pm | 3h | 6 phone screens |
| Fri 3/27 | 10:15-5:00pm | 6h 45m | 13 phone screens |

Best day for a big block: **Friday 3/27** (6h 45m open)
```

If the user only asked "when am I free" — stop here. Otherwise, continue to Part 2.

## Part 2: Match Candidates to Blocks

### Step 5: Find candidates needing scheduling

Pull from pipeline — candidates that need phone screens or tech screens:
- Candidates who proposed times but the hiring manager hasn't picked
- Candidates who confirmed a time but no calendar invite sent
- Candidates who need a reschedule
- New intros (J&J, WAAS) with no reply yet
- Candidates at Tech Screen needing exercise scheduling

### Step 6: Match candidates to the best block

Find the single best day that:
1. Has a block large enough to fit all candidates
2. Is soonest (don't push to next week if this week works)
3. Doesn't conflict with Demo Day, batch events, or onsites

If a candidate proposed specific times, check if those times overlap with the suggested block. Flag conflicts.

### Step 7: Present the proposal — DO NOT EXECUTE

```
## Schedule Suggestion

Best day: **Thursday March 26** | Block: **2:00-5:00pm PT**
Fits: 6 phone screens (30 min each)

| Slot | Candidate | Background | What to send |
|------|-----------|------------|-------------|
| 2:00-2:30 | Candidate Y | Hedge Fund + Tech Corp (J&J intro) | Reply to intro, propose Thu 2pm Zoom |
| 2:30-3:00 | Candidate J | Intern candidate | Confirm Thu 2:30pm |
| 3:00-3:30 | Candidate G | Block, Tech Lead | Already proposed Thu 3pm in outreach |
| 3:30-4:00 | Candidate Z | PhD, former CTO | Already proposed Thu 3:30pm in outreach |

**Conflicts / notes:**
- Candidate G and Jamie already received outreach with these times proposed
- Candidate J proposed times — Thu 2:30pm fits his availability

**Ready to go?** Say "yes" to create all calendar events and Gmail drafts, or edit the lineup.
```

### Step 8: Wait for confirmation

**DO NOT create calendar events or Gmail drafts until the hiring manager explicitly approves.**

The hiring manager may:
- "yes" → create all events + drafts
- Edit the lineup ("swap X and Y", "remove Z", "move to Friday")
- Ask for a different day
- Ask for more candidates to add

### Step 9: Execute (only after approval)

For each approved candidate:

#### 9a. Ensure candidate exists in Ashby
Before scheduling, check if the candidate has an Ashby application:
1. **Search Ashby** by name (`candidate_search`) and by email
2. **If not found**: create the candidate (`candidate_create` with name, email, LinkedIn, source) → create application (`application_create` with jobId, candidateId, interviewStageId for Phone Screen, sourceId) → note this was auto-created
3. **If found but no application on this job**: create the application
4. **If found with application**: proceed

#### 9b. Create Google Calendar event
No "screen"/"interview" in candidate-facing text:
- Title: `"{First name} and <hiring manager first name from user.md>"`
- Zoom: `"Thanks for making time to chat!\n\nJoin Zoom: <Zoom link from user.md>\n\n<hiring manager first name from user.md>"`
- In-person: Include office address from user.md + parking instructions
- Add candidate email + hiring manager email (from user.md) as attendees
- Create with `sendUpdates: "none"` first so the hiring manager can review, then send on confirmation

#### 9c. Create Ashby interview schedule
```
interview_schedule_create(
  applicationId: "app-uuid",
  interviewEvents: [{
    startTime: "2026-03-25T15:00:00.000Z",
    endTime: "2026-03-25T15:30:00.000Z",
    interviewId: "<interview ID from recruit-config/jobs/ for this role>",
    interviewers: [
      {email: "<hiring manager email from user.md>", feedbackRequired: true}
    ]
  }]
)
```
Use `lookup(type: "user")` to find co-worker emails if needed. Ask the hiring manager who should interview if not obvious.

**Note:** Ashby API does NOT create Google Calendar events — that's why step 9b is separate and required.

#### 9d. Offer follow-up email
After creating the calendar event and Ashby schedule, ask the hiring manager:
> "Calendar invite created (not sent yet). Want me to also draft a confirmation email to {name}? Or just send the calendar invite?"

Options:
- **"send invite"** → send the calendar invite (update event with `sendUpdates: "all"`)
- **"draft email"** → create a Gmail draft in the existing thread confirming time + Zoom link
- **"both"** → send invite + draft email
- **"skip"** → don't send anything yet

#### 9e. Update WAAS stage (if candidate is in WAAS)
If the candidate has a WAAS `short_id`, update their WAAS pipeline status to reflect the scheduled screen:
```
candidate_status_update(short_id, state: "screen", pipeline_stage: "Screen")
```
This keeps WAAS in sync with Ashby. For tech screens, use `state: "interviewing"`, `pipeline_stage: "Interview"`.

#### 9g. Verify (MANDATORY)
DO NOT report success until you have re-queried ALL of the following to confirm each action took effect:
- Calendar: `calendar_get_events` for the time window — verify event appears with correct attendees
- Ashby: `interview_schedule_list` — verify schedule was created
- WAAS: `candidate_status_show` — verify stage was updated (if applicable)
- Drafts: `gmail_list_drafts` — verify draft exists (if created)
Report what you verified in the confirmation message.

## Tech Screen Scheduling

When scheduling tech screens (not phone screens), the process is different:

1. **Ask the hiring manager who to CC** — the YC engineer running the exercise. Never guess. Present: "Who should run the exercise? (e.g. Jason, Lauren, Interviewer A)"
2. **Check the engineer's calendar too** — the time slot must work for both the hiring manager and the engineer.
3. **Include the interview process doc** in the email: `<interview process doc URL from recruit-config/jobs/>`
4. **Use the Post-Screen Advancement template** from the playbook.
5. **Tech screens are 1 hour**, not 30 minutes.

## Email formatting rule

**No hard line wrapping.** Write each paragraph as a single continuous line — let the email client handle wrapping. Never insert `\n` within a paragraph. Only use newlines between paragraphs.

## Gmail & Calendar Rules

- **ASCII only in emails.** No em dashes, curly quotes, or unicode. Use `--` instead of em dashes and straight quotes. The plain-text Gmail MCP tool mangles unicode into garbage.
- **Phone screen calendar invites** use the title `<Candidate First Name> and Ryan` -- no "Phone Screen" label. Later stages (tech screen, onsite, partner) include the stage name.
- **Use `mcp__claude_ai_Gmail__gmail_create_draft`** (not `mcp__gmail-drafts__gmail_create_draft`) when creating email drafts. Set `contentType: "text/html"` and use `<p>` tags. The `gmail-drafts` tool only supports `text/plain` which hard-wraps lines at 78 characters, making emails look broken.
- **Never create calendar invites until the candidate confirms the time.** Send the outreach or scheduling email first, wait for the candidate to reply confirming, then create the calendar event. Sending an unsolicited calendar invite before confirmation is presumptuous and creates a bad impression.

## Rules

1. **Always use America/Los_Angeles timezone.** The hiring manager's timezone is in user.md.
2. **Verify the day of week.** Always compute what day a date falls on. Don't guess.
3. **Skip weekends and known blocked days.** Demo Day, batch events, full-day onsites → note as blocked.
4. **Show blocks >= 30 min only.** Don't show 15-minute gaps.
5. **Round to clean times.** Event ends at 10:17 → block starts at 10:15 or 10:30.
6. **Back-to-back is fine for phone screens.** 30 min each, no buffer needed.
7. **NEVER create events or drafts without explicit approval.**
8. **Check candidate email threads** for proposed times before suggesting — don't propose a time they already rejected.
9. **Check for email address changes** in threads before creating events.
10. **Flag candidates who haven't replied.** If the hiring manager proposed a time and the candidate hasn't responded, note "waiting for reply."
11. **Always ask the hiring manager who to CC for tech screens.** Never guess the engineer.
12. **Stagger times** when scheduling multiple candidates — don't double-book.
