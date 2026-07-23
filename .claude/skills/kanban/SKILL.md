---
name: kanban
description: Use when the user invokes /kanban or asks to create, start, move, review, complete, close, or check project tickets using obsidian/kanban/context/kanban-flow.md, obsidian/kanban/work-kanban-board.md, and obsidian/kanban/tasks/WORK-XXXX.md ticket files. Supports creating tickets from a Markdown marketing or product requirement file.
metadata:
  short-description: Manage project Kanban tickets
---

# Kanban

Use this skill to operate the repo's local ticket workflow.

Always read `obsidian/kanban/context/kanban-flow.md` before changing ticket files. That file is the source of truth for board structure, gates, status movement, and required notes.

Use `prd.md` as the product-level source of truth for project scope, product goals, target users, non-goals, success metrics, and SEO/AI requirements. Read it before creating tickets, planning tickets, starting feature work from backlog, or reviewing product-facing / SEO-impacting tickets.

## Configuration

Before executing any command, read config from `.kanban.json` in the project root (if it exists):

```json
{
  "provider": "local",
  "workspaceAiUrl": "https://your-workspace-ai.example.com",
  "workspaceAiToken": "your-sanctum-token"
}
```

Config can also be provided through environment variables (take precedence over `.kanban.json`):

- `KANBAN_PROVIDER` — `local`, `workspace-ai`, or `lark`
- `WORKSPACE_AI_URL` — base URL of the workspace-ai app
- `WORKSPACE_AI_TOKEN` — Sanctum API token

**Provider routing:**

- `local` (default): use local `.md` files for all board operations. Run workspace-ai artisan commands when available in this project.
- `workspace-ai`: route board reads/writes through the workspace-ai REST API. Requires `WORKSPACE_AI_URL` and `WORKSPACE_AI_TOKEN`. If either is missing, stop and show the setup guide below.
- `lark`: route ticket operations through the Lark task API. Requires `lark-cli` to be installed and authenticated.

**Missing config error — show this message and stop when `workspace-ai` is selected but URL or token is absent:**

```txt
KANBAN_PROVIDER is set to "workspace-ai" but required config is missing.

Add a .kanban.json file to your project root:

{
  "provider": "workspace-ai",
  "workspaceAiUrl": "https://your-workspace-ai.app",
  "workspaceAiToken": "your-sanctum-token"
}

Or set environment variables:
  export KANBAN_PROVIDER=workspace-ai
  export WORKSPACE_AI_URL=https://your-workspace-ai.app
  export WORKSPACE_AI_TOKEN=your-sanctum-token

Generate a token at: <workspaceAiUrl>/settings/tokens
```

**Missing config error — show this message and stop when `lark` is selected but `lark-cli` is missing:**

```txt
KANBAN_PROVIDER is set to "lark" but lark-cli is not installed or authenticated.

Install and authenticate:
  lark-cli auth login --scope "task:task:write task:tasklist:read task:section:read task:section:write"

Also ensure LARK_TASKLIST_GUID is set in your project environment.
```

If `.kanban.json` is missing and no environment variables are set, default to `provider: local` — do not error.

## Response Language

Respond to the user in Thai by default for every Kanban interaction, including help output, status summaries, review results, blockers, implementation updates, and completion notes.

Use another language only when the user explicitly asks for it. Keep fixed workflow keywords, file paths, commands, ticket IDs, status values, and verdict values exactly as written in the flow, for example `Status:`, `PASS`, `NEEDS FIX`, `BLOCKED`, `Review Notes`, and `Done Notes`.

## Command Patterns

Treat these user messages as Kanban commands:

```txt
/kanban create WORK-XXXX
/kanban create WORK-XXXX from path/to/requirement.md
/kanban plan WORK-XXXX
/kanban start WORK-XXXX
/kanban move-review WORK-XXXX
/kanban review WORK-XXXX
/kanban done WORK-XXXX
/kanban close WORK-XXXX
/kanban status WORK-XXXX
/kanban sync
/kanban sync WORK-XXXX
/kanban talk WORK-XXXX
/kanban talk WORK-XXXX - WORK-YYYY
/kanban talk WORK-XXXX, WORK-YYYY, WORK-ZZZZ
/kanban talk WORK-XXXX --lang=en
/kanban lark status
/kanban lark enable
/kanban lark disable
/kanban lark sync WORK-XXXX
/kanban sprint close
/kanban sprint close --name=splint-3
/kanban api get-backlog
/kanban help
```

Also trigger this skill for natural language requests such as "ทำ Ticket WORK-XXXX", "สร้าง ticket จาก requirement file", or "ย้าย ticket ไป done".

## Help

For `/kanban help`, show a concise command reference and examples without changing any files.

Include these commands:

