---
name: reply
description: |
  Reply to a candidate. Reads the full situation — Ashby stage, email history, conversation
  context, meeting transcripts, playbook templates — and drafts the right response with the
  right tone. Handles scheduling, advancement, Q&A, mutual partings, congrats, warm close-outs,
  and everything in between. Use when the user says "reply to X", "draft for X", "write back to X",
  "email X", or any variation of responding to a candidate.
allowed-tools:
  - Bash
  - Read
  - mcp__google-workspace__gmail_query_emails
  - mcp__google-workspace__gmail_get_email
  - mcp__google-workspace__gmail_bulk_get_emails
  - mcp__google-workspace__calendar_get_events
  - mcp__claude_ai_Gmail__gmail_create_draft
  - mcp__gmail-drafts__gmail_send_draft
  - mcp__gmail-drafts__gmail_list_drafts
  - mcp__gmail-drafts__gmail_delete_draft
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
  - mcp__waas__candidate_note_create
---

> **Note:** Read `recruit-config/user.md` for the hiring manager's name, email, and preferences. Do not hardcode any user-specific values.


**Config**: Read these files before executing — they contain all IDs, tokens, templates, and rules:
- `recruit-config/user.md` — hiring manager email, Zoom link, office, Slack IDs, noise senders
- `recruit-config/ashby.md` — source IDs, archive reasons, email style, draft tool rules
- `recruit-config/waas.md` — WAAS pipeline stages, job IDs, MCP tools` — Slack/Gmail config, source IDs, archive reasons
- `recruit-config/CLAUDE.local.md` — Slack bot token (gitignored)
- `recruit-config/jobs/product-engineer.md` — job ID, stage IDs, candidate bar, outreach template, tone rules

# Reply to Candidate

The single skill for replying to any candidate in any situation. Reads the context, picks the right tone and content, and presents for approval before doing anything.

## When to use this vs `/reject`

| Situation | Use |
|-----------|-----|
| **The hiring manager/YC decided not to proceed** (one-sided) | `/reject` — "we're not going to proceed" language + tone ladder |
| **Everything else** | **`/reply` — this skill** |

This skill handles:
- **Scheduling** — confirm, propose, reschedule, pick from candidate's times
- **Advancement** — post-screen follow-up, adding interviewer, build exercise instructions
- **Q&A** — candidate questions about role, process, work auth, culture
- **Onsite logistics** — arrival instructions, lunch, Executive A availability
- **Mutual parting** — "we both agreed this isn't the right fit" (NOT a rejection)
- **Accepted other offer** — congratulations + stay in touch
- **Warm close-out** — great conversation, timing/location/fit didn't work, door open
- **Follow-up after a call** — personalized based on what was discussed
- **Candidate re-engagement** — they came back after going cold
- **Candidate withdrew** — gracious, no hard feelings, door open
- **Competing offers** — urgency, move fast
- **First reply to intro** — J&J, WAAS, cold inbound
- **Anything else** — if the user gives context, this skill can draft it

## Input

- `/reply Jane Doe — reschedule to Monday 10:15am`
- `/reply John Smith` — with transcript/context in conversation history
- `/reply Candidate O — congrats on the new gig, stay in touch`
- `/reply Candidate A, Candidate B, Candidate C` — multiple candidates
- "write back to X" / "draft for X" / "email X" (routed from `/recruit`)

## Procedure

### Step 1: Gather context

For each candidate:
1. **Ashby**: `candidate_search(name)` → stage, application history, source
2. **Gmail**: find the most recent thread, get threadId, read the latest exchange
3. **Gmail sent**: scan for what the hiring manager previously said to them
4. **Calendar**: check for upcoming events with this candidate
5. **Conversation context**: check what the user told you in the current session — transcripts, instructions, tone guidance. **This is the most important input.**
6. **Email thread**: check for email address change requests

### Step 2: Understand the situation and draft

Read the room. The right reply depends on what's actually happening:

#### Template situations (use playbook at `.coding-agent-plans/recruiting-assistant.md`)

| Situation | What to do |
|-----------|------------|
| Candidate proposed times, need to pick one | Check the hiring manager's calendar, pick a slot, use Scheduling Confirmation template |
| Candidate confirmed a time | Use Scheduling Confirmation — confirm variant, create calendar event |
| Need to propose times to candidate | Check the hiring manager's calendar, use Scheduling Confirmation — propose variant |
| Missed a call / dropping the ball | Use Scheduling Confirmation — reschedule variant |
| Pre-screen qualification (before scheduling a call) | Use Pre-Screen Qualification template (see below) |
| Post-screen, moving to tech screen | Use Post-Screen Advancement template — **MUST ask the hiring manager who to CC** (see below) |
| Post-screen, warm follow-up | Use Post-Screen Follow-up — Warm Advancement |
| Candidate has competing offers | Use Urgency / Competing Offers template |
| New J&J or WAAS intro, first reply | Personalized outreach based on background |
| Onsite logistics | Use Onsite templates (arrival, lunch, Executive A availability) |
| Candidate asked a question | Use standard answers from playbook Q&A section |

#### Context-driven situations (read the room, draft in the hiring manager's voice)

| Situation | Tone | Key signals |
|-----------|------|-------------|
| **Mutual parting** | Warm, peer-to-peer. "As we discussed" not "we decided." No rejection language. | The user says "we agreed", "mutual", transcript shows both sides acknowledged it |
| **Accepted other offer** | Congratulatory, genuine. Happy for them. Door open. | Ashby shows "Accepted Other Offer" or The user says they took another job |
| **Timing didn't work** | Warm, door open. "Keep us in mind." | Candidate had life event, relocation timing, visa issue |
| **Location/commute** | Empathetic, practical. Acknowledge the reality. | Commute discussion, remote preference, relocation hesitation |
| **Great candidate, no role fit** | Warmest — offer concrete help (intros to YC companies). | The user says they.re strong but not right for this specific role |
| **Post-call follow-up** | Casual, reference specific things from the call. | The user just had a call and wants to follow up |
| **Re-engagement** | Warm, no pressure, acknowledge the gap. | Candidate went cold, now back |
| **Candidate withdrew** | Gracious, no hard feelings, door open. | Candidate said they're pulling out |

### The hiring manager's voice patterns

**Openings** — always reference something specific:
- "Thanks for making time to chat — really enjoyed the conversation."
- "Thanks for getting back to me!"
- "Thanks for the honest conversation yesterday."
- "Thanks for letting me know!"

**Scheduling** — short and warm:
- "Put time on the calendar. Looking forward to it!"
- "{time} works great! Put time on the calendar."
- "Does {day} at {time} PT happen to work for a call?"
- "Sorry for dropping the ball on this — does {day} at {time} happen to work?"

**Mutual parting / close-out** — direct, not euphemistic:
- "As we discussed, [the thing] would be tough to make work."
- "Totally understand. Keep us in mind if things change down the road."

**Congrats on other offer** — genuinely happy:
- "Congrats — that sounds like a great fit."
- NOT "sorry to hear" — the hiring manager is happy for people.

**The offer to help** (calibrated by relationship):
- Brief interaction: "If we can ever be helpful, don't hesitate to reach out."
- Good conversation: "Let me know if I can be helpful — happy to chat anytime."
- Strong connection: "There are a number of great YC companies that would be lucky to have you. Let me know if you need help with intros."
- Personal: "Hope to stay in touch." Or reference specifics: "Tap me on the shoulder when that project gets closer."

**Sign-off**: "Thanks again!" or the hiring manager's first name (from user.md) — never "Best regards" or "All the best."

### Pre-Screen Qualification (before scheduling a call)

Before committing to a phone screen, the hiring manager sometimes wants to see evidence of full-stack building and AI depth. Use this when:
- A candidate replied to outreach or a J&J intro and the user wants to qualify before scheduling
- the user says "ask them to show me what they've built" or "qualify them first"
- The candidate's background doesn't clearly show full-stack or AI experience

**What the hiring manager wants to see:**
1. **Full-stack app** — have they built something real, end-to-end (database to frontend)? Something fairly sophisticated, not a tutorial project.
2. **AI depth** — what AI tools do they use, what have they built with AI, how deep do they go? An AI coding session summary or description of their setup.

**Template:**
```
Hey {candidateFirstName},

