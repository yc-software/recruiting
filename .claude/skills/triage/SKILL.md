---
name: triage
description: |
  Triage everything that needs the hiring manager's attention — inbox, Ashby pipeline, WAAS applicants,
  and other sources. The "start of day" view. Categorizes by activity type, surfaces what needs
  action, and filters out noise. Use when the user says "triage", "what do I need to deal
  with", "what's going on", "what's in my inbox", or any variation.
allowed-tools:
  - Bash
  - Read
  - mcp__google-workspace__gmail_query_emails
  - mcp__google-workspace__gmail_get_email
  - mcp__ashby__candidate_search
  - mcp__ashby__candidate_info
  - mcp__ashby__application_info
  - mcp__ashby__application_list
  - mcp__ashby__application_feedback_list
  - mcp__waas__applicant_list
  - mcp__waas__health_check
---

> **Note:** Read `recruit-config/user.md` for the hiring manager's name, email, and preferences. Do not hardcode any user-specific values.


**Config**: Read these files before executing — they contain all IDs, tokens, templates, and rules:

- `recruit-config/user.md` — hiring manager email, Zoom link, office, Slack IDs, noise senders
- `recruit-config/ashby.md` — source IDs, archive reasons, email style, draft tool rules
- `recruit-config/waas.md` — WAAS pipeline stages, job IDs, MCP tools` — Slack/Gmail config, source IDs, archive reasons
- `recruit-config/CLAUDE.local.md` — Slack bot token (gitignored)
- `recruit-config/jobs/product-engineer.md` — job ID, stage IDs, candidate bar, outreach template, tone rules

# Triage

Scan all sources — inbox, Ashby, WAAS applicants API, and others — to show the hiring manager what needs attention. The "start of day" view.

## Procedure

### Step 0: WAAS health check

Run `health_check()` first. If the response is `WAAS_API: expired` or `WAAS_API: error`, note "WAAS API unavailable — showing email-based WAAS data only" in the summary header and skip Step 3b.

### Step 1: Pull inbox

```
gmail_query_emails(user_id: "<hiring manager email from user.md>", query: "in:inbox newer_than:7d", max_results: 50)
```

### Step 2: Categorize each email

For each email, classify into one of these categories based on the signals below:

#### Hiring for YC

| Category                           | Signals                                                                              | Examples                                                         |
| ---------------------------------- | ------------------------------------------------------------------------------------ | ---------------------------------------------------------------- |
| **Scheduling — needs reply**       | Candidate proposed times, confirmed a time, asked "is this virtual?", rescheduling   | "I can do Thursday 2-4pm", "yes monday 3pm works"                |
| **Pipeline decisions**             | Ashby feedback notifications, post-interview thank-yous, candidate asking for update | "[Ashby] Feedback submitted", "Can you let me know if I passed?" |
| **Candidate Q&A**                  | Candidate asking questions about role, process, work auth, culture                   | "Is this remote?", "How often do engineers come to office?"      |
| **New applications (inbound)**     | See Step 2b for the full taxonomy of inbound sources                                 | WAAS apps, Ashby apps, J&J intros, WAAS digests, cold inbound    |
| **Sourcing — replies to outreach** | Replies to "Reaching out from YC" or "Follow up from YC" threads                     | "Thanks for reaching out, I'd love to learn more"                |

#### Non-Hiring for YC

| Category                        | Signals                                                                                                              |
| ------------------------------- | -------------------------------------------------------------------------------------------------------------------- |
| **Helping YC companies hire**   | FROM founders asking about WAAS, JDs, recruiting tools                                                               |
| **Vendor management**           | FROM jackandjill.ai, dover.com, coastalrecruiting.co, ashbyhq.com                                                    |
| **Building WAAS (product/eng)** | GitLab MRs, Mode reports, API specs                                                                                  |
| **Noise**                       | DoorDash, Yelp, Brex, Partiful, Vercel, TrueUp, Zapier, Handshake, self-emails, calendar accepts/declines from rooms |

### Step 2b: Classify inbound candidate sources

New candidates arrive from multiple sources. Each looks different but they all go into "New applications (inbound)" unless they're auto-archived.

#### WAAS Applications (if WAAS is configured)

The following email patterns are specific to [Work at a Startup](https://www.workatastartup.com). Skip this section if WAAS is not configured.

- **From**: `Work at a Startup <workatastartup@ycombinator.com>`
- **Subject pattern**: `{Name} ({score}) applied for {Job Title}`
- **Action**: Show in New Applications with name, score, and job title. Higher scores = stronger match.

#### WAAS Messages (if WAAS is configured)

- **From**: `Work at a Startup <workatastartup@ycombinator.com>`
- **Subject pattern**: `{Name} sent you a message`
- **Action**: Show in New Applications. These are candidates who actively reached out — higher intent signal.

#### WAAS AI Sourcer Alerts (if WAAS is configured)

- **From**: `Y Combinator <workatastartup@ycombinator.com>`
- **Subject pattern**: `{N} new top candidates for {Job Title} joined yesterday`
- **Action**: Show in New Applications as a batch to review.

#### WAAS Digests (if WAAS is configured)

- **From**: `Y Combinator <workatastartup@ycombinator.com>`
- **Subject pattern**: `Top new candidates for {Company}`
- **Action**: Show in New Applications as a batch to review. Note the count.

#### Ashby Direct Application (NOT auto-archived)

- **From**: `Ashby Notification <notifications@ashbyhq.com>`
- **Subject pattern**: `[Ashby] {Name} applied to a job you're watching`
- **How to verify it's active**: `application_info(applicationId)` → `status: "Active"` AND `currentInterviewStage.title: "Application Review"`
- **Examples**: Candidate B (Tech Corp/State University), Candidate C (Enterprise Co/East Coast U)
- **Action**: Show in New Applications with candidate profile (name, position, company, school, location, Ashby link). Do NOT evaluate fit — present facts only.

