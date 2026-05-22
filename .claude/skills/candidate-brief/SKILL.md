---
name: candidate-brief
description: |
  Generate a quick one-paragraph brief on a candidate by pulling from all available sources.
  Replaces opening LinkedIn, Ashby, and email tabs. Use when the user asks "who is this person",
  "tell me about X", "brief me on X", or is triaging candidates and needs context fast.
allowed-tools:
  - Bash
  - Read
  - mcp__ashby__candidate_search
  - mcp__ashby__candidate_info
  - mcp__ashby__application_info
  - mcp__ashby__candidate_list_notes
  - mcp__google-workspace__gmail_query_emails
  - mcp__google-workspace__gmail_get_email
  - mcp__waas__candidate_show
  - mcp__waas__candidate_status_show
  - mcp__waas__candidate_notes_list
---

> **OpenClaw note (read this first):** the `allowed-tools` list above uses Claude-Code MCP names. On OpenClaw, substitute: `gog --account ryan@ycombinator.com gmail/calendar/...` for Google Workspace, `curl` against `https://api.ashbyhq.com` (HTTP Basic with `/data/.ashby/api_key`, always set `User-Agent`) for Ashby, `curl` against `https://api.ycombinator.com` (Bearer token from `/data/.yc/waas-credentials.json`) for WAAS. Behavior the skill prescribes is the source of truth, not the literal tool names. Full mapping: `recruiting/.claude/skills/_meta/BEFORE-DRAFTING.md` section 5.

> **Before drafting any candidate-facing content, read `recruiting/.claude/skills/_meta/BEFORE-DRAFTING.md` first.** It tells you which voice/job/playbook files to load and the drift-check to run before showing a draft.

> **Note:** Read `recruit-config/user.md` for the hiring manager's name, email, and preferences. Do not hardcode any user-specific values.


# Candidate Brief

Generate a quick contextual brief on a candidate so the hiring manager can recall and calibrate without opening 5 tabs.

## Unified Resolver

Read `recruit-config/resolve-candidate.md` for the shared procedure to pull data from all 4 sources (WAAS MCP, Ashby, Gmail, Calendar). Use the WAAS data (positions, educations, role, experience) for richer background info than Ashby alone provides.

## Input

One or more candidate names:
- `/candidate-brief Jane Doe`
- `/candidate-brief John Smith, Candidate C`

## Procedure

For each candidate, pull from ALL sources in parallel:

### 1. Ashby profile
```
candidate_search(name) → position, company, school, socialLinks (LinkedIn), customFields (YC Bio Summary)
```

### 2. Ashby application
```
application_info(applicationId) → source, currentInterviewStage, applicationHistory (how long in each stage)
```

### 3. Ashby notes
```
candidate_list_notes(candidateId) → any notes from the hiring manager or interviewers
```

### 4. Email thread context
```
gmail_query_emails(from:candidate, newer_than:21d) → what they've said
gmail_query_emails(in:sent to:candidate, newer_than:21d) → what the hiring manager said to them
```
Look for: J&J intro descriptions, WAAS profile summaries, the hiring manager's outreach text (which often references specific background details), candidate's own words about what they're looking for.

### 5. WAAS notes (if short_id available)
```
candidate_notes_list(short_id) → internal notes from WAAS
```
Include any WAAS notes alongside Ashby notes for a complete picture.

### 6. LinkedIn URL
From Ashby socialLinks. If missing, check email threads for linkedin.com URLs.

## Output Format

For each candidate, present:

```
## <Name>
**<Position> @ <Company>** | <School> | <Location>
[LinkedIn](<url>) · [Ashby](<profile_url>)

<One paragraph brief — 3-5 sentences covering:>
1. Who they are professionally (current/recent role, what they built)
2. Why they're relevant to YC (finance + tech overlap, founder background, AI experience, etc.)
3. How they entered the pipeline (WAAS, J&J, direct outreach, applied) and how long ago
4. Any notable signal from email exchanges (enthusiasm level, competing offers, timeline pressure, questions asked)
5. Any interviewer feedback or notes if available

**Stage:** <Ashby stage> | **Source:** <how they entered> | **In pipeline since:** <date>
```

### GATE — Do not present results until this is complete

For EACH candidate, verify you have queried ALL sources listed in the Procedure (Ashby profile, Ashby application, Ashby notes, email threads, WAAS notes, LinkedIn). If a source returned nothing, say so in the output — do not silently skip it.

## Rules

1. **Pull from every source.** Don't present a brief based on just Ashby — check email threads for richer context.
2. **If Ashby has no bio**, synthesize one from the email thread. J&J intros are gold — they contain a full paragraph about the candidate.
3. **Include signal, not just facts.** "He said he has competing offers and wants to decide by end of month" is more useful than "Senior SWE at Big Tech Co."
4. **Flag unknowns.** If you can't find background info, say so: "Background unknown — no Ashby bio, no J&J intro, no WAAS profile. Only data is the hiring manager's outreach email."
5. **Keep it short.** This is for fast triage, not a dossier. One paragraph + metadata line.
