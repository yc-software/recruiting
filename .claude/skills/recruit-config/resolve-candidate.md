# Unified Candidate Resolver

Shared procedure for building a complete candidate view from all sources.
Other skills should reference this file when they need to look up a candidate.

## Usage

When a skill needs to resolve a candidate (by name, short_id, or email), follow this procedure.
Run all sources in parallel where possible.

## Step 1: Identify the candidate

Start with whatever you have:
- **Name** → search Ashby + WAAS
- **short_id** → query WAAS API directly
- **Email** → search Ashby + Gmail

## Step 2: Query all sources

### WAAS Profile (if short_id known)
```
mcp__waas__candidate_show(short_id: "{short_id}")
```
Returns: name, email, role, experience, location, us_authorized, us_visa_sponsorship,
positions, educations, github_url, linkedin_url, profile_url.

### WAAS Status (if short_id known)
```
mcp__waas__candidate_status_show(short_id: "{short_id}")
```
Returns: state, state_changed_at, archive_reason, pipeline_stage, contacted_at, last_messaged_at.

### WAAS Messages (if short_id known)
```
mcp__waas__candidate_messages_list(short_id: "{short_id}")
```
Returns: conversation history with sender_name, from_candidate flag, message text, timestamps.

### Ashby
```
candidate_search(name: "Candidate Name")
→ candidate_info(id: candidate_id)
→ application_info(applicationId: app_id)
```
Returns: Ashby stage, application status, interview schedule, source, custom fields.

### Ashby Feedback (if applicationId known)
```
application_feedback_list(applicationId: app_id)
```
Returns: all submitted feedback — reviewer name, score (1-4: Strong No → Strong Yes), written feedback, interview type, submission date. Check this to determine if feedback has been submitted and what the scores are.

### Gmail
```
gmail_query_emails(user_id: "<hiring manager email from user.md>", query: "newer_than:14d {candidate name or email}")
```
Returns: email threads, last contact, who replied last, draft status.

### Calendar
```
calendar_get_events(calendarId: "primary", timeMin: 2_weeks_ago, timeMax: 4_weeks_ahead, q: "candidate name")
```
Returns: scheduled interviews, responseStatus (accepted/needsAction/declined).

### Gmail Drafts
```
gmail_list_drafts(user_id: "<hiring manager email from user.md>", query: "{candidate name or email}")
```
Returns: unsent draft replies.

## Step 3: Merge into unified view

Combine all results into this structure:

```
## {Name}

### Identity
- **Email**: {from WAAS or Ashby}
- **Location**: {from WAAS}
- **Role**: {role} | {experience} years
- **Work Auth**: {us_authorized} | Visa: {us_visa_sponsorship}
- **LinkedIn**: {url}
- **WAAS Profile**: {profile_url}

### Pipeline Status
- **WAAS State**: {state} (since {state_changed_at})
- **Ashby Stage**: {stage} ({status})
- **Applied**: {applied_at} for {job_titles}

### Feedback
- **{Reviewer name}**: {score}/4 ({Strong No/No/Yes/Strong Yes}) — {one-line summary of written feedback}
- **{Reviewer name}**: {score}/4 — {one-line summary}
(If no feedback submitted, note "No feedback submitted")

### Communication
- **Last email**: {date} — {who sent it, subject}
- **Last WAAS message**: {date} — {from_candidate or from_company}
- **Company messaged at**: {date or "never"}
- **Draft pending**: {yes/no}

### Upcoming
- **Next interview**: {date, type, responseStatus}

### Background
- **Current**: {title} @ {company}
- **Previous**: {positions summary}
- **Education**: {school, degree}
```

## Step 4: Handle partial failures

If any source fails (API down, no results), include what you have and note the gap:
- "WAAS API: not configured" or "WAAS API: token expired"
- "Ashby: no candidate found"
- "Gmail: no threads found"
- "Calendar: no events found"

Never fail entirely because one source is down. Present what you have.

## Step 5: Deduplication rules

- If a candidate appears in both WAAS and Ashby, prefer WAAS for profile data (more detailed) and Ashby for pipeline stage.
- If email differs between sources, note both.
- WAAS `state` and Ashby `stage` are independent — show both.
