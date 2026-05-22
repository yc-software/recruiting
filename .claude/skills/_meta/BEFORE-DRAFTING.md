# BEFORE-DRAFTING.md — Mandatory Pre-Draft Gate

**Read this file in full before producing any candidate-facing draft.** Outreach, reply, nudge, follow-up, reject, brief, scheduling note, intro -- everything that lands in a candidate's inbox or Slack DM.

If you skipped this gate and went straight to drafting, stop, read this, and redraft.

---

## 1. Why this file exists

The bootstrap `hooks/bootstrap/AGENTS.md` listed a table of skills to load. Tables get skimmed. This file is a single explicit gate that you `read` once per drafting task, and it tells you exactly which other files to load for the situation in front of you. One mandatory hop, no guessing.

---

## 2. Load order (do this every time, in this order)

For **any** candidate-facing draft:

1. **`USER.md`** -- hiring manager identity, formatting non-negotiables (ASCII, `--`, no hard wraps, sign-off rules), Zoom link, office, voice rules.
2. **`SOUL.md`** -- the always-injected voice non-negotiables. (You already have this in context; treat it as the floor, not the ceiling.)
3. **`VOICE-NOTES.md`** -- the full file. Sections 5 (anti-patterns), 6 (phrase bank), 7 (personalization mechanics), 8 (tone shifts) are load-bearing. Do not skim. Skipping 5-8 is the single most common voice-drift failure.
4. **`PLAYBOOK.md`** -- process scaffolding, tool mappings, hiring manager voice summary.
5. **The job playbook** (`jobs/<role>.md`) -- role-specific outreach template, bullet bank, candidate bar, tone rules, anti-patterns Ryan has explicitly called out.
6. **The relevant skill file** -- `outreach`, `reply`, `reject`, `schedule`, `candidate-brief`, etc. The skill defines the *procedure*; voice files define *how it sounds*.
7. **Source-system context** -- whichever of Ashby / WAAS / Gmail / Calendar is relevant to the candidate in front of you. Always pull the most recent email thread; never draft a "first email" if there's prior conversation.

If any of those files is empty or missing, say so to the user instead of substituting your own judgment for the missing content.

---

## 3. Drift-check (run before showing any draft)

Answer these explicitly to yourself before presenting. If any answer is no, fix the draft and re-check.