#### Ashby Direct Application (auto-archived by bot)

- **From**: `Ashby Notification <notifications@ashbyhq.com>`
- **Subject pattern**: `[Ashby] Candidate {Name} has changed stage` — with "Ashby Bot 🤖" in snippet
- **How to verify**: `application_info(applicationId)` → `status: "Archived"` AND `archiveReason` present AND actorId is NOT the hiring manager's (see Ashby User ID in `user.md`)
- **Example**: Candidate A — "Lacks Work Authorization"
- **Action**: **Archive the notification.** The hiring manager trusts the auto-archive bot. Put in noise/archivable.

#### Sourcing vendor intros (configure vendor patterns in recruit-config/)

If you use a sourcing vendor (e.g., recruiting agency, sourcing service), their intro emails should be classified as **New applications (inbound)**. Their status update emails should be classified as **Vendor management**, unless they contain decision requests — flag those in Pipeline Decisions.

Configure your vendor's email patterns (from address, subject patterns) in the job playbook or a dedicated vendors config file.

#### Cold Inbound (direct email, no platform)

- **From**: personal email address
- **No standard subject pattern** — often referral-based
- **Example**: Candidate R — "Brooks: Referred by Candidate S"
- **Action**: Show in New Applications. Check Ashby to see if they exist. Flag as cold inbound.

#### Candidate replies to rejection emails

- **Subject pattern**: `Re: Update on Your YC Application` or similar rejection thread
- **Content**: Thank-you, well wishes, "I appreciated the conversation"
- **Example**: Candidate T — "Thanks so much for taking the time to review my application"
- **Action**: **Noise / archivable.** Candidate was already rejected. Their gracious reply doesn't require a response. Check Ashby to confirm they're Archived before archiving the email.

#### Other Ashby Notifications (not applications)

- Feedback submitted → check Ashby stage first:
  - If candidate is at **Offer** or **Archived** → **Noise / archivable** (decision already made, notification is informational)
  - If candidate is still active in pipeline → **Pipeline decisions** (feedback needs to be reviewed for a pending decision)
- RSVP decline → **Pipeline decisions**
- Stage change (not auto-archive, not by the hiring manager) → **Pipeline decisions**
- Stage change (by the hiring manager) → **Noise / archivable** (the hiring manager already knows, they did it)
- Feedback reminders → **Noise / archivable**

### Step 3: Scan Ashby pipeline for candidates needing action

