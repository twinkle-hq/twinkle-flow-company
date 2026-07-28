# project-management

`tier: company` · per-workspace **library** (attach to a workspace; not
force-attached).

The **executive** agent's core plugin. Where the other company-tier plugins are
thin MCP connectors, this one is the substance: it turns a worker into a
PM-agent that drives the whole project lifecycle in Linear — decomposing
approved projects into worker tasks, monitoring health on a schedule, reviewing
worker results, and escalating to a human when it hits an authority limit or a
confidence floor. The agent **never writes code**; everything is delegated to
worker agents through Linear issues.

```yaml
# plugin.yaml
name: project-management
description: "Executive PM-agent — coordinates worker agents through Linear, manages project lifecycle, monitors health, reviews results, escalates blockers"

session:
  mcp:
    - linear-mcp
  skills:
    - skills/executive-overview/SKILL.md
    - skills/twinkle-executive-framework/SKILL.md
  capabilities:
    - create-worker-task
    - defer-and-delegate

flows:
  - on-project-approved
  - on-project-feedback
  - on-task-review
  - on-task-blocked
  - on-escalation-ready
  - review-gate-flow
  - project-completion-flow
  - projects-health-check-flow
  - initiatives-health-check-flow
  - project-progress-review-flow
  - initiative-kickoff
  - task-decomposition
```

## Tier & attach model

This plugin lives in the **company tier**, a **per-workspace library** — it is
not injected into every session like the system/core force-attach tiers. To put
an executive on a workspace you attach the company library to the workspace
(`workspace_libraries`) and assign the bundled
[`executive`](../../roles/executive.yaml) role to the worker. That role bundles
`project-management` together with `workers-orchestra`, `workers-collaboration`,
`core-memory`, and `core-summary`. The executive then has this plugin's Linear
coordination logic plus persistent memory and a structured execution summary.

## Session config

The plugin contributes no hooks or gates of its own — it shapes the session
entirely through MCP, skills, and capabilities:

| Mechanism | Value | Effect |
|---|---|---|
| `session.mcp` | `linear-mcp` | Scoped Linear access via the internal `linear-mcp-proxy` ([`mcp/linear-mcp.yaml`](../../mcp/linear-mcp.yaml), `connection_type: internal`). The runner registers a per-task session (`task_id`, `task_identifier`, `team_id`, `scope_labels`) and writes the scoped token + `mcp_url` into the worker's `.mcp.json`; the Linear API key never reaches the agent. |
| `session.skills` | `executive-overview`, `twinkle-executive-framework` | Two SKILL.md files compiled into every executive session. They define the operating doctrine — see below. |
| `session.capabilities` | `create-worker-task`, `defer-and-delegate` | Two task definitions compiled into SKILL.md as always-available process guides. Every flow that needs to create a worker issue or hand off unfinished work routes through these, never through a raw `save_issue`. |

### The two skills (operating doctrine)

- **`executive-overview`** — the architecture overview. Establishes the four
  core principles (never write code; ephemeral one-task-per-session; `result.md`
  controls status — never self-change status; everything through tasks), the
  `result.md` frontmatter contract (`status: done | needs_human | failed`), the
  Linear MCP tool table, the self-verification rule (every write call followed
  by a read), the label scheme (worker label, `Model: Opus/Sonnet`,
  `Timeout: …`, `review_required`), the 1–10 **complexity score** that drives
  model/review/split decisions, the **confidence thresholds**, the project /
  issue / initiative status machines, the executive's **authority limits**, the
  anti-duplication rule, and the per-process iteration caps.
- **`twinkle-executive-framework`** — the condensed authority charter:
  Autonomous vs Requires-human vs Forbidden action lists, the `Team.md` contract
  (read worker capabilities before assigning; verify against
  `list_issue_labels`), confidence scoring, and self-verification.

## The project lifecycle it drives