```txt
/kanban help
  Show basic commands and usage examples.

/kanban create WORK-XXXX
  Create a new ticket file and add it to todo.

/kanban create WORK-XXXX from path/to/requirement.md
  Convert a Markdown marketing/product requirement file into a ticket.

/kanban status WORK-XXXX
  Show ticket status, board location, source/spec references, and blockers.

/kanban sync
  Pull all tickets from the workspace-ai API and write them to obsidian/kanban/tasks/. Prompts before overwriting a local file that is newer than the server version.

/kanban sync WORK-XXXX
  Pull a single ticket from the workspace-ai API. Prompts before overwriting if the local file is newer.

/kanban talk WORK-XXXX
  Use the management-talk skill to rewrite the ticket for marketing, content, SEO, or management readers.

/kanban talk WORK-XXXX - WORK-YYYY
  Rewrite an inclusive ticket range in numeric order from the first ID to the second ID.

/kanban talk WORK-XXXX, WORK-YYYY, WORK-ZZZZ
  Rewrite multiple tickets in the exact queue order provided.

/kanban talk WORK-XXXX --lang=en
  Rewrite marketing tickets in English instead of the default Thai.

/kanban lark status
  Show whether optional Lark Ticket Command sync is enabled and whether local setup appears ready.

/kanban lark enable
  Enable optional Lark Ticket Command sync for future kanban create/status-move commands.

/kanban lark disable
  Disable optional Lark Ticket Command sync and continue using only local Markdown tickets.

/kanban lark sync WORK-XXXX
  Backfill an existing local ticket into Lark without moving its local board status.

/kanban plan WORK-XXXX
  Analyze planning needs, ask Yes/No to continue, then plan and move the ticket to Plan.

/kanban start WORK-XXXX
  Start a ticket, move it to in-progress, and begin implementation. Backlog tickets may skip Plan when ready.

/kanban move-review WORK-XXXX
  Add review notes and move the ticket to review after validation passes.

/kanban review WORK-XXXX
  Run the final review gate, using scrutinize when risk warrants it, and report PASS, NEEDS FIX, or BLOCKED.

/kanban done WORK-XXXX
  Move the ticket to done after all required gates pass. For meaningful bugfixes, run post-mortem first when root cause should be preserved.

/kanban close WORK-XXXX
  Close a canceled, duplicate, obsolete, or out-of-scope ticket. Preserve post-mortem details first when closing a fixed bug with known root cause.

/kanban sprint close
  Close the current sprint: archive done/closed tickets into a new splint-X file, carry forward unfinished tickets in work-kanban-board.md.

/kanban sprint close --name=splint-3
  Same as above but use a specific name for the new sprint archive file.

/kanban api get-backlog
  Pull Backlog cards from workspace-ai API and write them as .md files into obsidian/kanban/specs/backlog/.
  Requires workspaceAiUrl and WORKSPACE_AI_TOKEN to be configured.
```

Also include two short examples:

```txt
/kanban create WORK-1012 from prd.md
/kanban plan WORK-1012
/kanban start WORK-1012
```

## Board Architecture

This project uses two types of board files:

**Active board** — `obsidian/kanban/work-kanban-board.md`
- The single living board for all current work.
- Ticket entries move between `## COLUMN` sections as work progresses.
- Always read this file to find where a ticket currently lives.

**Sprint archive boards** — `obsidian/kanban/splint-X-kanban-board.md`
- Immutable records of closed sprints. Do not modify these files during normal ticket operations.
- Each splint board shows only the tickets that were **completed or closed** during that sprint (`## DONE` and `## CLOSED` sections).
- Tickets that were not finished when the sprint closed are carried forward to `work-kanban-board.md ## BACKLOG`.
- These files exist as historical logs — they answer "what did sprint N deliver?" not "what is in progress now?"

When the user asks to **close a sprint** or **start a new sprint**:

1. Create a new `obsidian/kanban/splint-X-kanban-board.md` (increment X from the latest existing splint number).
2. Copy only the `## DONE` and `## CLOSED` ticket entries from `work-kanban-board.md` into the new splint file.
3. Remove from `work-kanban-board.md` any tickets that were moved into the new splint archive.
4. Leave unfinished tickets (`## BACKLOG`, `## PLAN`, `## INPROGRESS`, `## REVIEW`) in `work-kanban-board.md` — they carry forward automatically.
5. Do not move or rename ticket files in `obsidian/kanban/tasks/` — only board entries change.

The splint file format is minimal (only done/closed sections):

```md
---

kanban-plugin: board

---

## DONE

- [WORK-XXXX](tasks/WORK-XXXX.md): Ticket title


## CLOSED

- [WORK-XXXX](tasks/WORK-XXXX.md): Ticket title

%% kanban:settings
{"kanban-plugin":"board","list-collapse":[false,false]}
%%
```

## Core Rules

- Ticket files live at `obsidian/kanban/tasks/WORK-XXXX.md`.
- The board is a single file at `obsidian/kanban/work-kanban-board.md` with `## COLUMN` sections: `## BACKLOG`, `## PLAN`, `## TODO`, `## INPROGRESS`, `## REVIEW`, `## DONE`, `## CLOSED`.
- Move status by moving the ticket entry between sections in the board file and updating `Status:` inside the ticket file.
- In the first ticket heading, add a type emoji before the ticket ID when the ticket type can be identified: `🟢` for Feature and `🔴` for Bugfix. If the type cannot be identified, leave the emoji blank.
- Do not duplicate the same ticket entry across multiple sections of the board file.
- Do not rename or move the ticket file during normal status changes.
- Never mark a ticket done unless the gates in `obsidian/kanban/context/kanban-flow.md` are satisfied.
- If a requirement is ambiguous, add `### Open Questions` to the ticket instead of inventing risky details.

## Optional Lark Ticket Command Sync

Lark Ticket Command sync is optional. Local `obsidian/kanban/tasks` ticket files and the board file remain the source of truth. Lark sync must never block normal local Kanban work when the user has not installed `lark-cli`, has not authenticated, has disabled sync, or a Lark command fails.

Use `obsidian/kanban/tasks/lark-sync.json` as the small local preference file for this skill:

```json
{
  "enabled": false
}
```

If the file is missing, treat Lark sync as disabled. For `/kanban lark enable`, create or update the file with `"enabled": true`; this records that the user has confirmed they want Lark sync enabled for future commands. For `/kanban lark disable`, create or update it with `"enabled": false`. For `/kanban lark status`, report the preference, whether `lark-cli` is available, and whether project configuration appears present.