Pull ALL active applications from Ashby — not just later stages.

```
application_list(status: "Active", jobId: "<job ID from recruit-config/jobs/ for this role>", limit: 100)
```

#### Application Review candidates (new applications)

For each candidate at **Application Review**, check:

- **Is there an Ashby notification in the inbox?** If not (notification was already archived or never received), the candidate could be invisible to the hiring manager. Surface them.
- **Was the candidate auto-archived by the bot?** If status is still Active at Application Review, the bot did NOT catch them — the hiring manager needs to review.
- **Cross-reference with inbox Step 2b.** If the candidate already appeared as a "New application" from the inbox scan, don't double-count. But if they ONLY exist in Ashby (no inbox email), they must appear in the Ashby section.

**Key rule: Every active Application Review candidate that the bot didn't archive must appear somewhere in the triage output** — either under "Inbox — New applications" (if there's an email) or under "Ashby Pipeline — New applications (no inbox email)". Never let a new applicant be invisible.

#### Later-stage candidates (Tech Screen+)

For each candidate at **Technical Screen, Partner Interview, Onsite, or Offer** stage, check:

- **Has feedback been submitted?** Use `application_feedback_list(applicationId)` to check directly. Shows scores (1-4), reviewer name, and written feedback. If no feedback exists, the hiring manager may need to submit it or follow up with the interviewer.
- **Is the candidate waiting for a decision?** If feedback exists (especially multiple reviewers), and the last stage change was 3+ days ago with no email sent, flag it — decision is overdue.
- **Does the candidate need to be moved?** If the hiring manager has already sent a rejection or advancement email but hasn't updated Ashby, flag the mismatch.

Cross-reference with Gmail sent mail to see if the hiring manager already took action outside Ashby.

#### All stages

Classify into:
| Category | Signals |
|----------|---------|
| **New applications (Ashby only)** | At Application Review, active, NOT auto-archived, no corresponding inbox email — the hiring manager hasn't seen them |
| **Needs decision** | At Tech Screen/Onsite/Partner Interview, feedback exists but no advancement or rejection sent |
| **Needs feedback** | At Tech Screen/Onsite, interview happened (check calendar for past events) but no feedback in Ashby notes |
| **Ashby out of sync** | The hiring manager already sent rejection/advancement email but Ashby stage hasn't been updated |
| **Stale** | Candidate at any stage with no activity (email or stage change) in 7+ days |

### Step 3b: Pull WAAS applicants (Work at a Startup)

Query the WAAS API for applicants who applied to the hiring manager's company's jobs. This gives a direct, real-time view of the WAAS pipeline — more reliable than parsing email notifications.

```
mcp__waas__applicant_list(needs_response: true, compact: true)
```

**Always use `compact: true` for triage.** This returns ~60% smaller payloads that fit within tool result limits. Full profiles are available via `candidate_show` for deep dives.

This returns applicants the company hasn't messaged yet. For each applicant, you get:

- `short_id` — use with `/v1/candidates/{short_id}` for full profile, messages, notes
- `name`, `email`, `role`, `experience`, `location`
- `state` — reviewing, shortlisted, screen, interviewing, offer
- `applied_at` — when they applied
- `applied_jobs` — which job(s) they applied to (id + title)
- `company_messaged_at` — null means no one from the company has responded
- `profile_url` — direct link to their WAAS profile on Bookface

Also pull all active (non-archived) applicants to get the full picture:

```
mcp__waas__applicant_list(compact: true)
```

Classify WAAS applicants into:
| Category | Signals |
|----------|---------|
| **Needs response (WAAS)** | `company_messaged_at` is null — applied but no one replied |
| **In progress (WAAS)** | `company_messaged_at` is set — conversation started, check state for next action |
| **New today (WAAS)** | `applied_at` is today — just came in |

**Deduplication**: WAAS applicants may also appear in the inbox (as WAAS application emails) and in Ashby (if they were uploaded). When presenting, note the overlap but don't double-count. The WAAS API is the source of truth for application status.

**Available WAAS API actions** (for follow-up after triage):