1. **Skill loaded?** Did I read the SKILL.md for this task type (outreach / reply / reject / schedule / etc.) this turn?
2. **Voice loaded?** Did I re-read VOICE-NOTES.md sections 5 (anti-patterns), 7 (personalization), 8 (tone shifts) this turn? Section 5 is non-negotiable for every send.
3. **Anti-patterns clear?** Does the draft pass every "Never" rule in VOICE-NOTES section 5 *and* the job playbook's anti-patterns section?
4. **Phrase-bank check?** Does the language come from VOICE-NOTES section 6 or the job playbook, or have I just invented openers/closes that "sound like Ryan"? Invented language is a drift signal.
5. **Personalization shape?** Does the hook follow Move A (arc + read), B (echo their stated wants), or C (specific project/artifact reaction) from VOICE-NOTES section 7 -- *not* a resume-fact list?
6. **Formatting clean?** ASCII only, `--` not `—`, straight quotes, no hard line wraps, sign-off is `Ryan` alone (no `Best`, no `Cheers`, no full sig block in candidate emails).
7. **Threading right?** If a prior thread exists, am I replying in-thread, not starting a new one?
8. **Source verified?** Every specific claim about the candidate ("shipped X at Y", "MIT grad", "ICCC paper") came from a file I actually read (Ashby resume, WAAS profile, prior thread, candidate's stated material) -- not inferred or invented.

---

## 4. Common drift modes (catch these specifically)

These are failures I (the agent) have actually made. Pattern-match against this list before sending.

- **Inventing openers that "sound like Ryan":** *"One more note from me"*, *"Wanted to flag this again"*, *"Quick follow-up"*, *"Circling back"*, *"Know the X start is fresh"*, *"Not trying to pull you out"*. None of these are in the VOICE-NOTES phrase bank. Real follow-up openers from the corpus: dive straight in with `Hi {Name},` + a specific reason for the renewed reach, or skip the salutation entirely and start with the substantive line.
- **Resume-list hooks:** stacking 2-3 company names with no read on what they say about the candidate ("MIT to Regard to Stratos to Vast AI") is exactly the anti-pattern called out in the job playbook. Pick **one** thing and say something specific about it.
- **AI-flavored writerly words:** *arc*, *interesting move*, *plant a flag on*, *saying out loud*, *weigh it*, *dogfooded*. Cut.
- **Superlatives about the candidate:** *exactly*, *perfect fit*, *ideal candidate*, *exactly the depth we need*. Cut.
- **Corporate filler in apologies:** *"Apologies for any inconvenience"*. Real Ryan: *"Sorry for dropping the ball on this"*, *"augh! sorry about that"*.
- **Closing with `Best,` / `Cheers,` / `All the best,`:** never. Just `Ryan` on its own line.
- **Adding follow-up-cadence acknowledgments:** *"following up"*, *"circling back"*, *"bumping this"*, *"last note from me"*, *"I know I've tried a couple times"*. Per the reply skill: the recipient doesn't care about your outreach cadence.
- **Recapping or contradicting the prior email in-thread:** the prior message is already above in the thread. Don't repeat its content; build on it.
- **Capitalization too formal in warm 3rd-4th-reply threads:** VOICE-NOTES section 8 -- capitalization decays as the thread warms. A 4th in-thread reply in full sentence case reads off.

---

## 5. OpenClaw tool translation (one place, not scattered)

Skill files list Claude-Code MCP tool names (`mcp__google-workspace__*`, `mcp__ashby__*`, `mcp__waas__*`). On OpenClaw, the *behavior* is the same; the *invocation* is different. Map:

| Skill says | OpenClaw equivalent |
| --- | --- |
| `mcp__google-workspace__gmail_*` / `mcp__claude_ai_Gmail__*` | `gog --account ryan@ycombinator.com gmail …` |
| `mcp__google-workspace__calendar_*` / `mcp__claude_ai_Google_Calendar__*` | `gog --account ryan@ycombinator.com calendar …` |
| `mcp__gmail-drafts__gmail_create_draft` | `gog gmail drafts create --body-html …` (HTML wrapper avoids 78-char hard wrap) |
| `mcp__gmail-drafts__gmail_send_draft` | `gog gmail drafts send <draftId>` |
| `mcp__ashby__*` | `curl` to `https://api.ashbyhq.com/<endpoint>` with HTTP Basic auth (key at `/data/.ashby/api_key`, **always set `User-Agent` header** -- without it Ashby returns 403). |
| `mcp__waas__*` | `curl` to `https://api.ycombinator.com/v1/<endpoint>` with `Authorization: Bearer <token>` (token at `/data/.yc/waas-credentials.json`). |
| `mcp__claude_ai_Slack__*` (read) | Not currently wired -- ask the user. |
| Sending Slack as Copilot bot | Use the bot token from a gitignored config file when present; otherwise tell the user it's not wired. |

Behavior rules from PLAYBOOK that still apply:
- **Always send Slack messages as the Copilot bot** (not as the hiring manager), so Ryan gets notifications. If the bot token isn't configured locally, surface that to the user.
- **Always thread Gmail replies on the existing threadId.** Never start a new thread when a prior exchange exists.
- **Never create a calendar invite until the candidate confirms the time.**
- **Never send anything without explicit approval** (the d/s/e/? loop).

---

## 6. The approval loop

Every candidate-facing draft -- regardless of channel -- is presented to the user with these options before any send/draft/archive happens:

- `d` -- create as Gmail draft (or platform-equivalent), do not send
- `s` -- send now (+ any companion actions named in the draft, e.g. archive in Gmail, advance in Ashby)
- `e` -- edit; the user tells you what to change
- `?` -- anything else / questions / discuss

Never send, draft, or change Ashby/WAAS stage without an explicit `s` (or equivalent affirmative) from the user *for that specific draft*. A previous `s` does not authorize the next send.

---

## 7. When in doubt

- If the voice doesn't match a real Ryan example, **say it plainer.** Short and direct beats clever every time.
- If the candidate's situation is novel and no template fits, **ask the user** rather than guess.
- If a file referenced in this gate is missing or empty, **tell the user**, do not improvise.
