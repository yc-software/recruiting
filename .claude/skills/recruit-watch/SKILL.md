---
name: recruit-watch
description: |
  Background recruiting monitor. Watches for new Ashby candidates without outreach,
  candidate replies waiting for the hiring manager, upcoming interviews, and stale pipeline items.
  Only surfaces things that need action — stays quiet otherwise. Designed to run on
  a loop: `/loop 10m /recruit-watch`.
allowed-tools:
  - Bash
  - Read
  - Grep
  - Glob
  - WebSearch
  - mcp__google-workspace__gmail_query_emails
  - mcp__google-workspace__gmail_get_email
  - mcp__google-workspace__calendar_get_events
  - mcp__ashby__candidate_search
  - mcp__ashby__candidate_info
  - mcp__ashby__application_info
  - mcp__ashby__application_list
  - mcp__ashby__interview_schedule_list
  - mcp__ashby__candidate_list_notes
  - mcp__claude_ai_Gmail__gmail_search_messages
  - mcp__claude_ai_Gmail__gmail_read_message
  - mcp__claude_ai_Google_Calendar__gcal_list_events
  - mcp__claude_ai_Slack__slack_send_message
  - mcp__claude_ai_Slack__slack_read_channel
  - mcp__claude_ai_Slack__slack_read_thread
  - mcp__claude_ai_Slack__slack_search_users
  - mcp__waas__applicant_list
  - mcp__waas__candidate_show
  - mcp__waas__candidate_status_show
  - mcp__waas__candidate_status_update
  - mcp__waas__candidate_message_send
  - mcp__waas__candidate_messages_list
  - mcp__waas__candidate_notes_list
  - mcp__waas__candidate_note_create
  - mcp__waas__health_check
  - mcp__gmail-drafts__gmail_list_drafts
---

> **Note:** Read `recruit-config/user.md` for the hiring manager's name, email, and preferences. Do not hardcode any user-specific values.


# Recruit Watch

Background recruiting monitor that does research, makes recommendations, and communicates with the hiring manager over Slack. Designed to run on a loop via `/loop 10m /recruit-watch`.

**IMPORTANT**: On startup, read these config files for IDs, templates, and candidate bar:
- `recruit-config/user.md` — hiring manager email, Zoom link, office, Slack IDs, noise senders
- `recruit-config/ashby.md` — source IDs, archive reasons, email style, draft tool rules
- `recruit-config/waas.md` — WAAS pipeline stages, job IDs, MCP tools` — Slack/Gmail config, source IDs, archive reasons
- `recruit-config/CLAUDE.local.md` — Slack bot token (gitignored)
- `recruit-config/jobs/product-engineer.md` — job ID, stage IDs, candidate bar, Slack format, tone rules

Do NOT hardcode any IDs, tokens, emails, or templates. Always read from config.

## Philosophy

**Be proactive, not passive.** Don't just flag things — research them and make a recommendation. Send findings to Slack so the hiring manager can act from anywhere. Accept replies from the hiring manager on Slack to keep the conversation going.

**Stay quiet when nothing is new.** If nothing changed since the last check, don't send anything.

## Slack Configuration

Read channel ID, the hiring manager's Slack user ID, and bot token from config files. Use the copilot bot user ID from config to distinguish bot messages from the hiring manager's.

### Sending messages (as Copilot bot)

**IMPORTANT**: Always send Slack messages using the copilot bot token via curl, NOT the Claude AI Slack MCP. This sends as "Copilot" bot so the hiring manager gets notifications (the MCP sends as the hiring manager, which doesn't notify).

Read the bot token from `recruit-config/CLAUDE.local.md` and the channel ID from `ashby.md`.

```bash
cat <<'PAYLOAD' | curl -s -X POST https://slack.com/api/chat.postMessage \
  -H "Authorization: Bearer $BOT_TOKEN" \
  -H "Content-Type: application/json; charset=utf-8" \
  -d @-
{"channel":"$CHANNEL_ID","text":"your message here"}
PAYLOAD
```

- **Use `\n` for newlines** in the JSON text field
- **CC the hiring manager** at end: `cc <@$HIRING_MANAGER_SLACK_ID>` — do NOT tag at beginning
- **Thread replies**: add `"thread_ts":"<parent_ts>"` to the JSON payload
- **Message format**: Slack markdown (`*bold*`, `•` for bullets)

### Reading messages (for the hiring manager's replies)

Use the same bot token via curl. Use form-encoded for reading — JSON doesn't work reliably.

```bash
curl -s -X POST https://slack.com/api/conversations.history \
  -H "Authorization: Bearer $BOT_TOKEN" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "channel=$CHANNEL_ID&limit=15"