- `GET /v1/candidates/{short_id}` — full candidate profile
- `GET /v1/candidates/{short_id}/messages` — conversation history
- `POST /v1/candidates/{short_id}/messages` — send a message (requires `waas:messages:manage` scope)
- `GET /v1/candidates/{short_id}/status` — pipeline state
- `PUT /v1/candidates/{short_id}/status` — update state (requires `waas:stages:manage` scope)

If the WAAS API call fails (no credentials, token expired), skip this step gracefully and note "WAAS API not configured — showing email-based WAAS data only" in the summary.

### GATE — Do not present results until this is complete

Before presenting ANY output, verify you have completed ALL of the following:

1. **Inbox scanned** — all emails categorized, personal-address senders checked in Ashby
2. **Ashby pipeline pulled** — ALL open jobs, not just Product Engineer
3. **Feedback checked for ALL Tech Screen+ candidates** — count them, query all N, verify you got N results
4. **WAAS applicants pulled** — needs_response + all active (or noted as unavailable)
5. **Calendar checked** — upcoming events for the next 2 weeks, cross-referenced with pipeline candidates
6. **Gmail sent checked** — for every candidate surfaced in scheduling/pipeline decisions, verify the hiring manager's last sent message
7. **Gmail drafts checked** — verify whether drafts exist for any active candidate threads

If any source was not checked, you MUST note it in the output header as "Skipped: [source] — [reason]". Presenting data without checking all sources creates false confidence and wastes the hiring manager's time.

### Step 4: Present the summary

This is a **summary only** — counts and top items per category. Do NOT show full candidate tables, sub-tables, Ashby stages, or detailed status here. Every category gets the same treatment: count + top items in one line. Detail comes when the hiring manager picks a category to focus on.

```
## Triage (as of {date})
Checked: Ashby ({N} active across {N} jobs), Gmail inbox (7d, {N} emails), Gmail sent (14d), Drafts, Calendar (2 weeks), WAAS ({N} active, {N} needs response), Feedback ({N}/{N} Tech Screen+ candidates)
Skipped: [none / list with reason]

### New Applications (all sources combined)
| # | Category | Count | Top items |
|---|----------|-------|-----------|
| 1 | New applications | 14 | Candidate F (WAAS 82), Candidate B (Ashby, Tech Corp), Candidate C (Ashby, Enterprise Co), Candidate D (Ashby), Candidate E (Ashby), ... |
| 2 | WAAS needs response (never messaged) | 8 | Candidate G (eng, applied 3/15), Candidate H (eng, applied 3/14) |
| 3 | WAAS new today | 2 | Candidate I (design, applied today) |

New applications are merged from inbox (WAAS emails, Ashby notifications, J&J intros) + Ashby Application Review (bot didn't catch) + WAAS API (needs_response). Deduplicated — each candidate appears once with their source noted.

### Pipeline — Needs Attention
| # | Category | Count | Top items |
|---|----------|-------|-----------|
| 4 | Scheduling — needs reply | 1 | Candidate J (3rd follow-up) |
| 5 | Pipeline decisions | 5 | Candidate K (asking for update), Candidate L (feedback pending) |
| 6 | Candidate Q&A | 1 | Candidate M (post-onsite questions) |
| 7 | Needs decision (post-interview) | 2 | Candidate L (Tech Screen), Candidate K (Tech Screen) |
| 8 | Needs feedback | 1 | Candidate N (Tech Screen Mon, no notes yet) |
| 9 | Ashby out of sync | 2 | Candidate O (rejected but still Active), Candidate P (emailed but not archived) |
| 10 | Stale (7+ days no activity) | 3 | Candidate Q (Tech Screen, no reply since 3/14), ... |
| 11 | WAAS in progress | 12 | 3 reviewing, 5 shortlisted, 2 screen, 2 interviewing |

### Inbox — Non-Hiring
| # | Category | Count | Top items |
|---|----------|-------|-----------|
| 12 | Vendor management | 3 | Dover, Coastal x2, vendor contact/Dover |
| 13 | Building WAAS (eng) | 17 | GitLab MRs, failed pipeline |
| 14 | Noise / archivable | 16 | GitLab, calendar notifications, SpotHero, etc. |

What do you want to focus on?
```

### Step 5: Wait for the hiring manager to pick, then drill down