Before running any automatic Lark command:

1. Check `obsidian/kanban/tasks/lark-sync.json`.
2. If sync is disabled or the file is missing, skip all Lark work and continue the normal Kanban command.
3. If sync is enabled, do not re-check `command -v lark-cli` on every `/kanban` command; rely on the previous `/kanban lark status` / `/kanban lark enable` confirmation to reduce token usage.
4. Run the existing Artisan command for the requested Lark operation.
5. If the Artisan command reports that `lark-cli`, auth, or config is missing, explain setup and ask whether to disable sync or skip Lark for this command. Report failures clearly without reverting local Kanban state.

Setup guidance when `lark-cli` is missing or not authenticated:

```sh
lark-cli auth login --scope "task:task:write task:tasklist:read task:section:read task:section:write"
```

The project also needs Lark task configuration available to Laravel, including `LARK_TASKLIST_GUID` and optionally `LARK_TASK_ASSIGNEE_OPEN_ID`. Users can inspect local command usage with:

```sh
php artisan ticket:help
```

Status mapping for automatic move sync:

```txt
Backlog -> backlog
Todo -> todo
In Progress -> inprogress
Review -> review
Done -> done
Closed -> closed
```

Do not run `/kanban lark sync WORK-XXXX` automatically from `/kanban lark enable`. It is an explicit manual backfill command for existing local tickets that are missing from the Lark Kanban Board.

## Manual Lark Sync

For `/kanban lark sync WORK-XXXX`:

1. Read `obsidian/kanban/context/kanban-flow.md`.
2. Check `obsidian/kanban/tasks/lark-sync.json`.
3. If sync is disabled or the file is missing, report that Lark sync is disabled and ask the user to run `/kanban lark enable` before syncing.
4. Read `obsidian/kanban/tasks/WORK-XXXX.md`.
5. Find which board section in `obsidian/kanban/work-kanban-board.md` contains the ticket entry and report the current local `Status:` and board location.
6. Run the equivalent `/kanban talk WORK-XXXX` workflow to create or update `obsidian/kanban/tasks/marketing-tickets/WORK-XXXX.md`.
7. After the marketing ticket exists, run `php artisan ticket:create-lark WORK-XXXX`.
8. Do not move the local board entry.
9. Do not change `Status:` in the local ticket file.
10. If the marketing ticket step or Lark create command fails, report the failure clearly without changing local Kanban state.

## Create Ticket

For `/kanban create WORK-XXXX`:

1. Read `obsidian/kanban/context/kanban-flow.md`.
2. Read `prd.md` unless the ticket is purely internal tooling with no product, content, admin, or SEO impact.
3. If a source file is provided, read it and convert it into implementation-ready ticket requirements.
4. Align the ticket scope, non-goals, acceptance criteria, and validation with the PRD. If the request conflicts with the PRD, add `### Open Questions` and ask whether to update the PRD or treat the request as an exception.
5. Create `obsidian/kanban/tasks/WORK-XXXX.md`.
6. Add the ticket entry to the `## TODO` section of `obsidian/kanban/work-kanban-board.md`.
7. Keep the board entry format from the flow document.
8. Sync the new card based on provider:
   - `local` or unset: run `php artisan kanban:import --ticket=WORK-XXXX` to create the DB card. Report any failure but keep the local ticket in Todo.
   - `workspace-ai`: (1) GET `{workspaceAiUrl}/api/kanban/columns` with header `X-API-Key: {workspaceAiToken}` to find the `id` of the column whose `name` slug matches `todo`. (2) POST `{workspaceAiUrl}/api/kanban/cards` with `X-API-Key: {workspaceAiToken}`, `Content-Type: application/json`, and body `{ "title": "{TICKET}: {title}", "ticket": "{TICKET}", "description": "### Summary\n{summary text}\n\n### Acceptance Criteria\n{ac text}", "priority": "{priority}", "column_id": {todo_column_id} }`. Use plain ticket ID and title without emoji in the API title. Report any failure but keep the local ticket in Todo and do not revert local state.
9. If optional Lark sync is enabled:
   a. Check whether the source spec file contains a `Lark GUID:` line (e.g. `Lark GUID: \`0a42bd30-b84b-4811-a596-22d5a84bb1c0\``). Extract the GUID value (strip backticks).
   b. **If the spec has a Lark GUID** — a Lark task already exists. Do NOT run `ticket:create-lark`. Instead:
      - Store `lark_guid: <GUID>` in the YAML frontmatter of the ticket file.
      - Run `php artisan ticket:rename-lark WORK-XXXX --task-guid=<GUID>` to set the title to the standard format.
      - Run `php artisan ticket:move-lark WORK-XXXX todo` to move the task to the todo section.
   c. **If the spec has no Lark GUID** — run the equivalent `/kanban talk WORK-XXXX` workflow to create or update `obsidian/kanban/tasks/marketing-tickets/WORK-XXXX.md`. After the marketing ticket exists, run `php artisan ticket:create-lark WORK-XXXX`. If the Lark task was created, run `php artisan ticket:rename-lark WORK-XXXX` to set the title format, then run `php artisan ticket:move-lark WORK-XXXX todo`.
10. If any Lark command fails, report the failure clearly and keep the local ticket in Todo.

Ticket content should usually include:

```md
---
ticket_id: WORK-XXXX
title: Ticket title
type: feature
status: todo
priority: medium
spec: path/to/spec.md
tags:
  - kanban/ticket
  - work
---

## 🟢 WORK-XXXX: Ticket title

Status: Todo
Priority: Medium
Source: `path/to/source.md`

### Summary
...

### Context
...

### Scope
- ...

### Acceptance Criteria
- ...

### Validation
- ...
```

