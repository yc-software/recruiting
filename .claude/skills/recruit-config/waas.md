# WAAS — Pipeline Config

WAAS pipeline stages and job IDs for the MCP tools.

## Pipeline Stages (same for all jobs)

| Stage | Name        | Description                         |
| ----- | ----------- | ----------------------------------- |
| 0     | In Review   | New applicant, not yet reviewed     |
| 1     | Reached Out | Company has messaged the candidate  |
| 2     | Screen      | Phone screen / initial call         |
| 3     | Interview   | Technical screen / deeper interview |
| 4     | Offer       | Offer stage                         |

**IMPORTANT**: When moving a candidate in WAAS, ALWAYS set both `state` AND `pipeline_stage`:

```
candidate_status_update(short_id, state: "screen", pipeline_stage: "Screen")
```

Setting only `state` updates the company-level status but doesn't move them in the per-job kanban pipeline. Setting `pipeline_stage` moves them in the UI.

### State ↔ Pipeline Stage mapping

| Action            | state          | pipeline_stage       |
| ----------------- | -------------- | -------------------- |
| Mark as reviewing | `reviewing`    | `In Review`          |
| Reached out       | `reviewing`    | `Reached Out`        |
| Move to screen    | `screen`       | `Screen`             |
| Move to interview | `interviewing` | `Interview`          |
| Move to offer     | `offer`        | `Offer`              |
| Archive           | `archived`     | (no pipeline change) |

## Active Jobs

| Job                                           | ID    | Notes                |
| --------------------------------------------- | ----- | -------------------- |
| Product Engineer, Post Batch                  | 41302 | Main PE role         |
| Product Engineer, Applications Operations     | 92543 | Apps & Ops PE role   |
| Software Product Design Engineer              | —     | Design engineer role (may not be on WAAS) |
| Investment Associate & Product Engineer       | 87931 | IA+PE combo role     |

## Job Postings (create / edit)

Create/edit WAAS job postings with `job_create` / `job_update` (read one first with `job_show`). Requires the `waas:jobs:manage` scope — on a 403, run `waas login`.

**A posting must be complete to save** (the API rejects partial ones): `title`, `description`, `role`, the role's subtype, `job_type`, `remote`, `us_work_authorization_required`, seniority, and — unless remote — a location. `job_create` defaults to `state: hidden` (draft); set `state: visible` to publish. `job_update` is a partial update.

### Enum values (send the key, not the label)

**role** — `eng` · `design` · `product` · `science` · `sales` · `marketing` · `support` · `operations` · `recruiting` · `finance` · `legal`

**subtype** — an array, required (≥1) only for four roles; the other seven take none. The field name depends on `role`:

| role | subtype field | values |
|------|---------------|--------|
| eng | `eng_type` | android, be, data_sci, devops, embedded, eng_mgmt, fe, fs, ios, ml, qa, robotics, hw, electrical, mechanical, bio, chemical |
| design | `design_type` | web, mobile, product, ui_ux, user_research, brand_graphic, illustration, animation, hardware, ar_vr, design_mgmt |
| science | `science_type` | bio, biotech, chem, genetics, health, immuno, lab, onc, pharma, process, research |
| recruiting | `recruiting_type` | sourcer, recruiter, coordinator, lead, operations, fullcycle, manager |
| product / sales / marketing / support / operations / finance / legal | (none) | — |

**job_type** — `fulltime` · `cofounder` · `intern` · `contract`

**seniority** — `min_experience` (integer years) for non-intern roles, OR `min_school_year` for `job_type: intern` (`any` · `freshman` · `sophomore` · `junior` · `senior`)

**remote** — `yes` (remote ok) · `no` (in-person — REQUIRES `locations`) · `only` (remote only)

**us_work_authorization_required** — boolean. `true` = candidate must already be US-authorized (no sponsorship); `false` = not required / sponsorship OK.

**pay_period** — `year` · `month` · `hour` (default year) · **currency** — `USD` · `CAD` · `INR` · `EUR` · `GBP`

**state** — `hidden` (draft) · `visible` (published). `deleted` is not settable here.

**skills** — free-text names (e.g. "Ruby", "React"), resolved per-role and case-insensitively; unmatched names are silently dropped — check the echoed `skills` in the response. **locations** — free-text strings, e.g. "San Francisco, CA, USA". **equity_min/equity_max** — percent, 0–100 (0.5 = 0.5%). **time_to_hire** — days.

**Gotchas:** changing `role` on an update re-scopes skills to the new role (old-role skills drop); `description` must not contain an email address; the response echoes `role_type` (unified subtype), resolved `skills`, and `us_work_authorization_required` so you can confirm what was accepted.