Don't dive into any category until the hiring manager says which one.

When the hiring manager picks a category, use the appropriate skill to show the detailed view:

| User picks          | What to do                                                                                                                                                                                       |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Pipeline decisions  | Show the full candidate table (standard format) for candidates with pending decisions. Cross-reference Ashby + email.                                                                            |
| Candidate Q&A       | Show the candidate's question and suggest a reply using `/reply`                                                                                                                                 |
| Scheduling          | Show the scheduling table, then ask for a calendar block via `/schedule`                                                                                                                         |
| New applications    | Show candidate profiles with Ashby links. Let the hiring manager review and decide engage/pass.                                                                                                                |
| WAAS needs response | Show a table of WAAS applicants who haven't been messaged. For each: name, role, experience, applied_at, job title, profile_url. Offer to draft outreach via the WAAS messaging API or `/reply`. |
| WAAS new today      | Show today's WAAS applicants with full profile details.                                                                                                                                          |
| WAAS in progress    | Show applicants grouped by state (reviewing, shortlisted, screen, etc.) with last_messaged_at.                                                                                                   |
| Needs decision      | Show candidates with pending post-interview decisions. Cross-reference feedback + email.                                                                                                         |
| Needs feedback      | Show which interviews happened and need the hiring manager's notes/feedback in Ashby.                                                                                                                          |
| Ashby out of sync   | Show mismatches and offer to fix (archive in Ashby, move stage, etc.)                                                                                                                            |
| Stale               | Show candidates with no activity and ask the hiring manager to advance, reject, or re-engage.                                                                                                                  |
| Vendor management   | List the vendor emails with snippets. Let the hiring manager decide what to respond to.                                                                                                                        |
| Noise / archivable  | Run `/inbox-archive` to present the list and let the hiring manager confirm bulk archive.                                                                                                                      |

## Rules

1. **Scan all inbox emails** — don't filter too aggressively in the Gmail query. Better to categorize a wider set than miss something.
2. **Show counts and top items** — The hiring manager needs to know the volume and what's most notable in each category, not every email.
3. **"Noise / archivable" is a real category.** Surface it so the hiring manager can bulk-archive.
4. **Order by priority** — scheduling first (people are waiting), then decisions, then Q&A, then inbound, then sourcing replies.
5. **New items since last check** — if the user ran this earlier in the session, highlight what's new.
6. **Check Ashby for EVERY email from a personal email address.** Before categorizing ANY email from a gmail, hotmail, yahoo, .edu, or other personal address — regardless of what the subject or content looks like — search Ashby by the sender's name. Emails from active candidates can look like casual/social messages ("Catch up?", "Portfolio", "Thanks for having me") but are actually pipeline-relevant. The email content is NOT a reliable signal for whether someone is a candidate. Only Ashby knows. If they're Active at any stage in any job → categorize based on their pipeline stage, not the email's surface appearance.
7. **Check ALL open jobs, not just Product Engineer.** The hiring manager hires for multiple roles (Product Engineer, Recruiter and Event Operations, Design Engineer, Investment Associate, etc.). A candidate may be active on a different job. Always check Ashby regardless of which job they applied to.
8. **Known noise senders that DON'T need Ashby checks:** GitLab (@mg.gitlab.com), Ashby Notification (notifications@ashbyhq.com — handle per Step 2b), Mode (noreply@modeanalytics.com), DoorDash, Yelp, Brex, Partiful, Vercel, TrueUp, Zapier, Handshake, SpotHero, Mezmo, The Athletic, GitHub (no-reply@github.com), YC Calendar (calendar@ycombinator.com), Zoom (no-reply@zoom.us), HR Conclave (Eventbrite). Everything else from a personal address → check Ashby.
9. **Double-check conversational emails in noise (internally — don't show your work).** After categorizing, silently review every email in the noise tier that looks like a back-and-forth between people (reply threads, "Re:" subjects, conversational snippets). For each, verify: Did you check Ashby? Is this a confirmed rejection reply? Is this a calendar notification vs. a real message? Could this be an active candidate? If any doubt, move it OUT of noise into the right category. Do this verification internally — don't present a "verification table" to the hiring manager. Just make sure the final triage is correct.
