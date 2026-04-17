---
name: review-applicants
description: |
  Review all new inbound applicants across Ashby, WAAS, and inbox. Shows candidates
  at Application Review who the bot didn't catch, WAAS applicants needing response,
  and new inbox applications — all in one view with recommendations. Use when the user
  says "show me applicants", "review inbound", "who applied", "new candidates".
allowed-tools:
  - Bash
  - Read
  - mcp__google-workspace__gmail_query_emails
  - mcp__google-workspace__gmail_get_email
  - mcp__ashby__candidate_search
  - mcp__ashby__candidate_info
  - mcp__ashby__application_info
  - mcp__ashby__application_list
  - mcp__ashby__application_change_stage
  - mcp__ashby__archive_reason_list
  - mcp__waas__applicant_list
  - mcp__waas__candidate_show
  - mcp__waas__candidate_status_show
  - mcp__waas__candidate_status_update
  - mcp__waas__candidate_message_send
  - mcp__waas__candidate_messages_list
  - mcp__waas__candidate_batch
  - mcp__waas__candidate_notes_list
  - mcp__waas__candidate_note_create
  - mcp__waas__health_check
---

> **Note:** Read `recruit-config/user.md` for the hiring manager's name, email, and preferences. Do not hardcode any user-specific values.


**Config**: Read these files before executing — they contain all IDs, tokens, templates, and rules:
- `recruit-config/user.md` — hiring manager email, Zoom link, office, Slack IDs, noise senders
- `recruit-config/ashby.md` — source IDs, archive reasons, email style, draft tool rules
- `recruit-config/waas.md` — WAAS pipeline stages, job IDs, MCP tools` — Slack/Gmail config, source IDs, archive reasons
- `recruit-config/CLAUDE.local.md` — Slack bot token (gitignored)
- `recruit-config/jobs/product-engineer.md` — job ID, stage IDs, candidate bar, outreach template, tone rules

# Review Applicants

Show all new inbound candidates across every source in one unified view. The "who applied?" skill.

## Sources

Pull from all three:

1. **Ashby** — active applications at Application Review (bot didn't catch)
2. **WAAS** — applicants needing response (via API or inbox emails)
3. **Inbox** — WAAS application emails, Ashby notifications, J&J intros, cold inbound

Deduplicate across sources. Each candidate appears once.

## Procedure

### Step 0: WAAS health check

Run `health_check()` first. If the response is `WAAS_API: expired` or `WAAS_API: error`, note "WAAS API unavailable — showing Ashby + inbox data only" in the summary header and skip WAAS-specific steps.

### Step 1: Pull Ashby Application Review candidates

```
application_list(status: "Active", jobId: "<job ID from recruit-config/jobs/ for this role>", limit: 100)
```

Filter to candidates at **Application Review** stage. These are candidates the auto-archive bot didn't catch — the hiring manager needs to review them.

Also check other jobs (Investment Associate, Design Engineer, etc.) if the hiring manager has multiple open roles.

### Step 2: Pull WAAS applicants

Query the WAAS API for applicants needing response. This is the source of truth for WAAS applicants — more reliable than inbox emails.

```
mcp__waas__applicant_list(needs_response: true, compact: true)
```

**Always use `compact: true` for applicant review.** This returns ~60% smaller payloads that fit within tool result limits. Use `candidate_show(short_id)` for full profiles when the hiring manager wants to dig into a specific candidate.

Also pull all active applicants to get the full count:
```
mcp__waas__applicant_list(compact: true)
```

Results are ordered by `applied_at` descending (newest first).

When you need richer profile data for multiple candidates, use `candidate_batch(short_ids)` (max 25 per call) instead of individual `candidate_show` calls.

**Job filtering**: If the user asks to focus on a specific role (e.g., "just PE applicants"), filter by `job_id`. Job IDs are discovered from the `applied_jobs` array in the initial unfiltered response — extract unique job id/title pairs, then use the ID for follow-up queries:
```
mcp__waas__applicant_list(needs_response: true, job_id: 41302)
```

**Date filtering**: To scope to recent applicants:
```
mcp__waas__applicant_list(needs_response: true, since: "2026-03-15T00:00:00Z")
```

**Combining filters**:
```
mcp__waas__applicant_list(needs_response: true, job_id: 41302, since: "2026-03-15T00:00:00Z")
```

The API returns full candidate profiles: name, email, role, experience, location, applied_at, applied_jobs, state, company_messaged_at, profile_url, us_authorized, us_visa_sponsorship, positions, educations.

For each WAAS applicant, use these fields for tiering:
- `role` + `experience` → seniority signal
- `location` + `us_authorized` + `us_visa_sponsorship` → work auth / location signal
- `applied_jobs[].title` → which role they applied for
- `positions` → work history (company names, titles, tenure)
- `educations` → schools and degrees
- `profile_url` → link to their full WAAS profile on Bookface
- `applied_at` → how long they've been waiting

If the API call fails (no creds, token expired), fall back to inbox:
```
gmail_query_emails(user_id: "<hiring manager email from user.md>", query: "in:inbox from:workatastartup@ycombinator.com newer_than:7d", max_results: 20)
```

### Step 3: Pull inbox applications

```
gmail_query_emails(user_id: "<hiring manager email from user.md>", query: "in:inbox (from:workatastartup@ycombinator.com OR from:notifications@ashbyhq.com OR from:jill@jackandjill.ai) newer_than:7d", max_results: 20)
```

Filter to application-related emails only (not stage changes, feedback reminders, etc.).

### Step 4: Merge and deduplicate

Combine all candidates from Steps 1-3. If a candidate appears in both Ashby and inbox, show them once with both sources noted.

### Step 5: Research each candidate

For each candidate, present:
- **Name**
- **Role @ Company** (from Ashby)
- **School** (from Ashby)
- **Location** (from Ashby)
- **Source** (Applied/Ashby, WAAS score, J&J intro, cold inbound)
- **Work auth** (from application form answers if available — check the Ashby notification email)
- **Flags** — needs visa, multiple applications, intern-level, non-technical role

### Step 6: Tier the candidates

Group into three tiers based on observable signals. **Do NOT evaluate subjective fit** — just flag objective signals.

Read the **candidate bar** from the job playbook (`recruit-config/jobs/`) for the role the candidate applied to. Use the hard requirements, auto-archive rules, strong signals, and exception candidates defined there to tier each candidate.

#### Auto-archive (clear dealbreakers)
Apply the auto-archive criteria from the job playbook. Additionally, flag for hiring manager review (do not auto-archive) candidates where work authorization or location is unclear — absence of information is not a dealbreaker.

#### Worth reviewing (mixed signals, hiring manager decides)
Apply the job playbook's criteria. Candidates who partially match the hard requirements or have mixed signals belong here for the hiring manager to decide.

#### Strong signal (worth a call)
Apply the strong signals and exception candidate criteria from the job playbook. WAAS score 80+ is an additional positive signal regardless of role.

### GATE — Do not present results until this is complete

Verify you have completed ALL of:
1. Ashby Application Review pulled — ALL open jobs, not just Product Engineer
2. WAAS applicants pulled (needs_response + all active), or noted as unavailable
3. Inbox applications scanned and cross-referenced
4. All candidates deduplicated across sources
5. Each candidate has: name, background, location, source, work auth (where available)

Your output MUST start with a "Sources checked" line. If any source was skipped, state which and why.

### Step 7: Present

```
## Applicant Review (as of {date})
Sources: Ashby ({count} at App Review) + WAAS ({count} needs response, by job: PE={n}, IA={n}, Ops={n}) + Inbox ({count} new)