```
initiative (Active)
  └─ initiative-kickoff → review-gate (per project)
       └─ project: Pending Review
            ├─ human Approved   → task-decomposition → project: In Progress
            └─ Changes Requested → handle-project-feedback → Pending Review (max 3 rounds)

project In Progress
  ├─ worker tasks: Ready → In Progress → Done | In Review | Blocked | Skipped
  │     ├─ In Review (+review_required) → review-worker-task → Done | Ready (≤2 returns) | Blocked
  │     └─ Blocked                       → unblock-worker     → Ready | Skipped | escalate
  └─ all tasks Done/Skipped (Completing) → project-completion → Completed (confidence ≥ 85)

continuous overlay
  ├─ projects-health-check (every 4h)      — completion detection, cancelled cleanup, Team.md validation
  ├─ project-progress-review (daily 12:00) — deep KPI-coverage check, fill gaps with new tasks
  └─ initiatives-health-check (daily 09:00) — status updates, escalate completed/empty initiatives
```

## Flows, grouped by kind

All twelve flows dispatch a single `type: agent` job onto the executive worker.
Each carries `workflow_dispatch: true` — they can also be triggered manually, or
via team-mcp.

### Linear-event reactions

These fire off a Linear webhook event filtered to a specific status/label
transition, and invoke the matching reaction task:

| Flow | Trigger | Task |
|---|---|---|
| [`on-project-approved`](flows/on-project-approved.yaml) | `project.update` → status `Approved` | `task-decomposition` |
| [`on-project-feedback`](flows/on-project-feedback.yaml) | `project.update` → status `Changes Requested` | `handle-project-feedback` |
| [`on-task-review`](flows/on-task-review.yaml) | `issue.update` → status `In Review` + label `review_required` | `review-worker-task` |
| [`on-task-blocked`](flows/on-task-blocked.yaml) | `issue.update` → status `Blocked` | `unblock-worker` |
| [`on-escalation-ready`](flows/on-escalation-ready.yaml) | `issue.update` → status `Ready` + label `escalation` | `handle-escalation` |
| [`review-gate-flow`](flows/review-gate-flow.yaml) | `issue.create` → status `Ready` + labels `executive`, `create-project` | `review-gate` |
| [`project-completion-flow`](flows/project-completion-flow.yaml) | `project.update` → status `Completing` | `project-completion` |
| [`initiative-kickoff`](flows/initiative-kickoff.yaml) | `initiative.create` / `initiative.update` → status `Active` | `initiative-kickoff` |

`project-completion-flow` is normally tripped not directly by a human but by the
`projects-health-check` task, which moves a project whose tasks are all
terminal into the `Completing` status that this flow listens for.

### Scheduled health-checks

Cron-triggered scans that keep the board honest between event reactions:

| Flow | Schedule | Task |
|---|---|---|
| [`projects-health-check-flow`](flows/projects-health-check-flow.yaml) | `0 */4 * * *` (every 4h) | `projects-health-check` |
| [`project-progress-review-flow`](flows/project-progress-review-flow.yaml) | `0 12 * * *` (daily 12:00) | `project-progress-review` |
| [`initiatives-health-check-flow`](flows/initiatives-health-check-flow.yaml) | `0 9 * * *` (daily 09:00) | `initiatives-health-check` |

### Manual / dispatch-only

| Flow | Trigger | Task |
|---|---|---|
| [`task-decomposition`](flows/task-decomposition.yaml) | `workflow_dispatch` only | `task-decomposition` |

`task-decomposition.yaml` is intentionally dispatch-only. The canonical
event-driven decomposition path is `on-project-approved`, which invokes the same
`task-decomposition` task on the `project.update`/`Approved` event. Adding an
`on.linear` block here would double-fire decomposition on a single Approved
event and create duplicate worker tasks; the manual flow exists for re-runs and
team-mcp dispatch.

## Task catalog

Sixteen task definitions live under [`tasks/`](tasks/). Two of them
(`create-worker-task`, `defer-and-delegate`) are also wired as session
**capabilities** — always-available process guides every other task calls into.
Grouped by purpose:

**Decomposition & project creation**
- [`task-decomposition`](tasks/task-decomposition.yaml) (`complexity: critical`) —
  break an approved project's framework into concrete worker tasks: complexity
  scoring, worker assignment from `Team.md`, `blocked_by` dependencies, then move
  the project to In Progress. Splits complexity ≥ 7 into flat subtasks; offloads
  scope > 10 via `defer-and-delegate`; creates every issue through
  `create-worker-task`.
- [`review-gate`](tasks/review-gate.yaml) (`complexity: complex`) — formulate a
  project's goal, requirements, measurable KPIs, and high-level task framework,
  pass it through an independent review loop, and create it in `Pending Review`.
