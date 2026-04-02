---
name: pipeline
description: |
  Show the full recruiting pipeline with priority actions at the top. Leads with 🔴/🟡 urgency
  tiers (what needs the hiring manager right now), then shows the full roster by stage. Cross-references
  Ashby, Gmail, Calendar, and Drafts. Use when the user asks "pipeline", "show me everyone",
  "full status", "what's the pipeline look like", or wants a complete view.
allowed-tools:
  - Bash
  - Read
  - mcp__google-workspace__gmail_query_emails
  - mcp__google-workspace__gmail_get_email
  - mcp__google-workspace__calendar_get_events
  - mcp__gmail-drafts__gmail_list_drafts
  - mcp__ashby__job_list
  - mcp__ashby__candidate_search
  - mcp__ashby__candidate_info
  - mcp__ashby__application_info
  - mcp__ashby__application_list
  - mcp__ashby__application_feedback_list
  - mcp__ashby__interview_schedule_list
  - mcp__waas__applicant_list
  - mcp__waas__candidate_batch
  - mcp__waas__health_check
---

> **Note:** Read `recruit-config/user.md` for the hiring manager's name, email, and preferences. Do not hardcode any user-specific values.


**Config**: Read these files before executing — they contain all IDs, tokens, templates, and rules:
- `recruit-config/user.md` — hiring manager email, Zoom link, office, Slack IDs, noise senders
- `recruit-config/ashby.md` — source IDs, archive reasons, email style, draft tool rules
- `recruit-config/waas.md` — WAAS pipeline stages, job IDs, MCP tools` — Slack/Gmail config, source IDs, archive reasons
- `recruit-config/CLAUDE.local.md` — Slack bot token (gitignored)
- `recruit-config/jobs/product-engineer.md` — job ID, stage IDs, candidate bar, outreach template, tone rules

# Pipeline Status

Show the complete recruiting pipeline — every candidate with activity in the last 21 days.

## Procedure

### Step 0: WAAS health check

Run `health_check()` first. If the response is `WAAS_API: expired` or `WAAS_API: error`, note "WAAS API unavailable — showing Ashby + Gmail data only" in the summary header and skip WAAS-specific steps.

### Step 1: Gather all active candidates

Pull candidates from multiple sources to build the full list:

1. **Ashby active applications**: First call `job_list()` to get all open jobs, then `application_list(status: "Active")` for each job — this is the primary source of truth for who's in the pipeline. Do NOT hardcode a single jobId; The hiring manager hires for multiple roles (Product Engineer, Investment Associate, Design Engineer, Recruiter, etc.).

2. **WAAS applicants**: Use the WAAS MCP to pull all active applicants:
```
mcp__waas__applicant_list(compact: true)
```
**Always use `compact: true` for pipeline scans.** This returns slim payloads (~60% smaller) that fit within tool result limits. Full profiles are available via `candidate_show` for deep dives.

This returns candidates who applied via Work at a Startup. WAAS applicants may or may not be in Ashby — the API is the source of truth for the WAAS pipeline.

   When you need richer profile data for multiple WAAS candidates, use `candidate_batch(short_ids)` (max 25 per call) instead of individual `candidate_show` calls.

3. **Gmail inbox**: `gmail_query_emails(query: "newer_than:21d (from:workatastartup OR from:jackandjill OR subject:'Reaching out from' OR subject:'Introduction:' OR subject:'Intro:' OR subject:'Follow up from YC')")` — catch candidates who are in email but may not be in Ashby yet.

4. **Deduplicate** by candidate name/email across all sources.

### Step 2: Run candidate-status for each

For each candidate found, run the full pre-flight checklist from the `/candidate-status` skill:
- Ashby search + application info
- Gmail inbox + sent (14d)
- Gmail drafts
- Calendar (4 weeks) with responseStatus

### GATE — Do not present results until this is complete

For EVERY candidate you will show, verify you have queried ALL of:
1. Ashby stage + application info
2. Gmail inbox (14d) — latest email from candidate
3. Gmail sent (14d) — the hiring manager's last message to candidate
4. Gmail drafts — any pending draft for this candidate
5. Calendar (4 weeks) — upcoming events + responseStatus
6. WAAS status (if candidate has short_id)
7. Ashby feedback — for ALL Tech Screen+ candidates (count them, query all N)

If you have N candidates to present, you must have made N × 7 source checks (minus any that don't apply). If you checked fewer, go back and query the missing ones.

Your output MUST start with a "Sources checked" line listing what you queried with counts. If any source was skipped, state which and why.

### Step 3: Present the full table

Group by Ashby stage (highest stage first):

```
## Pipeline (as of {date})
Checked: Ashby ({N} active across {N} jobs), Gmail inbox (21d), Gmail sent (14d), Drafts, Calendar (4 weeks), WAAS ({N} applicants), Feedback ({N}/{N} Tech Screen+ candidates)
Skipped: [none / list with reason]

### 🔴 URGENT — Action needed now
Candidates where:
- Candidate replied 3+ days ago and the hiring manager hasn't responded
- Invite sent but NOT accepted and event is within 48 hours
- Candidate asked a question that's blocking scheduling
- Post-interview decision overdue

| # | Candidate | Background | Stage | Waiting since | Action Needed |

### 🟡 NEEDS ACTION — Reply or decide
Candidates where:
- Candidate proposed times or confirmed — needs calendar invite
- Post-screen follow-up needed (advancement or rejection)
- Candidate emailed with a question
- Stale 7+ days — re-engage or close out

| # | Candidate | Background | Stage | Waiting since | Action Needed |

---

### Full Pipeline by Stage

### Offer
| # | Candidate | Background | Source | Status |

### Onsite
| # | Candidate | Background | Source | Status |

### Partner Interview
| # | Candidate | Background | Source | Status |

### Technical Screen
| # | Candidate | Background | Source | Status |

### Phone Screen
| # | Candidate | Background | Source | Status |

### Reached Out
| # | Candidate | Background | Source | Status |

### In Consideration
| # | Candidate | Background | Source | Status |

### WAAS — In Progress (messaged)
| # | Candidate | Background | WAAS State | Applied For | Last Messaged | Status |

### WAAS — Needs Response (not messaged)
| # | Candidate | Background | WAAS State | Applied For | Applied At | Status |
```

**Priority section goes first** — the 🔴/🟡 tiers are the quick "what do I need to do" view. The full pipeline below gives the complete roster.

**Show ALL candidates in the full pipeline** — scheduled, waiting, on hold, everything. Candidates that appear in 🔴/🟡 also appear in the stage tables (with a note like "⚠️ needs action").

**Filter rules for priority tiers:**
- **Exclude** from 🔴/🟡: Scheduled and accepted (they're coming), ball in candidate's court (the hiring manager's last email was more recent), archived/rejected
- **Include** in 🔴: 3+ days waiting on the hiring manager, event within 48 hours not accepted, blocking questions
- **Include** in 🟡: Needs scheduling, needs decision, needs follow-up, stale 7+ days

Within each tier, order by: days waiting (longest first) → pipeline stage (later stages first) → candidate quality signal.

Use all rules from the playbook at `.coding-agent-plans/recruiting-assistant.md` — especially the pre-flight checklist, responseStatus checking, and "what to say when you don't know" guidelines.
