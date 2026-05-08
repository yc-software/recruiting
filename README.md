# Recruiting Co-Pilot

An AI recruiting assistant built with [Claude Code](https://claude.ai/code) skills. Manages the full hiring loop — sourcing, outreach, scheduling, interviewing, and pipeline management — with native integration for [Work at a Startup](https://www.workatastartup.com), Gmail, Google Calendar, and Slack. Also supports [Ashby](https://ashbyhq.com), which is what YC uses for interview scheduling and feedback.

## Why this exists

Recruiting is one of the hardest things you do as a founder. The only way to get good at it is to genuinely connect with people — understand their motivations, their skills, and be honest with yourself if it's a fit for your company and what you have to offer.

We've learned by watching and helping thousands of companies hire, and we've codified some of the best practices of hiring and running a recruiting process into this set of skills. We use this tool ourselves at YC, and it has helped us spend more time on talent/fit evaluation, and less time on the mundane operational work.

This tool is NOT intended to replace your judgement as a founder; you still have to do the work to find great talent, interview them and convince them to work with you. And this is certainly not intended for spamming candidates -- on [Work at a Startup](https://www.workatastartup.com), or otherwise. From the data we have, that doesn't work, and it's a waste of your own time.

Lastly, if you think this kind of toolkit is the kind of thing you'd want to build to help 1000s of YC companies hire and scale their teams, YC's own software team is hiring -- scroll to the bottom for open roles and the type of work we do.

## What it does

Talk to Claude Code in natural language and it handles:

- **Pipeline management** — view your full hiring pipeline by stage, with urgency tiers
- **Outreach** — draft personalized cold emails using candidate backgrounds and your outreach template
- **Follow-ups** — draft follow-up emails for candidates who haven't replied
- **Scheduling** — find open calendar slots, create Zoom/in-person events, send invites
- **Triage** — scan inbox, WAAS, and Ashby for what needs attention
- **Candidate lookup** — cross-reference WAAS, Ashby, Gmail, and Calendar for any candidate
- **Inbox cleanup** — suggest safe-to-archive emails
- **Applicant review** — review new inbound applicants with recommendations
- **Rejections** — draft personalized rejection emails calibrated to pipeline stage

## See it work

These are real examples from a single session. In each example, the quoted text is what the hiring manager typed into Claude Code. Everything after the dash is what the co-pilot did. Names have been changed.

**Morning triage across all systems**

> "triage" — Claude pulled 27 inbox emails, 38 active candidates in the ATS, 50 applicants from Work at a Startup, 2 weeks of calendar, sent mail, and drafts — all in one pass. Categorized everything into scheduling needs, pipeline decisions, new applications, sourcing replies, vendor emails, and noise. Presented a summary with counts and top items, then drilled into whichever category the hiring manager picked.

**Scheduling a phone screen from an inbox reply**

> "Reply to `Candidate Name`, she proposed 8:30-10:30 Thursday. Let's do 9:30-10:00, then have her chat with Jane for the second half hour." — Claude checked the hiring manager's calendar, updated the existing calendar invite to 9:30-10:00, created a separate 10:00-10:30 event with Jane, drafted a reply in the existing Gmail thread confirming the time, and moved the candidate to Phone Screen in the ATS — all from one message.

**Bulk candidate review and rejection**

> "Show me profiles for the people who applied to my open product engineer role. Rate them according to my product-engineer.md file." — Claude pulled full profiles for 8 applicants and rated each on fit with a short summary. The hiring manager said "reject all of these" — Claude drafted 8 personalized rejection messages (standard template with a custom line per person referencing their background), sent all via the Work at a Startup messaging API, and archived each candidate. Took about 2 minutes.

**Closing out a candidate with context**

> "Archive `Candidate Name`. Add a note that she declined — she wants to be on the investing side. I really liked her energy." — Claude found the candidate in the ATS, added structured interview feedback as a note, archived the application, archived in Work at a Startup, and drafted a warm close-out email in the existing Gmail thread. The hiring manager reviewed it, tweaked it, and said "send it."

**Rescheduling an interview across systems**

> "Reschedule `Candidate Name` for Thursday." — Claude checked the hiring manager's calendar, found open slots, and suggested 2:00 PM. Moved the existing calendar invite, confirmed the ATS interview event synced, and sent the candidate an email with the new time — calendar, ATS, and email all updated together.

## Skills

| Skill                | What it does                                     | Invoke with                      |
| -------------------- | ------------------------------------------------ | -------------------------------- |
| `/recruit`           | Router — dispatches to the right skill           | "recruit", "candidates", "go"    |
| `/pipeline`          | Full pipeline status by stage with urgency tiers | "pipeline", "show me everyone"   |
| `/triage`            | Start-of-day inbox + pipeline scan               | "triage", "what needs attention" |
| `/outreach`          | Draft cold outreach to new candidates            | "reach out to X", "email X"      |
| `/reply`             | Reply to candidates in existing threads          | "reply to X", "draft for X"      |
| `/reject`            | Draft rejection email, archive in Ashby          | "reject X", "pass on X"          |
| `/schedule`          | Find availability, batch schedule screens        | "schedule", "when am I free"     |
| `/review-applicants` | Review new inbound applicants                    | "show me applicants"             |
| `/candidate-status`  | Look up a single candidate's full state          | "status of X", "where is X"      |
| `/candidate-brief`   | Quick one-paragraph background brief             | "who is X", "brief me on X"      |
| `/inbox-archive`     | Suggest safe-to-archive emails                   | "clean up inbox"                 |
| `/recruit-watch`     | Background monitor (run on a loop)               | `/loop 10m /recruit-watch`       |

## Setup

### 1. Install Claude Code

https://claude.ai/code

### 2. Install MCP servers

These skills need MCP (Model Context Protocol) servers for external API access:

```bash
# WAAS (Work at a Startup) — for YC founders hiring through WAAS
uv tool install git+https://github.com/yc-software/waas-mcp

# Ashby ATS (optional — for teams using Ashby as their ATS)
uv tool install git+https://github.com/ryankicks/mcp-ashby
```

Google Workspace (Gmail, Calendar) uses Claude's built-in integrations — no install needed.

#### Upgrading MCP servers

When a new version of the MCP server is released, pull the latest and reinstall:

```bash
# If installed from git
uv tool install git+https://github.com/yc-software/waas-mcp --force

# If installed from a local clone
cd /path/to/waas-mcp && git pull origin main
uv tool install -e . --force
```

After reinstalling, restart Claude Code for the new tools to load. You can verify by asking Claude to search for the new tools (e.g. "do you have pipeline_show?").

### 3. Configure MCP servers

Create a `.mcp.json` file in the root of this repo (not in `.claude/`). This is where Claude Code picks up project-scoped MCP servers:

```json
{
  "mcpServers": {
    "waas": {
      "type": "stdio",
      "command": "waas",
      "args": [],
      "env": {}
    },
    "ashby": {
      "type": "stdio",
      "command": "ashby",
      "args": [],
      "env": {
        "ASHBY_API_KEY": "your-ashby-api-key"
      }
    }
  }
}
```

You can use WAAS alone, Ashby alone, or both together. YC's own team uses both — WAAS for sourcing and Ashby for full pipeline tracking.

### 4. Clone this repo

```bash
git clone https://github.com/yc-software/recruiting.git
cd recruiting
```

Claude Code auto-discovers all skills in `.claude/skills/` — just run `claude` from the repo root.

### 5. Configure your settings

Three config files need to be created from their `.example` templates. These are gitignored so your real IDs, emails, and templates stay local.

```bash
# Create all config files from examples
cp .claude/skills/recruit-config/user.md.example .claude/skills/recruit-config/user.md
cp .claude/skills/recruit-config/ashby.md.example .claude/skills/recruit-config/ashby.md
cp .claude/skills/recruit-config/jobs/product-engineer.md.example .claude/skills/recruit-config/jobs/product-engineer.md
```

**`user.md`** — your identity and preferences:

- **Name and email** — used for Gmail queries, calendar lookups, and email signatures
- **Zoom link** — included in calendar invites
- **Office address** — for in-person meeting invites
- **Timezone** — for calendar operations
- **Slack channel and user IDs** — for bot notifications
- **Ashby user ID and feedback form ID** — for interview feedback attribution
- **Interviewer list** — names, emails, and roles for your interview panel
- **Noise sender list** — emails to skip during inbox triage

**`ashby.md`** — your Ashby org config:

- **Source IDs** — how candidates enter your pipeline (WAAS, direct application, Chrome Extension, sourcing vendor)
- **Archive reason IDs** — standardized reasons for archiving candidates

Find these IDs using the Ashby MCP: `candidate_search` to see source fields, or check Ashby Settings → Sources and Archive Reasons.

### 6. Set up your job playbook

The included `product-engineer.md.example` is YC's playbook (anonymized). To create your own:

```bash
# Copy the example and customize
cp .claude/skills/recruit-config/jobs/product-engineer.md.example .claude/skills/recruit-config/jobs/your-role.md
```

Edit with your:

1. **Ashby job ID and pipeline stage IDs** — find yours with the Ashby MCP: `job_list` → `interview_stage_list`
2. **Interview process doc URL** — link to your interview guide (Google Doc, Notion, etc.)
3. **Outreach email template** — your cold outreach with `{name}`, `{PERSONALIZED_HOOK}`, and `{PROPOSED_TIME}` placeholders
4. **Tone and style rules** — how hooks should read, what to emphasize
5. **Candidate evaluation criteria** — hard requirements, strong signals, auto-archive rules
6. **Hiring manager name in the template** — sign-off and intro line

You can have multiple job playbooks — one per open role. All files in `jobs/` (except `.example` files) are gitignored.

### 7. Create secrets file (optional — for Slack)

If you want Slack bot notifications, create `.claude/skills/recruit-config/CLAUDE.local.md` with your bot token:

```markdown
## Slack Bot Token

xoxb-your-slack-bot-token
```

Add `**/CLAUDE.local.md` to your `.gitignore`.

### 8. Authenticate

- **Gmail/Calendar**: Claude Code will prompt for OAuth on first use
- **WAAS**: Run `waas login` to authenticate via browser
- **Ashby**: Set `ASHBY_API_KEY` in MCP config (if using Ashby)

## How it works

```
You (natural language) → Claude Code → Skills (process) → Config (data) → MCP Servers → APIs
```

- **Skills** contain the process — how to triage, draft emails, schedule, etc.
- **Config** contains the data — your email, job IDs, templates, candidate bar
- **MCP Servers** provide API access — WAAS, Ashby, Gmail, Calendar, Slack
- **`user.md`** holds all user-specific values — swap this file to give the skills to someone else

## Repo structure

```
.claude/skills/                   ← Claude Code auto-discovers these
├── recruit/SKILL.md              ← Router — dispatches to the right skill
├── pipeline/SKILL.md             ← Full pipeline view with urgency tiers
├── triage/SKILL.md               ← Start-of-day inbox + pipeline scan
├── outreach/SKILL.md             ← Personalized cold outreach
├── reply/SKILL.md                ← Reply to candidates (any situation)
├── reject/SKILL.md               ← Rejection emails
├── schedule/SKILL.md             ← Calendar scheduling + Ashby sync
├── review-applicants/SKILL.md    ← Review new inbound applicants
├── candidate-status/SKILL.md     ← Single candidate cross-reference lookup
├── candidate-brief/SKILL.md      ← Quick one-paragraph candidate brief
├── inbox-archive/SKILL.md        ← Safe-to-archive email suggestions
├── recruit-watch/SKILL.md        ← Background monitor (run on a loop)
└── recruit-config/               ← Shared config
    ├── user.md.example           ← Template — copy to user.md and fill in your details
    ├── ashby.md                  ← Ashby org config (sources, archive reasons, email rules)
    ├── waas.md                   ← WAAS pipeline stages and MCP tools
    ├── resolve-candidate.md      ← How to resolve a candidate across systems
    ├── playbook.md               ← Full recruiting playbook (email templates, calendar templates, rules)
    ├── README.md                 ← Architecture and setup docs
    └── jobs/
        └── product-engineer.md   ← YC's Product Engineer playbook (template, bar, IDs)
```

## Requirements

- [Claude Code](https://claude.ai/code)
- [Work at a Startup](https://www.workatastartup.com) account (YC founders)
- Google Workspace (Gmail + Calendar)
- [uv](https://docs.astral.sh/uv/) for MCP server installation
- Optional: [Ashby](https://ashbyhq.com) account with API access (for full ATS pipeline tracking)
- Optional: Slack workspace with a bot

## About YC's Engineering Team

Y Combinator has a small software team that builds everything YC runs on — from the application platform that processes 30,000+ startup applications per batch, to the tools founders use during and after the batch, to the systems that manage $6B+ in fund assets.

We're 18 engineers, no PMs, no designers. Every engineer owns their product end-to-end — from understanding the problem to designing the UI to shipping the system. We work directly with YC partners on what to build.

We use AI heavily across the YC lifecycle — from helping founders get customers and hire, to powering how we make investment decisions. Engineers here get unlimited AI tokens and sit in on batch events, Demo Day, and founder dinners.

**What makes it different:**

- You build software that thousands of startups depend on every day
- You work with 20 years of startup data — the richest dataset in the startup world
- You attend YC events and stay connected to founders building the future
- Compensation: $250K-$500K base + carry in the fund (like startup equity)
- In-person in San Francisco (Dogpatch)

**Open roles:**

- [Product Engineer, Applications & Operations](https://www.ycombinator.com/companies/y-combinator/jobs/MsFL1rP-product-engineer-applications-operations)
- [Product Engineer, Post Batch](https://www.ycombinator.com/companies/y-combinator/jobs/fK75gxxbq-product-engineer-post-batch)
- [Software Product Design Engineer](https://www.ycombinator.com/companies/y-combinator/jobs/9yNWf1G-software-product-design-engineer)