**Obsidian Properties (YAML frontmatter) rules:**

- Every ticket file MUST start with a YAML frontmatter block (`---` … `---`) before the first heading.
- `ticket_id`: the ticket ID (e.g. `WORK-1007`).
- `title`: plain-text ticket title without the emoji prefix.
- `type`: `feature` for Feature tickets, `bugfix` for Bugfix tickets, `chore` for internal/tooling work, or `unknown` when the type cannot be determined.
- `status`: lowercase status value — one of `backlog`, `plan`, `todo`, `inprogress`, `review`, `done`, `closed`. Keep this in sync with the `Status:` field in the ticket body whenever the status changes.
- `priority`: lowercase — `high`, `medium`, or `low`. Keep in sync with the `Priority:` field in the ticket body.
- `spec`: path to the spec file if one exists; omit this key when there is no spec.
- `lark_guid`: Lark task GUID if the ticket was linked to an existing Lark task (e.g. from a backlog spec with `Lark GUID:`); omit this key when there is no pre-existing Lark task.
- `tags`: always include `kanban/ticket` and `work`.
- When updating `Status:` in the ticket body (e.g. during `/kanban start`, `/kanban move-review`, `/kanban done`), also update the `status:` frontmatter field to match.

Use `## 🔴 WORK-XXXX: Ticket title` for Bugfix tickets. If the ticket type is unclear, use `## WORK-XXXX: Ticket title` without an emoji.

When converting a marketing or product requirement file:

- Summarize the business intent into implementation language.
- Convert marketing goals into observable acceptance criteria.
- Preserve important brand, SEO, compliance, content, and conversion requirements.
- Split unrelated work into separate proposed tickets if one source file contains multiple independent features.
- Add SEO validation when the ticket affects public pages, visible copy, metadata, structured data, internal links, or content quality.
- Avoid copying a long requirement file verbatim into the ticket.

## Plan Ticket

For `/kanban plan WORK-XXXX`:

1. Read `obsidian/kanban/context/kanban-flow.md`.
2. Read the ticket file.
3. Read `prd.md` when the ticket affects product scope, public pages, admin-managed website content, SEO, structured data, content quality, or user-facing behavior.
4. Read referenced specs, context files, acceptance criteria, and validation requirements.
5. Before changing any files, show a short pre-plan assessment:
   - ticket status and current board section
   - whether `brainstorming` is recommended before writing the plan
   - the reason for the `brainstorming` recommendation
   - whether `writing-plans` is recommended for this ticket
   - the reason for the recommendation
   - whether the ticket aligns with `prd.md`, when PRD context applies
   - what will change if planning continues
6. Ask the user with a Yes/No menu whether to continue planning.
   - Use `request_user_input` when available.
   - Label the recommended option `Yes (Recommended)` when planning should continue.
   - If `request_user_input` is unavailable, ask for a plain `Yes` or `No` response and wait.
7. If the user chooses `No`, stop without changing the ticket file or board file.
8. If required context or acceptance criteria are missing, or the ticket conflicts with the PRD and the conflict is not approved by the user, add or update `### Open Questions` in the ticket and stop without moving the ticket to Plan.
9. If the assessment recommends `brainstorming`, use the `brainstorming` skill as a pre-plan gate to clarify scope, split unrelated work, choose the approach, or resolve risky decisions before writing `### Implementation Plan`.
10. If `brainstorming` surfaces unresolved questions, add or update `### Open Questions` in the ticket and stop without moving the ticket to Plan.
11. If the assessment recommends `writing-plans`, use the `writing-plans` skill to create the implementation plan after the `brainstorming` gate is resolved.
12. After `writing-plans` returns, ask the user: **"Did you choose to execute the plan (Subagent-Driven or Inline Execution) during writing-plans?"** with a Yes/No menu. This determines whether implementation already happened and which status to assign.
    - If the user answers **Yes** (execution happened): set target status to `In Progress` / `inprogress` for steps 14–17. Skip step 18.
    - If the user answers **No** (plan only, no execution): set target status to `Plan` / `plan` for steps 14–17. Apply step 18.
13. If the assessment does not recommend `writing-plans`, create a concise implementation plan directly after the `brainstorming` gate is resolved or deemed unnecessary. Target status is `Plan` / `plan`.
14. Add or update `### Implementation Plan` in the ticket file.
15. Move the ticket entry to the correct board section based on the target status from step 12 or 13:
    - `Plan` → move to `## PLAN`
    - `In Progress` → move to `## INPROGRESS`
16. Change `Status:` and the YAML `status:` frontmatter to match the target status. Both must be updated together in the same edit.
17. Sync based on provider:
    - `local` or unset: run `php artisan kanban:sync --ticket=WORK-XXXX` if available. Report any failure but do not revert.
    - `workspace-ai`: no API call for plan stage — skip.
    - `lark`: skip artisan sync; use Lark commands in the next step instead.
18. If optional Lark sync is enabled (or `KANBAN_PROVIDER=lark`), run `php artisan ticket:move-lark WORK-XXXX <target-status>` after sync, using `plan` or `inprogress` as appropriate. Report any failure without reverting.
19. Do not implement code changes during `plan` unless the user confirmed execution already happened via `writing-plans`.

Recommend `brainstorming` when the ticket has unclear requirements, multiple possible approaches, unresolved `### Open Questions`, likely scope splitting, ambiguous data ownership, PRD tension, or decisions that should not be invented during implementation. Recommend `writing-plans` when the ticket touches multiple files, multiple layers, database changes, public SEO content, security/auth/validation behavior, external integrations, migrations, imports/exports, or has meaningful regression risk. For small copy-only, board-only, documentation-only, or single-file low-risk tickets, recommend planning without `writing-plans`.