{Warm opener — excited about their interest, reference something specific if available.}

I would love to see two things:

- Any full stack app that you have built that you could share with me. Something fairly sophisticated.
- An AI summary of one of your most recent chats or builds, to see how much you are building with AI.

Looking forward to hearing from you!

{hiring manager's first name from user.md}
```

**When to use vs when to skip:**
- **Use** when: candidate's background is strong but unclear on full-stack or AI (e.g. systems engineer, backend-only, finance background transitioning to eng)
- **Skip** when: candidate clearly has full-stack + AI evidence already (e.g. WAAS profile shows projects, they mentioned AI tools in their message, their resume shows full-stack apps)

**Rules:**
1. Personalize the opener based on what you know about them
2. Keep the two asks as bullet points — clean and direct
3. Don't over-explain why you're asking — just ask

### Post-Screen Advancement (Phone Screen → Tech Screen)

When advancing a candidate from Phone Screen to Technical Screen, use this template. **Before drafting, you MUST ask the hiring manager two things:**

1. **"Who should I CC on this?"** (e.g. Interviewer A, Interviewer B, Interviewer A) — never guess the engineer.
2. **"When do you have 1-hour blocks available?"** — check the hiring manager's calendar (and the CC'd engineer's if possible) using `/schedule`, then present the open 1-hour slots so the hiring manager can pick one or let the candidate propose.

**Interview process doc (always include):** `<interview process doc URL from recruit-config/jobs/>`

**Template (initial — asking for availability):**
```
Hi {candidateFirstName}!

Thank you so much for your time {today/yesterday}!

{1 sentence referencing something from the call — their background, a topic discussed, or what interested the hiring manager. Keep it muted and natural. Don't over-sell ("exactly what we need", "great fit"). Just note the relevance. Examples:
- "Really enjoyed hearing about the risk infrastructure you've been building at [Investment Bank] — there's a lot of overlap with how we think about building tools to manage YC's fund."
- "Really enjoyed the conversation — I think you'd find the concrete metrics and clear goals here at YC to be a refreshing change from what you described at [Previous Co]."
- "Really enjoyed hearing about your fintech and asset allocation work at [Startup A] and [Startup B] — there's a lot of that on the Investment Ops side here."}

As promised, here is the doc re: our interview process. I hope it gives you a sense of what we're looking for, and how we evaluate for it:

<interview process doc URL from recruit-config/jobs/>

I'd love to introduce you to {engineerName}, who is an engineer on our team. {He/She}'ll be running through an exercise — you'll be building a full webapp in an hour. Please come prepared with a full stack environment — database, backend, and frontend — all ready to go. Use all the AI you want.

Let us know when you might have an hour to go through the exercise. Happy to answer any questions, and excited to hear how it goes!

{hiring manager's first name from user.md}
```

**Template (time already set — adding the engineer):**
```
Put time on the calendar, and also adding {engineerName} on our team who will go through the exercise.

Please come with a full stack environment set up on your laptop, and any AI tools you typically use to build. We'll be giving you a prompt to build an application, and watching you speed build it in an hour. We are looking for developers who are very comfortable coding with AI, building quickly, and understanding ultimately the code and technology underneath.

Let us know what questions you have in advance. I'm excited for you to meet {engineerName} and learn more about YC and go through the exercise!

{hiring manager's first name from user.md}
```

**Rules for this template:**
1. **ALWAYS ask the hiring manager who to CC** before drafting. Present it as: "Who should I CC on this? (e.g. Interviewer A, Interviewer B, Interviewer A)"
2. **ALWAYS check calendar** for available 1-hour blocks and present them to the hiring manager. Format: "You have these 1-hour blocks open next week: Mon 2-3pm, Thu 2-3pm, Thu 3-4pm. Want to suggest one to the candidate, or let them propose?"
3. **CC the engineer** on the email — they need to see the thread to coordinate scheduling.
4. **Include the Google Doc link** in the initial template. Not needed in the "time already set" variant.
5. **Key exercise framing phrases** (always include): "full stack environment — database, backend, and frontend", "use all the AI you want", "building a full webapp in an hour".
6. **Don't change the template structure** — the hiring manager uses this exact flow: thanks → personalized line → doc link → introduce engineer → ask for availability.
7. **Personalization tone** — add one line referencing something from the phone screen (their background, a topic, relevance to YC). Keep it **muted and natural**. Don't over-sell or exaggerate ("exactly what we need", "great fit", "perfect match"). Just note the relevance or overlap. "There's a lot of that here" > "that's exactly what we're looking for."

### What NOT to do

- **Don't use rejection language for mutual partings.** "We're not going to proceed" is a rejection. Use `/reject` for that.
- **Don't be generic.** Reference something specific — their project, background, what they said.
- **Don't over-explain.** The hiring manager is concise. If the call covered it, the email doesn't relitigate.
- **Don't be corporate.** No "we wish you the best in your future endeavors."
- **Don't assume tone.** If the user gave instructions, follow them. If ambiguous, ask.

### Email formatting rule

**No hard line wrapping.** Write each paragraph as a single continuous line — let the email client handle wrapping. Never insert `\n` within a paragraph. Only use newlines between paragraphs (e.g. between the body and the hiring manager's name sign-off).

### Choosing the channel: Gmail vs WAAS messaging

Before drafting, determine the right channel for this candidate:

1. **Check for an existing Gmail thread** (`gmail_query_emails` for emails from/to candidate).
2. **Check for a WAAS profile** — use the unified resolver (`resolve-candidate.md`). If the candidate has a `short_id`, pull their WAAS messages via `candidate_messages_list`.

| Situation | Channel | Why |
|-----------|---------|-----|
| Existing Gmail thread | **Gmail** (reply in-thread) | Continue the conversation where it started |
| No Gmail thread, but has WAAS short_id | **WAAS messaging** (`candidate_message_send`) | WAAS messages appear as emails to the candidate — no need to create a separate thread |
| No Gmail thread, no WAAS short_id | **Gmail** (new thread) | Only option — candidate came via J&J, cold inbound, or Ashby direct |

When using WAAS messaging:
- Read the conversation history first via `candidate_messages_list`
- The d/s/e/? approval flow is the same — but "d" is not available (WAAS has no draft concept). Options become **s/e/?**.
- After sending, update the WAAS pipeline stage if the reply changes the candidate's status (e.g., scheduling a screen → `candidate_status_update(state: "screen", pipeline_stage: "Screen")`)
- If the candidate is also in Ashby, update both systems

### Step 3: Present for approval — DO NOT CREATE YET

```
## Reply: {Name}
**Thread:** {subject line}
**To:** {email} (note any address change)
**Situation:** {one-line description of what's happening}
**Ashby action:** {Archive (reason) / None — candidate stays active}

---
{draft email}
---

**d)** draft — create in Gmail drafts
**s)** send and archive — send email + archive in Gmail + archive in Ashby
**e)** edit — tell me what to change
**?)** something else
```

If the reply doesn't close out the candidate (e.g. scheduling, Q&A, follow-up), omit the Ashby action line and adjust the options:

```
**d)** draft — create in Gmail drafts
**s)** send — send immediately
**e)** edit — tell me what to change
**?)** something else
```

### Step 4: Wait for approval

**DO NOT call any Gmail or Ashby tools until the user picks an option.**

| User says | Action |
|-----------|--------|
| **d** | Create Gmail draft via `mcp__claude_ai_Gmail__gmail_create_draft` with `contentType: "text/html"`. Report "Draft created — review in Gmail." |
| **s** (send and archive) | Create draft → `gmail_send_draft` → `gmail_archive` → archive in Ashby if applicable. Report all actions taken. |
| **s** (send, no archive) | Create draft → `gmail_send_draft`. Report "Sent." |
| **e** | Ask what to change, update, re-present with options |
| **?** | Ask the user what they want |

### Step 5: Execute (only after approval)

1. **Gmail draft**: `mcp__claude_ai_Gmail__gmail_create_draft(to, subject, body, threadId, contentType: "text/html")` — wrap body in `<p>` tags, use `<br><br>` between paragraphs
2. **If sending**: `gmail_send_draft(draftId)`
3. **If archiving Gmail**: `gmail_archive(message_id)` on the original inbound email
4. **If archiving Ashby**: `application_change_stage(applicationId, interviewStageId: "<Archived stage ID from recruit-config/jobs/>", archiveReasonId: <selected reason>)`

#### WAAS messaging channel
1. **Send message**: `candidate_message_send(short_id, message)`
2. **Update WAAS stage** (if the reply changes status): `candidate_status_update(short_id, state, pipeline_stage)`
3. **If also in Ashby**: update Ashby stage via `application_change_stage` to keep both systems in sync
   - Archive reasons:
     - Archive reason IDs: read from `recruit-config/ashby.md`
   - Match the reason to the situation.
5. **If creating calendar event**: use calendar event templates from playbook (no "screen"/"interview" in candidate-facing text)
6. **Verify (MANDATORY)**: DO NOT report success until you have re-queried and confirmed the action took effect. If you created a draft, check `gmail_list_drafts` and verify the threadId matches. If you sent, check sent mail. If you updated WAAS, check `candidate_status_show`. Report what you verified.

## Rules

1. **NEVER create drafts, send, or archive without explicit approval.** Always present first with d/s/e/? options.
2. **Context is king.** What the user told you in conversation > playbook templates > what you'd guess. If the user gave a transcript, use details from it.
3. **Read the room.** Match the emotional reality — mutual parting ≠ rejection, other offer = congrats, etc.
4. **Always personalize.** Reference something specific from the conversation or their background.
5. **Use the threadId** from the existing email thread. Never start a new thread.
6. **Check for email address changes** before setting the To field.
7. **"s" does all applicable actions**: send email, archive in Gmail (if closing out), archive in Ashby (if closing out). Confirm each in the output.
8. **Don't over-archive.** Only suggest archive when the conversation is clearly closing out. Scheduling replies, Q&A, follow-ups — candidate stays active.
9. **For multiple candidates**, present all drafts first, then ask "create all?" or let the hiring manager approve one by one.
10. **Read the playbook templates** (`.coding-agent-plans/recruiting-assistant.md`) for scheduling, advancement, and onsite situations. Use the hiring manager's established patterns.
11. **When in doubt about tone, ask the user.** Better to ask than guess wrong.

## Gmail & Calendar Rules

- **ASCII only in emails.** No em dashes, curly quotes, or unicode. Use `--` instead of em dashes and straight quotes. The plain-text Gmail MCP tool mangles unicode into garbage.
- **Phone screen calendar invites** use the title `<Candidate First Name> and Ryan` -- no "Phone Screen" label. Later stages (tech screen, onsite, partner) include the stage name.
- **Use `mcp__claude_ai_Gmail__gmail_create_draft`** (not `mcp__gmail-drafts__gmail_create_draft`) when creating email drafts. Set `contentType: "text/html"` and use `<p>` tags. The `gmail-drafts` tool only supports `text/plain` which hard-wraps lines at 78 characters, making emails look broken.

## Follow-Up Email Rules

When replying to candidates who haven't responded (follow-ups on cold outreach):

### Tone
- **Don't say "following up," "circling back," "bumping this," "last note from me," or "I know I've tried a couple times."** The recipient doesn't care about your outreach cadence. Get to the strongest lead immediately.
- **No hyperbole.** Don't use "exactly," "perfect fit," "ideal candidate," or superlatives about the candidate's background.
- **Don't flatter.** Don't say things like "exactly the depth we need" about their experience. It reads as recruiter-speak.

### Content
- **Follow the tone and content rules in the job playbook** (`recruit-config/jobs/`) for role-specific framing, hooks, and what to emphasize.
- **Personalize based on their background** — reference specific things they've built or done, not generic bullet points.

### Structure
- **Two paragraphs max.** First paragraph: the hook + why their background is relevant. Second paragraph: the ask.
- **The call-to-action must be on its own line**, separated from the body. Don't bury it in the middle of a paragraph.
- **Combine the call time and coffee offer in one line:** "Would Thursday at 9am or Friday at 11am work for a call? Also happy to grab coffee at our Dogpatch office if you're nearby."
