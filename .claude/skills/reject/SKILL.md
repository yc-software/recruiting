---
name: reject
description: |
  Draft a personalized rejection email for a candidate, with tone calibrated to how far
  they got in the process. Archives in Ashby after approval. Use when the user says "reject X",
  "pass on X", "not moving forward with X", or any variation of declining a candidate.
allowed-tools:
  - Bash
  - Read
  - mcp__google-workspace__gmail_query_emails
  - mcp__google-workspace__gmail_get_email
  - mcp__claude_ai_Gmail__gmail_create_draft
  - mcp__gmail-drafts__gmail_send_draft
  - mcp__gmail-drafts__gmail_list_drafts
  - mcp__google-workspace__gmail_archive
  - mcp__ashby__candidate_search
  - mcp__ashby__candidate_info
  - mcp__ashby__application_info
  - mcp__ashby__application_change_stage
  - mcp__ashby__archive_reason_list
  - mcp__waas__candidate_show
  - mcp__waas__candidate_status_show
  - mcp__waas__candidate_status_update
  - mcp__waas__candidate_messages_list
  - mcp__waas__candidate_message_send
---

> **Note:** Read `recruit-config/user.md` for the hiring manager's name, email, and preferences. Do not hardcode any user-specific values.


**Config**: Read these files before executing — they contain all IDs, tokens, templates, and rules:
- `recruit-config/user.md` — hiring manager email, Zoom link, office, Slack IDs, noise senders
- `recruit-config/ashby.md` — source IDs, archive reasons, email style, draft tool rules
- `recruit-config/waas.md` — WAAS pipeline stages, job IDs, MCP tools` — Slack/Gmail config, source IDs, archive reasons
- `recruit-config/CLAUDE.local.md` — Slack bot token (gitignored)
- `recruit-config/jobs/product-engineer.md` — job ID, stage IDs, candidate bar, outreach template, tone rules

# Reject Candidate — One-Sided Decision

Draft a personalized rejection email when **the hiring manager/YC decided not to proceed** — a one-sided decision. Uses "we're not going to proceed" language.

**For mutual partings** (both sides agreed it doesn't work — commute, timing, location), use `/reply` instead. Rejection language is wrong when the candidate also agreed. Same for candidates who took another offer — that's a congratulations, not a rejection.

## Input

- `/reject Jane Doe`
- `/reject Jane Doe — not a good match`
- "pass on Jane" or "not moving forward with Jane" (routed from `/recruit`)

## Procedure

### Step 1: Gather context

1. **Ashby**: `candidate_search(name)` → stage, application history, who interviewed them
2. **Gmail sent**: find the most recent thread with this candidate, get threadId
3. **Gmail sent**: scan for what the hiring manager said to them (exercise instructions, who they met, etc.)

From the application history, determine:
- How far they got (Phone Screen, Tech Screen, Onsite, Partner Interview)
- Who they met (interviewer names from calendar events or email CC's)
- How long they've been in the pipeline
- What exercise they did (build exercise, data exercise, partner chat)

### Step 2: Pick tone from the ladder

| How far they got | Tone | What to include |
|---|---|---|
| **Phone screen only** | Short, gracious | Thank them for the call. 3-4 sentences. |
| **Tech screen** | Medium, reference the exercise + interviewer by name | "Thanks for coming through the exercise with {interviewer}." Offer to help if genuine. |
| **Onsite / multi-round** | Warm, reference specific things about them or what they built | Acknowledge the time investment. Mention something specific. Offer concrete help (intros to YC companies). |
| **YC founder / strong candidate** | Warmest, offer real help | Reference their background specifically. Offer intros to YC companies hiring. "Would be lucky to have you." |

If the user provided a reason ("not a good match", "lacks backend experience"), weave it in gently if appropriate — but never be harsh or specific about what they did wrong. The hiring manager's rejections are always gracious.

### Step 3: Draft the rejection

Use the hiring manager's voice patterns from real rejections:

**Opening**: Always thank them for their time + reference what they did
- "Thanks for making time to go through the exercise with {interviewer}."
- "Thanks for coming in the other day and meeting with {interviewer}!"
- "Thanks for making time yesterday to come to YC HQ and meet more of the team."

**The no**: Soft, uses "we" not "I", never says "you failed"
- "We got a chance to catch up, and unfortunately we're not going to proceed with the process at this time."
- "We got a chance to discuss it, and I don't think we will be proceeding with the process at this moment."
- "We rounded up this morning, and unfortunately we're not going to proceed with the process at this time."

**The offer to help** (calibrated by stage):
- Phone screen: "Thanks again!" (no offer)
- Tech screen: "If I can be helpful as you're looking for your next role, don't hesitate to reach out."
- Onsite: "If there's anything we can do to be helpful for your next role, please let us know."
- Strong candidate: "There are a number of great YC companies that would be lucky to have you. Let me know if you need help with intros."

**Sign-off**: "Thanks again!" or "Thank you again!" — the hiring manager doesn't use "Best regards" typically (one candidate was an exception).

### Email formatting rule

**No hard line wrapping.** Write each paragraph as a single continuous line — let the email client handle wrapping. Never insert `\n` within a paragraph. Only use newlines between paragraphs.

### Step 4: Present for approval — DO NOT CREATE YET

```
## Rejection: Jane Doe
**Thread:** Re: Reaching out from Y Combinator
**To:** janedoe@gmail.com
**Stage:** Technical Screen | **Met with:** Interviewer A
**Ashby archive reason:** Not a good match