- [`initiative-kickoff`](tasks/initiative-kickoff.yaml) — analyze a freshly
  activated initiative and create the projects it needs (each via `review-gate`).
- [`handle-project-feedback`](tasks/handle-project-feedback.yaml) — address human
  feedback on a project in `Changes Requested`, reply per comment, revise, and
  resubmit to `Pending Review`.

**Worker-task lifecycle (creation, review, unblock)**
- [`create-worker-task`](tasks/create-worker-task.yaml) **(capability)** — the
  mandatory path for creating ANY worker issue: journal research → self-sufficient
  draft → subagent review loop → `save_issue` + verify. Invoked by
  task-decomposition, unblock-worker, project-progress-review, and
  defer-and-delegate.
- [`review-worker-task`](tasks/review-worker-task.yaml) — review a completed
  `review_required` task: accept (→ Done), return with feedback (→ Ready, ≤ 2
  returns), or Blocked + escalate after the return cap.
- [`unblock-worker`](tasks/unblock-worker.yaml) (`complexity: complex`) — diagnose
  why a task is Blocked and fix it (rewrite description, change model, split) or
  escalate; capped at 2 attempts before Skipped + escalate.
- [`review-draft`](tasks/review-draft.yaml) / [`review-loop`](tasks/review-loop.yaml) —
  the independent-reviewer subagent machinery; used as a `type: task` phase inside
  review-gate, task-decomposition, unblock-worker, and create-worker-task.

**Health checks & progress review (scheduled)**
- [`projects-health-check`](tasks/projects-health-check.yaml) — lightweight status
  scanner: detect completable projects (dispatch `project-completion`), clean up
  Cancelled projects (Skip open tasks), validate `Team.md` against Linear labels.
- [`project-progress-review`](tasks/project-progress-review.yaml) — deep per-project
  KPI-coverage check on In Progress projects; fills gaps with new tasks via
  `create-worker-task`.
- [`initiatives-health-check`](tasks/initiatives-health-check.yaml) — scan Active
  initiatives, write status updates, escalate Completed or empty initiatives
  (the executive cannot close an initiative itself).
- [`project-completion`](tasks/project-completion.yaml) — verify every requirement
  and KPI is genuinely met, then close the project (confidence ≥ 85) or escalate.

**Escalation & human-in-the-loop**
- [`defer-and-delegate`](tasks/defer-and-delegate.yaml) **(capability)** — the
  mandatory path when work can't finish in the current session: create a helper
  task (worker or `[Escalation]` issue) plus a `blocked_by` continuation task,
  then close the current one with a delegation record. Invoked by
  task-decomposition, create-worker-task, project-completion, unblock-worker,
  review-worker-task, handle-project-feedback, initiatives-health-check, and
  projects-health-check.
- [`handle-escalation`](tasks/handle-escalation.yaml) — process a human's response
  when an `escalation` issue is moved back to `Ready`: resolve and journal the
  decision, or ask clarifying questions.
- [`gather-input`](tasks/gather-input.yaml) — the `needs_human` ask/answer
  protocol for collecting human answers mid-execution.

## Human gates

The executive is deliberately bounded. It runs autonomously inside its
authority envelope and stops at hard limits — it **cannot** cancel projects,
create/activate/complete/delete initiatives, or change its own task status (the
last is also blocked by the MCP proxy). Beyond those structural limits it
escalates whenever a decision falls below a confidence floor or hits an
iteration cap: project completion needs confidence ≥ 85; accepting a worker
result or unblocking needs ≥ 75; decomposition needs ≥ 50; project feedback caps
at 3 rounds, task review at 2 returns, unblock at 2 attempts, and the subagent
reviewer at 3 iterations. Every escalation goes through `defer-and-delegate`
(helper `[Escalation]` issue + `blocked_by` continuation), and the human's reply
re-enters via the `on-escalation-ready` → `handle-escalation` path.

> The full set of authority limits, confidence thresholds, and per-process
> iteration caps is the source of truth in the
> [`executive-overview`](skills/executive-overview/SKILL.md) and
> [`twinkle-executive-framework`](skills/twinkle-executive-framework/SKILL.md)
> skills, compiled into every executive session.