```

## Gmail filtering rules

**IMPORTANT**: When querying Gmail, do NOT blanket-exclude `notifications@ashbyhq.com`. Ashby sends two types of emails:
- **Application alerts** (`[Ashby] X applied to a job you're watching`) — these are IMPORTANT, keep them
- **Stage change / feedback reminders** (`[Ashby] Candidate X has changed stage`, `[Ashby] Reminder:`) — these are noise, safe to exclude

Read the noise sender list from `ashby.md` and exclude them from inbox queries.

Or better: **always check Ashby API directly** for new applicants via `application_list`. The Ashby API is the source of truth.

## What it watches (and what it DOES about it)

### 1. New applicants → Research + Recommend

**Primary method**: Check Ashby API directly for candidates at "Application Review" stage (get stage ID from job config).
**Secondary method**: Check Gmail for Ashby application notification emails.

For each new applicant:
- Pull their Ashby profile
- **For WAAS applicants**: Always read the full email body — never recommend based on subject line alone.
- Check against candidate bar in job config
- **Slack the hiring manager with a recommendation** using the format from job config
- **Recommendation categories:**
  - **Archive** — fails hard requirements (non-engineering background, incomplete profile). **Always require hiring manager approval before archiving.** Do not auto-archive based on location or work authorization — flag these for human review instead.
  - **Dig deeper** — some signal but not enough to decide. Research LinkedIn/GitHub before recommending
  - **Ask for examples** — borderline candidate where more information would help (e.g., limited portfolio or unclear experience level). Slack the hiring manager with a recommendation to send a short email asking for examples of projects. **Do not archive until the hiring manager reviews the response or decides to pass.** Never auto-archive a candidate who has been asked to provide more information.
  - **Outreach** — strong signal, draft outreach via `/outreach`

### 2. Candidate replies → Draft response

When a candidate replies to the hiring manager's email:
- Read the email content, check Ashby stage
- Draft the appropriate response
- Slack the hiring manager with the draft

### 3. In-progress candidates → Recommend next step

For candidates at active stages where the ball is in the hiring manager's court:
- Check last interaction, determine next step
- Draft email if appropriate, Slack the hiring manager

### 4. New sourcing lists → Research each candidate

When WAAS AI Sourcer or digest emails arrive:
- Parse the candidate list, research each one
- Slack the hiring manager with a ranked summary

### 5. Upcoming interviews → Brief + reminder

2 hours before, send a Slack reminder with a quick brief.

### 6. Stale candidates (daily)

Once per day, flag candidates with no activity in 7+ days.

### GATE — Do not post to Slack until this is complete on each iteration

Before posting ANY recommendation or status update to Slack, verify you have checked ALL of the following on this iteration:

1. **Ashby pipeline** — `application_list` for ALL open jobs (not just PE), checked for new applicants and stage changes
2. **Gmail inbox** — scanned for new candidate replies, WAAS application emails, sourcing digests
3. **Gmail sent** — for every candidate you're about to surface, verified the hiring manager hasn't already replied (search by email address, not name)
4. **Gmail drafts** — checked `gmail_list_drafts` to see if the hiring manager has a pending draft for any candidate you're about to flag
5. **Calendar** — checked upcoming events for interview reminders, cross-referenced with pipeline candidates
6. **WAAS** — `health_check` first, then `applicant_list` for new applicants (if API available)
7. **Slack** — read thread contents for the hiring manager's replies (per rule 14)