---
Hey Jane,

Thanks for making time to come through the exercise with Interviewer A.

We got a chance to catch up, and unfortunately we're not going to proceed with the process at this time.

I appreciate you taking time out of your schedule for this, and if I can be helpful as you're looking for your next role, don't hesitate to reach out.

Thanks again!

{hiring manager's first name from user.md}
---

**d)** draft — create in Gmail drafts
**s)** send and archive — send email + archive in Gmail + archive in Ashby
**e)** edit — tell me what to change
**?)** something else
```

### Step 5: Wait for approval

| User says | Action |
|-----------|--------|
| **d** | Create Gmail draft. Don't archive in Ashby yet. |
| **s** | Create Gmail draft → send via `gmail_send_draft` → archive email via `gmail_archive` → archive in Ashby via `application_change_stage` with archive reason |
| **e** | Ask what to change, update, re-present |
| **?** | Ask the user what they want |

### Step 6: Execute (only after approval)

1. **Gmail draft**: `mcp__claude_ai_Gmail__gmail_create_draft(to, subject, body, threadId, contentType: "text/html")` — wrap body in `<p>` tags, use `<br><br>` between paragraphs
2. **If sending**: `gmail_send_draft(draftId)` → `gmail_archive(message_id)`
3. **Ashby archive**: `application_change_stage(applicationId, interviewStageId: "<Archived stage ID from recruit-config/jobs/>", archiveReasonId: <selected reason>)`

#### WAAS messaging channel
1. **Send message**: `candidate_message_send(short_id, message)`
2. **WAAS archive**: `candidate_status_update(short_id, state: "archived", archive_reason: "not_qualified")`
3. **Ashby archive** (if also in Ashby): `application_change_stage(applicationId, interviewStageId: "<Archived stage ID from recruit-config/jobs/>", archiveReasonId: <selected reason>)`
   - Archive reason IDs: read from `recruit-config/ashby.md`. Match the reason to the situation.
   - If the user specified a reason, use the matching one. If not, default to "Not a good match."
4. **Verify (MANDATORY)**: DO NOT report success until you have re-queried to confirm each action took effect. Check `gmail_list_drafts` or sent mail for the email. Check `application_info` for the Ashby archive. Check `candidate_status_show` for WAAS archive. Report what you verified.

## Choosing the channel: Gmail vs WAAS messaging

Before drafting, determine the right channel:

1. **Check for an existing Gmail thread** with the candidate.
2. **Check if the candidate has a WAAS short_id** — use the unified resolver (`resolve-candidate.md`).

| Situation | Channel |
|-----------|---------|
| Existing Gmail thread | **Gmail** (reply in-thread) |
| No Gmail thread, has WAAS short_id | **WAAS messaging** (`candidate_message_send`) — messages appear as emails to the candidate |
| No Gmail thread, no WAAS short_id | **Ashby-only archive** (no email — Ashby sends its own rejection if configured) |

When using WAAS messaging:
- Read conversation history first via `candidate_messages_list(short_id)`
- No draft concept — options become **s/e/?** (no "d")
- After sending, archive in WAAS: `candidate_status_update(short_id, state: "archived", archive_reason: "not_qualified")`
- If also in Ashby, archive there too

## Application Review Rejections (no contact)

If the candidate is at Application Review and the hiring manager never contacted them:
- **WAAS applicants**: Send a brief rejection via `candidate_message_send` + archive in WAAS (`candidate_status_update(state: "archived")`) + archive in Ashby
- **Ashby direct applicants with no thread**: Just archive in Ashby (no email needed — Ashby sends its own rejection if configured)
- **J&J intros with no reply**: Just archive in Ashby (no email to the candidate; J&J handles it)

## Rules

1. **NEVER send or archive without explicit approval.**
2. **Always personalize.** Reference the interviewer's name, the exercise type, something specific. Never use a generic template verbatim. For Application Review rejections (no call), a brief "thanks for your interest, not proceeding" is fine.
3. **Never say what the candidate did wrong.** The hiring manager's rejections are gracious. "Not proceeding" is enough.
4. **Use the threadId** from the existing email thread. Never start a new thread.
5. **Check for email address changes** before drafting.
6. **"s" does three things**: send email, archive in Gmail, archive in Ashby. All three. Confirm this in the output.