## MCP Tools

Use `mcp__waas__*` tools:

### Read tools

- `mcp__waas__applicant_list` — list/filter applicants (by state, needs_response, job_id, since, cursor)
- `mcp__waas__candidate_show` — single candidate profile by short_id (positions, educations, work auth)
- `mcp__waas__candidate_batch` — batch lookup up to 25 candidates by comma-separated short_ids
- `mcp__waas__candidate_status_show` — pipeline status (state, pipeline_stage, archive_reason, messaging timestamps)
- `mcp__waas__candidate_messages_list` — all messages between company and candidate
- `mcp__waas__candidate_notes_list` — all internal notes on a candidate
- `mcp__waas__job_list` — list company's jobs with pipeline stages (id, title, state, stage names)
- `mcp__waas__job_show` — full editable detail for one job posting by id (role, subtypes, comp, equity, skills, locations) — read before editing
- `mcp__waas__pipeline_show` — full pipeline board for a job — all stages with candidates (short_id, name, entered_at, state, needs_response)
- `mcp__waas__health_check` — validate connection (returns ok/expired/error)

### Write tools

- `mcp__waas__candidate_status_update` — update state + pipeline_stage (requires `waas:stages:manage`)
- `mcp__waas__candidate_message_send` — send message to candidate (requires `waas:messages:manage`)
- `mcp__waas__candidate_note_create` — add internal note to candidate (requires `waas:notes:manage`)
- `mcp__waas__pipeline_move` — bulk-move candidates to a pipeline stage for a job (requires `waas:stages:manage`)
- `mcp__waas__candidate_create` — add a new candidate to a job's pipeline with optional resume upload from a local file path. The candidate will only be visible to your company. Requires a real pipeline stage (e.g. "In Review") — "Applied" is not a stage, it's a virtual view of candidates who applied but haven't been placed yet. (requires `waas:candidates:manage`)
- `mcp__waas__job_create` — create a job posting (defaults to `state: hidden` draft; set `state: visible` to publish). See "Job Postings" below for the required fields and enum values. (requires `waas:jobs:manage`)
- `mcp__waas__job_update` — edit a job posting (partial — send only changed fields). (requires `waas:jobs:manage`)

### Usage patterns

- **Messages list is the inbound-classification source of truth**: Before drafting any outreach for a WAAS candidate, run `candidate_messages_list`. Any `from_candidate: true` message means this is a reply, not cold outreach — even when Ashby shows `sourceType: Sourced` and there is no WAAS notification email in Gmail. WAAS direct-messages do not generate Gmail notifications and do not set Ashby's `appliedViaJobPostingId`, so they look like pure sourcing unless you check messages. See `resolve-candidate.md` Step 0 for the full Applied vs. Direct-messaged vs. Sourced classification.
- **Pre-flight**: Run `health_check` before any skill that depends on WAAS data
- **Compact mode for triage**: Always use `applicant_list(compact: true)` for first-pass scanning (triage, pipeline, review-applicants). Returns ~60% smaller payloads that fit within tool result limits. Use full mode (no compact) only when you need email addresses, full work history, or complete looking_for text for a deep dive.
- **Batch lookups**: Use `candidate_batch(short_ids)` instead of multiple `candidate_show` calls (max 25)
- **WAAS as messaging channel**: Use `candidate_message_send` for WAAS-only candidates instead of Gmail — messages appear as emails to the candidate
- **Dual-system sync**: When a candidate is in both WAAS and Ashby, update both systems (state in WAAS, stage in Ashby)
- **Moving candidates**: Use `pipeline_move` to move candidates between pipeline stages — it works for all candidates. `candidate_status_update` can also update state (reviewing, archived, etc.) on any candidate.

### Replying to WAAS candidate emails via Gmail

**IMPORTANT:** WAAS notification emails (e.g. "{Name} sent you a message") come FROM `workatastartup@ycombinator.com`, but replying to that address does NOT route to the candidate. The actual reply-to address is an `@inbound.ycombinator.com` address embedded in the email body as a mailto link.

When replying to a WAAS candidate email via Gmail:
1. Read the full email body using `gmail_get_message`
2. Extract the `@inbound.ycombinator.com` address from the mailto link (pattern: `bf-j-*@inbound.ycombinator.com`)
3. Send the reply TO that inbound address, NOT to `workatastartup@ycombinator.com`
4. Use the original email's `threadId` to keep it threaded

If the `@inbound.ycombinator.com` address cannot be found, fall back to `candidate_message_send` via the WAAS MCP tool instead.
