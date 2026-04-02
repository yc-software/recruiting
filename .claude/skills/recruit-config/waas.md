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

## MCP Tools

Use `mcp__waas__*` tools:

### Read tools

- `mcp__waas__applicant_list` — list/filter applicants (by state, needs_response, job_id, since, limit, offset)
- `mcp__waas__candidate_show` — single candidate profile by short_id (positions, educations, work auth)
- `mcp__waas__candidate_batch` — batch lookup up to 25 candidates by comma-separated short_ids
- `mcp__waas__candidate_status_show` — pipeline status (state, pipeline_stage, archive_reason, messaging timestamps)
- `mcp__waas__candidate_messages_list` — all messages between company and candidate
- `mcp__waas__candidate_notes_list` — all internal notes on a candidate
- `mcp__waas__health_check` — validate connection (returns ok/expired/error)

### Write tools

- `mcp__waas__candidate_status_update` — update state + pipeline_stage (requires `waas:stages:manage`)
- `mcp__waas__candidate_message_send` — send message to candidate (requires `waas:messages:manage`)
- `mcp__waas__candidate_note_create` — add internal note to candidate (requires `waas:notes:manage`)

### Usage patterns

- **Pre-flight**: Run `health_check` before any skill that depends on WAAS data
- **Compact mode for triage**: Always use `applicant_list(compact: true)` for first-pass scanning (triage, pipeline, review-applicants). Returns ~60% smaller payloads that fit within tool result limits. Use full mode (no compact) only when you need email addresses, full work history, or complete looking_for text for a deep dive.
- **Batch lookups**: Use `candidate_batch(short_ids)` instead of multiple `candidate_show` calls (max 25)
- **WAAS as messaging channel**: Use `candidate_message_send` for WAAS-only candidates instead of Gmail — messages appear as emails to the candidate
- **Dual-system sync**: When a candidate is in both WAAS and Ashby, update both systems (state in WAAS, stage in Ashby)