### Auto-archive ({count})
| # | Name | Background | Location | Why |
|---|------|-----------|----------|-----|
| 1 | Candidate A | QA Analyst @ TechCorp | Remote | Non-engineering background |
| 2 | Candidate B | SWE Intern @ BigCo | Remote, no work auth | Intern, no work authorization |

### Worth reviewing ({count})
| # | Name | Background | Location | Source | Signal |
|---|------|-----------|----------|--------|--------|
| 3 | Candidate C | Assoc Consultant @ Enterprise Co | East Coast | Ashby | Enterprise Co, entry-level |
| 4 | Candidate D | Founder @ Stealth | Maryland | Ashby | Founder signal, not local |

### Strong signal ({count})
| # | Name | Background | Location | Source | Signal |
|---|------|-----------|----------|--------|--------|
| 5 | Candidate E | SWE @ Tech Corp | Bay Area | Ashby | Strong employer, relevant stack |
| 6 | Candidate F | Full-stack eng | SF | WAAS (82) | High score, sent message |

**Actions:**
- "archive 1-2" → archive the auto-archive tier in Ashby
- "pass on 3" → archive specific candidate
- "reach out to 6" → draft outreach email
- "tell me more about 5" → pull full profile
```

### Step 8: Wait for the hiring manager's decisions

The hiring manager will go through the list and say things like:
- "archive all the auto-archives" → batch archive in Ashby + WAAS
- "pass on 3 and 4" → archive specific candidates
- "reach out to 6" → draft and send message via WAAS
- "tell me more about 5" → use `/candidate-brief`

### Step 9: Execute WAAS actions

When the hiring manager approves an action on a WAAS candidate, use the WAAS API to execute it:

**Archive/update state**:
```
mcp__waas__candidate_status_update(short_id: "{short_id}", state: "archived", archive_reason: "not_qualified")
```

Valid states: `reviewing`, `shortlisted`, `screen`, `interviewing`, `offer`, `archived`.
Archive reasons: `not_qualified`, `team_fit`, `might_consider_later`, `turned_down_offer`, `hired`, `other`.

**Send message**:
```
mcp__waas__candidate_message_send(short_id: "{short_id}", message: "Hi! Thanks for applying...")
```

**Read message history** before replying:
```
mcp__waas__candidate_messages_list(short_id: "{short_id}")
```

**Batch archive**: Call `mcp__waas__candidate_status_update` for each short_id.

**Confirm before acting**: Always show the hiring manager exactly what you're about to do before executing write actions. Present the action, wait for approval, then execute.

## Gmail & Calendar Rules

- **ASCII only in emails.** No em dashes, curly quotes, or unicode. Use `--` instead of em dashes and straight quotes. The plain-text Gmail MCP tool mangles unicode into garbage.
- **Phone screen calendar invites** use the title `<Candidate First Name> and Ryan` -- no "Phone Screen" label. Later stages (tech screen, onsite, partner) include the stage name.
- **Use `mcp__claude_ai_Gmail__gmail_create_draft`** (not `mcp__gmail-drafts__gmail_create_draft`) when creating email drafts. Set `contentType: "text/html"` and use `<p>` tags. The `gmail-drafts` tool only supports `text/plain` which hard-wraps lines at 78 characters, making emails look broken.

## Rules

1. **Present facts, not opinions on fit.** Tier by observable signals (location, company, school, work auth), not subjective quality judgments.
2. **Always check work auth** from Ashby application form answers when available.
3. **Flag location clearly.** Apply the location and work authorization criteria from the job playbook (`recruit-config/jobs/`). Flag unclear cases for hiring manager review rather than auto-archiving.
4. **Flag multiple applications.** If someone has 3+ applicationIds, note it — could be persistent interest or spam.
5. **WAAS scores are directional.** 80+ is strong signal, 50-79 is mixed, below 50 is weak.
6. **Don't skip any active Application Review candidate.** Every one must appear in the output.
7. **Deduplicate across sources.** Don't show someone twice because they're in both Ashby and inbox.
8. **Batch actions are fine.** "Archive 1-5" should archive all 5 in one go.