If a candidate already has a Gmail draft pending, do NOT flag them as "needs action" — the hiring manager is already working on it. If the hiring manager already sent a reply, do NOT flag the candidate as "waiting for response."

Every Slack message must include a sources-checked line at the bottom:
```
_Checked: Ashby, Gmail in/sent/drafts, Calendar, WAAS, Slack_
```

## Slack interaction loop

On each cron iteration:

1. **Check for Slack replies** from the hiring manager (read thread contents per rule 14)
   - Parse reply and execute action
   - Confirm in thread

2. **Run the GATE checklist** — query all sources before evaluating what's new

3. **Surface new items** (applicants, replies, sourcing lists, calendar)
   - Only surface items not already sent
   - Research and recommend

4. **Print timestamp to console** (e.g., `[11:32am] recruit-watch: nothing new`)

## Parsing the hiring manager's Slack replies

| User says | Action |
|-----------|--------|
| 👍 or "yes" or "do it" | Execute the recommended action |
| 👎 or "no" or "skip" | Archive in Ashby (use archive reason from config). Archive notification emails in Gmail. Confirm in thread. |
| "advance" | Draft advancement email via `/reply` |
| "pass" or "reject" | Draft rejection via `/reject` |
| "1, 3" (numbers) | Act on those specific candidates from a list |
| "archive" | Archive in Ashby |
| "outreach" or "reach out" | Draft outreach email via `/outreach` |
| "ask" or "ask for examples" | Send "what are you building?" email (template above), archive notification in Gmail |
| Free text | Treat as instructions |

## State tracking

Track in memory within the session:
- Messages already sent to Slack (candidate ID + action type)
- Threads awaiting the hiring manager's reply
- Don't re-send the same recommendation twice

## Output

- **Primary**: Slack channel
- **Secondary**: Console output
- **Quiet mode**: No Slack message — but always print timestamp to console

## Email formatting rule

**No hard line wrapping.** Write each paragraph as a single continuous line — let the email client handle wrapping. Never insert `\n` within a paragraph. Only use newlines between paragraphs.

## Rules

1. **Always recommend, never just notify.**
2. **Draft emails proactively.** Use `/outreach` for cold outreach, `/reply` for responses.
3. **Accept Slack replies.** Check for the hiring manager's responses on each iteration and act on them.
4. **Confirm after acting.** Always reply in the Slack thread confirming what was done.
5. **Stay quiet when nothing is new.** No "all clear!" messages.
6. **De-duplicate across checks.** Don't re-send the same recommendation.
7. **Research before recommending.** Pull Ashby profile, check email history, look at background.
8. **Keep Slack messages concise.** Use the format from job config.
9. **Console output mirrors Slack.** Same content appears in both places.
10. **Never send emails without the hiring manager's approval.**
11. **Auto-archive related emails after taking action.** Archive Ashby notifications and WAAS emails after acting. **Do NOT archive J&J emails** — the hiring manager may want to read those.
12. **Never use old Slack messages as source of truth.** Check actual sources fresh each iteration.
13. **Verify before posting.** Check inbox, sent mail, calendar, and Ashby stage before claiming something is outstanding. **When checking sent mail, always search by the candidate's email address** (from Ashby profile), not their name. Name-based Gmail searches (`to:firstname lastname`) often miss sent emails — use `to:email@example.com` instead.
14. **Always read thread contents, not just reply counts.** Read actual messages in every thread with replies. Check if the last message is from the hiring manager without a bot response after it. Never infer state from `reply_count` alone.
15. **Broadcast thread replies to the channel.** Always include `"reply_broadcast": true` in thread reply payloads.
16. **Read config, never hardcode.** All IDs, tokens, emails, templates, and candidate bar criteria come from config files. Never embed them in this skill.