The implementation plan should be concise but execution-ready. Include:

```md
### Implementation Plan

- Context to verify:
  - ...
- Planned changes:
  - ...
- Files likely to change:
  - ...
- Validation:
  - ...
- Risks / decisions:
  - ...
```

Scale the resulting `### Implementation Plan` to the ticket size; keep it concise for simple work and more detailed for multi-file, database, public SEO content, or regression-prone work.

## Start Ticket

For `/kanban start WORK-XXXX`:

1. Read the ticket file.
2. Read `prd.md` when the ticket affects product scope, public pages, admin-managed website content, SEO, structured data, content quality, or user-facing behavior.
3. Read referenced specs, context files, and acceptance criteria.
4. If the ticket conflicts with the PRD and the conflict is not approved by the user, add or update `### Open Questions` and stop before moving to `In Progress`.
5. If the ticket has `Status: Plan` and `### Implementation Plan`, continue directly without invoking `writing-plans` again.
6. If the ticket has `Status: Backlog`, it may skip `Plan` and move directly to `In Progress` when all of these are true:
   - acceptance criteria are clear
   - required context/spec references are available or unnecessary
   - there are no unresolved `### Open Questions` or blocking notes
   - the work is small, low-risk, and does not require `writing-plans` under the Plan Ticket recommendation rules
7. If the backlog ticket does not satisfy every skip condition, run the Plan Ticket flow first and stop if planning creates new open questions.
8. Move the ticket entry to the `## INPROGRESS` section of `obsidian/kanban/work-kanban-board.md`.
9. Change `Status:` to `In Progress` in the ticket body **and** update `status: inprogress` in the YAML frontmatter block at the top of the ticket file. Both must be updated together in the same edit.
10. **REQUIRED: Complete steps 8–9 and confirm both files are saved before proceeding to any implementation step. Do not invoke any implementation skill or write any code until the board entry has moved and `Status:` reads `In Progress`.**
11. Sync based on provider:
    - `local` or unset: run `php artisan kanban:sync --ticket=WORK-XXXX` if available. Report any failure but do not revert.
    - `workspace-ai`: call `PATCH {workspaceAiUrl}/api/kanban/cards/{TICKET}` with `X-API-Key: {workspaceAiToken}`, `Content-Type: application/json`, body `{ "column": "inprogress" }`. Report any failure but do not revert local state.
    - `lark`: skip artisan sync; use Lark commands in the next step instead.
12. If optional Lark sync is enabled (or `KANBAN_PROVIDER=lark`), run `php artisan ticket:move-lark WORK-XXXX inprogress` immediately after the local board and ticket status are updated, before implementation work begins. Report any Lark failure without reverting the local move.
13. If the ticket is a bug report or regression, use the `debug-mantra` skill before proposing or implementing a fix. If the user asks to skip the recital, still apply its reproduce -> trace -> falsify -> breadcrumb workflow silently.
14. After implementation, check whether this ticket changed database migrations or seeders.
15. If this ticket created or modified database migration files, run `php artisan migrate --force` before reporting completion.
16. If this ticket created or modified a seeder, run `php artisan db:seed --class=NewSeeder --force`, replacing `NewSeeder` with the actual seeder class name.
17. If the seeder command from the previous step fails because old database records already exist and the seed data collides or overwrites existing data, explain the cause, split the new seed data into a separate seeder file, then run `php artisan db:seed --class=NewSeparateSeeder --force` with the new seeder class.
18. Run the ticket's required validation after any migration or seeding commands that apply.
19. After implementation and validation, report the result, whether the ticket appears ready for review, and what commands the user should run when deploying this change to Hostinger. Include `php artisan migrate --force` when migrations changed, include the exact `php artisan db:seed --class=... --force` command when seeders changed, and mention when neither command is required.
20. Do not move the ticket to Review automatically during `start`, even when implementation and validation pass.
21. Do not run `php artisan ticket:move-lark WORK-XXXX review` during `start`.
22. Wait for an explicit `/kanban move-review WORK-XXXX` command before adding `Review Notes`, moving the ticket entry to the `## REVIEW` section, changing `Status:` to `Review`, or syncing Lark to `review`.

Do not invoke `writing-plans` again during `start` when the ticket already has `Status: Plan` and `### Implementation Plan`.

If acceptance criteria or required context is missing, stop and ask for the missing information.

## Move To Review

For `/kanban move-review WORK-XXXX`:

1. Confirm implementation and required validation have passed.
2. Add or update `### Review Notes` in the ticket file.
3. Move the ticket entry to the `## REVIEW` section of `obsidian/kanban/work-kanban-board.md`.
4. Change `Status:` to `Review` in the ticket body **and** update `status: review` in the YAML frontmatter block at the top of the ticket file. Both must be updated together in the same edit.
5. Sync based on provider:
    - `local` or unset: run `php artisan kanban:sync --ticket=WORK-XXXX` if available. Report any failure but do not revert.
    - `workspace-ai`: call `PATCH {workspaceAiUrl}/api/kanban/cards/{TICKET}` with `X-API-Key: {workspaceAiToken}`, `Content-Type: application/json`, body `{ "column": "review" }`. Report any failure but do not revert local state.
    - `lark`: skip artisan sync; use Lark commands in the next step instead.
6. Create the pull request based on provider:
   - `local`: run `php artisan kanban:create-pr WORK-XXXX` to create the pull request. Report any failure without reverting.
   - `workspace-ai`: run `gh pr create` using the GitHub CLI. Use `--base main` (or the repo's default branch), title `{TICKET}: {ticket title}` (without emoji), and a body that includes the ticket summary, acceptance criteria, and a link to `obsidian/kanban/tasks/WORK-XXXX.md`. After the PR is created, call `PATCH {workspaceAiUrl}/api/kanban/cards/{TICKET}` with `X-API-Key: {workspaceAiToken}`, `Content-Type: application/json`, body `{ "pr_url": "{pr_url}" }` to link the PR URL to the card. If `gh` is not installed or not authenticated, report the error and skip PR creation without reverting local state.
   - `lark`: if `gh` is available and the repo has a GitHub remote, run `gh pr create` with the same title/body format as above. Report any failure without reverting.
7. If optional Lark sync is enabled (or `KANBAN_PROVIDER=lark`), run `php artisan ticket:move-lark WORK-XXXX review` after the local board and ticket status are updated. Report any Lark failure without reverting the local move.

## Review Ticket

For `/kanban review WORK-XXXX`:

1. Read `obsidian/kanban/context/kanban-flow.md`.
2. Read `prd.md` when the ticket affects product scope, public pages, admin-managed website content, SEO, structured data, content quality, or user-facing behavior.
3. Use the `scrutinize` skill when the plan, diff, or code change has regression risk, cross-layer impact, important behavior changes, or the user asked for review/audit/sanity-check/second opinion.
4. Use the `fullstack-guardian` skill when the ticket has full-stack, security, auth, validation, form submission, database write, upload, CRUD, route contract, model relationship, policy, middleware, Filament resource/action, or high regression impact.
5. If the ticket is low-risk and does not need `scrutinize` or `fullstack-guardian`, run the lightweight review described in `obsidian/kanban/context/kanban-flow.md` and record why those gates were skipped.
6. Review against the acceptance criteria, PRD alignment when applicable, and gates in `obsidian/kanban/context/kanban-flow.md`.
7. Report `PASS`, `NEEDS FIX`, or `BLOCKED`.
8. If fixes are required, implement them before moving the ticket forward. Use `debug-mantra` for new bug/regression root-cause work and `systematic-debugging` for unclear test/build/runtime failures.

## Done Ticket

For `/kanban done WORK-XXXX`:

1. Confirm all required gates have passed.
2. If this is a meaningful bugfix and reliable repro, known root cause, identified fix, and validation are all present, use the `post-mortem` skill before moving to Done when the root cause should be preserved.
3. Add or update `### Done Notes`, including which gates ran or were skipped and why.
4. Move the ticket entry to the `## DONE` section of `obsidian/kanban/work-kanban-board.md`.
5. Change `Status:` to `done` in the ticket body **and** update `status: done` in the YAML frontmatter block at the top of the ticket file. Both must be updated together in the same edit.
6. Sync based on provider:
    - `local` or unset: run `php artisan kanban:sync --ticket=WORK-XXXX` if available. Report any failure but do not revert.
    - `workspace-ai`: call `PATCH {workspaceAiUrl}/api/kanban/cards/{TICKET}` with `X-API-Key: {workspaceAiToken}`, `Content-Type: application/json`, body `{ "column": "done", "completed": true }`. Report any failure but do not revert local state.
    - `lark`: skip artisan sync; use Lark commands in the next step instead.
7. If optional Lark sync is enabled (or `KANBAN_PROVIDER=lark`), run `php artisan ticket:move-lark WORK-XXXX done` after the local board and ticket status are updated. Report any Lark failure without reverting.

## Close Ticket

For `/kanban close WORK-XXXX`:

1. Add or update `### Closed Notes` with the reason.
2. If closing a fixed bug that has reliable repro, known root cause, identified fix, and validation, use the `post-mortem` skill first when the root cause should be preserved.
3. Move the ticket entry to the `## CLOSED` section of `obsidian/kanban/work-kanban-board.md`.
4. Change `Status:` to `closed` in the ticket body **and** update `status: closed` in the YAML frontmatter block at the top of the ticket file. Both must be updated together in the same edit.
5. If `KANBAN_PROVIDER=workspace-ai`: call `PATCH {workspaceAiUrl}/api/kanban/cards/{TICKET}` with header `X-API-Key: {workspaceAiToken}`, `Content-Type: application/json`, body `{ "column": "closed" }`. On failure, report error without reverting local state.
6. If optional Lark sync is enabled, run `php artisan ticket:move-lark WORK-XXXX closed` after the local board and ticket status are updated. Report any Lark failure without reverting the local move.

Use this for canceled, obsolete, duplicate, or out-of-scope tickets.

## Sprint Close

For `/kanban sprint close` or `/kanban sprint close --name=splint-X`:

1. Read `obsidian/kanban/work-kanban-board.md` to collect all entries in `## DONE` and `## CLOSED`.
2. If neither section has any entries, report that there is nothing to archive and stop.
3. Determine the new splint file name:
   - If `--name=` is provided, use that value (e.g. `splint-3`).
   - Otherwise, scan existing `obsidian/kanban/splint-*.md` files, find the highest number, and increment by 1.
4. Create `obsidian/kanban/splint-X-kanban-board.md` with only the done and closed entries from step 1. Use the minimal splint format described in **Board Architecture**.
5. Remove the archived ticket entries from `## DONE` and `## CLOSED` in `work-kanban-board.md`. Leave all other sections (`## BACKLOG`, `## PLAN`, `## TODO`, `## INPROGRESS`, `## REVIEW`) untouched — unfinished tickets carry forward automatically.
6. Do not move or rename any ticket files in `obsidian/kanban/tasks/`.
7. Do not change `Status:` inside any ticket file.
8. Report: the new splint file created, which tickets were archived, and which tickets remain on the active board.

## Sync Tickets

For `/kanban sync` and `/kanban sync WORK-XXXX`:

**Prerequisites:** Requires `KANBAN_PROVIDER=workspace-ai` (or `provider: workspace-ai` in `.kanban.json`) with `WORKSPACE_AI_URL` and `WORKSPACE_AI_TOKEN` set. If config is missing, show the setup guide from the Configuration section and stop.

Workspace AI only:

0. Do not call `GET {WORKSPACE_AI_URL}/api/kanban/tickets/{ticket}/download` during sync. Local `.md` ticket files are the source for card content; sync only card existence, project, and status through the API.
1. Before local file sync, check the remote Kanban board with `GET {WORKSPACE_AI_URL}/api/kanban/tickets/{ticket}` using `X-API-Key: {WORKSPACE_AI_TOKEN}`.
2. If the API returns 404 or no card for `{ticket}`, create it with the Kanban Create Card API: `POST {WORKSPACE_AI_URL}/api/kanban/cards` using `X-API-Key: {WORKSPACE_AI_TOKEN}` and `Content-Type: application/json`.
   Include `ticket`, current title if known, current project, and the mapped current status.
3. If the API confirms `{ticket}` already exists, sync it with the user's current project and update the remote status with `PATCH {WORKSPACE_AI_URL}/api/kanban/cards/{ticket}` using `X-API-Key: {WORKSPACE_AI_TOKEN}` and `Content-Type: application/json`.
   Include the current project and mapped current status in the payload. Do not download or overwrite local files.
4. Apply this remote board existence/create/update flow only when provider is `workspace-ai`. Do not run it for `local` or `lark`.

### Column-to-section mapping (case-insensitive)

When mapping a column name from the API response to a board section, normalize the column value to lowercase and strip spaces before matching:

| Normalized column | Board section |
|---|---|
| `backlog` | `## BACKLOG` |
| `plan` | `## PLAN` |
| `todo` | `## TODO` |
| `inprogress` | `## INPROGRESS` |
| `review` | `## REVIEW` |
| `done` | `## DONE` |
| `closed` | `## CLOSED` |

Example: `"In Progress"` → strip spaces → `"InProgress"` → lowercase → `"inprogress"` → `## INPROGRESS`.

### Full sync: `/kanban sync`

1. Read config from `.kanban.json` or env. Show setup guide and stop if `WORKSPACE_AI_URL` or `WORKSPACE_AI_TOKEN` is missing.
2. Fetch `GET {WORKSPACE_AI_URL}/api/kanban/tickets` with header `X-API-Key: {WORKSPACE_AI_TOKEN}`. Report and stop on HTTP error.
3. The response is a JSON array. Each item has: `ticket` (e.g. `WORK-1033`), `title`, `column` (column name = status).
   - **Sub-tasks:** The API currently filters `whereNull('parent_id')` and does not return sub-task cards in this list. If the server adds a `?include_subtasks=true` query param in the future, append it to include sub-tasks. Until the API supports it, note in the sync summary that sub-task cards are excluded and the server-side endpoint must be updated to expose them.
4. For each ticket in the response, handle failures gracefully: if any per-ticket step fails (network error, unexpected response, or 404 on download), skip that ticket, record it in the failed list, and continue with the remaining tickets. Do not let a single ticket failure stop the entire sync.
   a. Check whether `obsidian/kanban/tasks/{ticket}.md` exists locally.
   b. If it does NOT exist: download and write it (step 5 below). No prompt needed.
   c. If it DOES exist: read the local file's `updated_at` or last git commit time. Compare with the server's `Last-Modified` response header (or fall back to downloading and comparing content hashes).
      - If local file content equals server content: skip (already in sync).
      - If server is newer: download and overwrite.
      - If local is newer or the same age but content differs: ask the user "Local `{ticket}.md` has local changes. Overwrite with server version? (Yes / Skip)" and wait for a response before overwriting.
5. To download a ticket: fetch `GET {WORKSPACE_AI_URL}/api/kanban/tickets/{ticket}/download` with `X-API-Key: {WORKSPACE_AI_TOKEN}`.
   - If the response is 404: the server has no `.md` file for this ticket. Skip it, add to skipped list with reason "no file on server", and continue.
   - On other HTTP error: skip ticket, add to failed list with the status code, and continue.
   - On success: write the response body to `obsidian/kanban/tasks/{ticket}.md`.
6. After writing all ticket files, update `obsidian/kanban/work-kanban-board.md`:
   - For each downloaded ticket, normalize the column name using the case-insensitive mapping above to determine the target board section.
   - If the ticket's board entry is in the wrong section (status mismatch), move it to the correct section. Report the move.
   - If no board entry exists, add one to the correct section: `- [{ticket}](tasks/{ticket}.md): {title}`.
   - Do not duplicate entries.
7. Report a summary: tickets downloaded, tickets skipped (already in sync), tickets skipped (no file on server), tickets failed, and tickets with local changes that were skipped or overwritten. If sub-tasks were excluded, note that in the summary.

### Single ticket sync: `/kanban sync WORK-XXXX`

1. Read config. Show setup guide and stop if config is missing.
2. Fetch `GET {WORKSPACE_AI_URL}/api/kanban/tickets/WORK-XXXX/download` with `X-API-Key: {WORKSPACE_AI_TOKEN}`.
   - On 404: report "Ticket WORK-XXXX not found on server." and stop.
   - On other HTTP error: report the error and stop.
3. Check whether `obsidian/kanban/tasks/WORK-XXXX.md` exists locally.
   - Does NOT exist: write the downloaded content. Continue to step 4.
   - DOES exist and content matches: report "Already in sync." and stop.
   - DOES exist and content differs: ask "Local `WORK-XXXX.md` has local changes. Overwrite with server version? (Yes / Skip)". If Yes, overwrite. If Skip, stop.
4. After writing, fetch `GET {WORKSPACE_AI_URL}/api/kanban/tickets/WORK-XXXX` with `X-API-Key: {WORKSPACE_AI_TOKEN}` to get the current column/status from the API.
5. Normalize the column name using the case-insensitive mapping above and update `obsidian/kanban/work-kanban-board.md` to ensure the ticket entry is in the correct section.
6. Report: downloaded or already in sync.

## Status

For `/kanban status WORK-XXXX`:

1. Find the ticket file at `obsidian/kanban/tasks/WORK-XXXX.md`.
2. Find which section of `obsidian/kanban/work-kanban-board.md` contains the ticket entry.
3. Report ticket title, `Status:`, board section, source/spec references, and whether there are open questions or blocking notes.
4. If `Status:` and board section disagree, report the mismatch and offer to fix it.

## Talk

For `/kanban talk ...`, accept any of these inputs:

```txt
/kanban talk WORK-XXXX
/kanban talk WORK-XXXX - WORK-YYYY
/kanban talk WORK-XXXX, WORK-YYYY, WORK-ZZZZ
/kanban talk WORK-XXXX --lang=en
```

Language options:

- Default output language is Thai for the generated marketing ticket content.
- Use `--lang=en`, `--lang=english`, `--language=en`, or `--language=english` to generate the marketing ticket content in English.
- Keep fixed ticket IDs, file paths, `Status:`, section labels, route names, commands, and technical identifiers unchanged regardless of output language.

Parse ticket targets as follows:

- A single ticket target processes that one ticket.
- A range target such as `WORK-1021 - WORK-1038` processes every ticket inclusively, in numeric order from the first ID to the second ID.
- A comma-separated target such as `WORK-1021, WORK-1028, WORK-1038` processes tickets in the exact queue order provided.
- Combined comma and range syntax is allowed, for example `WORK-1021 - WORK-1023, WORK-1038`; expand each range in place and preserve the overall queue order.
- If the same ticket appears more than once after expansion, process it only once at its first position and report that duplicates were skipped.

For each ticket in the resolved queue:

1. Use the `management-talk` skill.
2. Read `obsidian/kanban/tasks/WORK-XXXX.md`.
3. Follow the `management-talk` required context rules, including reading `prd.md` when the ticket affects public pages, website content, SEO, AI readability, admin-managed content, product scope, or brand behavior.
4. Create or update `obsidian/kanban/tasks/marketing-tickets/WORK-XXXX.md` using the `management-talk` output format.
5. Do not move the ticket between board sections.
6. Do not change `Status:` in the source ticket.
7. Continue to the next ticket even if one ticket has open questions or cannot be translated, unless the failure prevents safely interpreting the remaining queue.

After processing the queue, report:

- the resolved ticket order
- created or updated marketing ticket paths
- skipped duplicates, missing source tickets, and open questions that prevented a complete translation

## API Get Backlog

For `/kanban api get-backlog`:

1. Read config from `.kanban.json` or environment variables. Require both `workspaceAiUrl` (from `workspaceAiUrl` key or `WORKSPACE_AI_URL` env) and `WORKSPACE_AI_TOKEN`. If either is missing, stop with:

   ```
   KANBAN_API error: workspaceAiUrl หรือ WORKSPACE_AI_TOKEN ไม่ได้ตั้งค่า
   ตั้งค่าใน .kanban.json หรือ environment variables แล้วลองใหม่
   ```

2. Scan all files in `obsidian/kanban/specs/backlog/` (if the directory exists). For each file, read its content and parse lines matching `Card ID: {id}` — collect all IDs into a `skip_ids` array. If the directory does not exist, `skip_ids` is empty.

3. Call `POST {workspaceAiUrl}/api/kanban/backlog/export` with:
   - Header: `X-API-Key: {WORKSPACE_AI_TOKEN}`
   - Header: `Content-Type: application/json`
   - Body: `{ "skip_ids": [...] }`

4. Handle API errors:
   - 401 / 403: stop with "X-API-Key ไม่ถูกต้อง"
   - 404: stop with "Backlog column not found"
   - Network / timeout: stop with the error message
   - Any other non-2xx: stop with the status code and message

5. If `obsidian/kanban/specs/backlog/` does not exist, create it before writing files. If creation fails (permission error), stop with the OS error.

6. For each item in the API response array:
   - If `action` is `"SKIP"`: count as skipped, do not write.
   - If `action` is `"NEW"`: write the `content` value to `obsidian/kanban/specs/backlog/{filename}` where `filename` comes directly from the API response field `filename`. Do not derive or reconstruct the filename.

7. Report results regardless of whether any new files were written:
   - When new files exist: `kanban api get-backlog: exported N new, skipped M\nFiles written to obsidian/kanban/specs/backlog/`
   - When no new files: `kanban api get-backlog: ไม่มี card ใหม่ — skipped M card ที่มีอยู่แล้ว`

**Content format** written by the API (do not reformat):
```md
# {title}

Card ID: {id}
Created At: {created_at}

## รายละเอียด {type}
{description}
```

**Rules:**
- Never use Artisan commands — this command runs in external projects without Laravel.
- Use `filename` from the API response directly. Do not generate filenames.
- Write only cards with `action: "NEW"`.
- The `skip_ids` deduplication prevents rewriting existing cards.
- `obsidian/kanban/specs/backlog/` is auto-created if absent.
