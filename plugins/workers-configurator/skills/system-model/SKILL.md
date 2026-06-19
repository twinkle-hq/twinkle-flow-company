---
name: system-model
description: Orchestra system architecture — entity types, binding chain, runner assembly, MCP tools. Essential knowledge for creating and managing agent configurations.
---

# Orchestra System Model

You configure agents in the Orchestra system. This document defines every entity type, how they relate, and how the runner assembles them into working agent sessions.

## Entity Types

### MCP Entity
Standalone, reusable MCP server declaration. Lives in `mcp/*.yaml` (flat, no subdirectories). Plugins reference MCP entities by name; one MCP entity can be reused by many plugins.

Every declaration picks **one of five** `connection_type` values. The type determines which fields are required and how the runner wires the MCP at session start.

```yaml
# remote-pat — remote HTTPS MCP with a static API key / PAT in a header
name: sentry
description: Sentry MCP
connection_type: remote-pat
url: https://mcp.sentry.dev
header_name: Authorization
header_template: "Bearer {{token}}"
token_env_var: SENTRY_TOKEN            # key inside the integration's env_values
```

```yaml
# remote-oauth — remote HTTPS MCP where OAuth is delegated to the Nango broker
name: atlassian
description: Atlassian MCP
connection_type: remote-oauth
url: https://mcp.atlassian.com
broker_provider: atlassian-oauth       # Nango integration id
header_name: Authorization
header_template: "Bearer {{access_token}}"
resource_indicator: https://mcp.atlassian.com   # optional, RFC 8707
```

```yaml
# sidecar-http — local Docker image that speaks MCP over HTTP natively
name: granola
description: Granola notes
connection_type: sidecar-http
image: granola/mcp:latest
port: 3000
env_schema:
  - { name: GRANOLA_API_KEY, required: true, secret: true }
  - { name: GRANOLA_WORKSPACE, required: false, secret: false }
healthcheck: { path: /healthz, port: 3000 }
per_worker: false                      # default; true only for stateful MCPs
tool_filter: [search_notes, list_projects]    # optional allowlist (sidecar-only)
```

```yaml
# sidecar-stdio — local stdio image, auto-wrapped by Supergateway to expose HTTP
name: statsig
description: Statsig stdio MCP
connection_type: sidecar-stdio
image: statsig/mcp:latest
command: [node, server.js]
env_schema:
  - { name: STATSIG_KEY, required: true, secret: true }
per_worker: true                       # default true for stdio
```

```yaml
# internal — Orchestra core services. RESERVED: these live in the SHARED repo only.
# The five builtins (linear-mcp, journal-mcp, team-mcp, orchestra-api,
# telegram-inbox) are already seeded — do not redeclare them in a workspace repo.
name: linear-mcp
description: Scoped Linear access
connection_type: internal
url: http://linear-mcp-proxy:3001
registration_endpoint: http://linear-mcp-proxy:3002/internal/register
registration_payload_fields: [task_id, task_identifier, team_id, scope_labels]
registration_response: { token_field: token, url_field: mcp_url }
auth: { type: internal_secret }
```

**Rules `flows lint` enforces for MCP entities:**
- `connection_type` is required and must be one of the five above.
- **Workspace repos** cannot use `connection_type: internal` and cannot shadow any of the five reserved builtin names. Use `sidecar-http` / `sidecar-stdio` for a workspace-local container.
- `remote-pat` and `remote-oauth` URLs must be `https://`.
- `tool_filter` only works on sidecar types (enforced by `mcp-runtime /proxy`; remote paths have no proxy layer).
- `internal.registration_payload_fields` entries must be in the context-bag vocabulary: `task_id`, `task_identifier`, `team_id`, `scope_labels`, `visible_workers`, `role`. `workspace` and `worker` are always sent implicitly — do not list them.
- For sidecar `env_schema`, mark credential fields with `secret: true` so the Settings UI renders them as password inputs and the API redacts them on read.

Declarations live in the shared repo (`mcp/` at repo root — for reserved internals + any org-wide provider) or the workspace repo (`mcp/` at the workspace-repo root). Workspace entries override shared ones by name, except reserved internals. Create MCP entities before plugins that reference them.

### Plugin
The atomic unit of agent capability. Lives in `plugins/{name}/plugin.yaml`.

```
Plugin (atomic capability unit)
├── session
│   ├── mcp []           — MCP tool connections
│   ├── skills []        — static knowledge (SKILL.md files)
│   ├── capabilities []  — process guides (task names → compiled SKILL.md)
│   └── hooks {}         — lifecycle hooks
└── flows []             — active processes (DAGs, event-triggered)
```

```yaml
name: my-plugin
description: "What this plugin provides"
session:
  mcp: [mcp-entity-name]              # tools available to the agent
  skills: [skills/my-skill/SKILL.md]   # knowledge injected into .claude/skills/
  capabilities: [my-task]              # task names compiled into SKILL.md
  hooks:                                # Claude Code lifecycle hooks
    SessionStart:
      - matcher: "startup"
        hooks:
          - type: command
            command: "cat context.md 2>/dev/null || true"
    Stop:
      - hooks:
          - type: command
            command: "bash .claude/hooks/check.sh"
flows: [my-flow]                       # DAGs owned by this plugin
```

**Session fields:**
- `mcp` — array of MCP entity names. Runner resolves to full definitions and registers sessions.
- `skills` — file paths relative to plugin dir. Placed in `.claude/skills/` with plugin-name prefix.
- `capabilities` — array of task names. Runner compiles each task into a SKILL.md and injects at session start. Agent uses them as process guides based on trigger.
- `hooks` — Map of event → array of `{name, command}` entries. `name` is required (human-readable label shown in UI). Events: SessionStart, Stop, PreToolUse, PostToolUse. Stop hooks block via exit code 2 + stderr + optional `blocking: true` + `message`. SessionStart stdout is injected as agent context.

**Process fields:**
- `flows` — flow names. YAML files must exist in `plugins/{name}/flows/`.

**SessionStart hook as capability introduction:** Plugins that provide meaningful capabilities should include a SessionStart hook that briefly orients the agent — what abilities this plugin gives and which skill to invoke first for context. Keep it concise: 2-3 sentences, no lists of individual capabilities. Point to the primary knowledge skill (e.g., `/system-model`) so the agent knows where to start.

**What plugins do NOT contain:** tasks (referenced by flows), context (comes from runner), inline MCP definitions (use entities), inline skill content (use file refs).

## Capabilities

Plugins provide two types of capabilities:

### Active Capabilities (Flows)
- Defined in `plugin.flows[]`
- Triggered by events (schedule, Linear webhooks, manual dispatch)
- Create Linear tasks and orchestrate multi-step processes
- Other workers can dispatch them via team-mcp `dispatch_flow`

### Passive Capabilities
- Defined in `plugin.session.capabilities[]`
- List of task names compiled into SKILL.md at session start
- Available as process guides in every session — agent decides when to use them based on trigger
- Other workers see them via team-mcp `get_worker_capabilities`

### The `trigger` Field
Tasks used as capabilities should have a `trigger` field describing when to activate:
```yaml
trigger: "User asks to create a plugin, add a capability, or configure MCP tools"
```

At compilation, trigger merges into the SKILL.md frontmatter description:
`"{description}. Use when: {trigger}"`

Good triggers are specific and pushy — list concrete user phrases and situations.
Bad: "Plugin-related work"
Good: "User asks to create a plugin, add a new capability to a worker, or bundle MCP tools with skills"

### Role
Optional preset. Applying a role copies its plugins to a worker's config.

```yaml
name: configurator
description: "Agent configuration designer"
plugins:
  - core-memory
  - workers-configurator
  - core-summary
```

Roles group plugins into reusable presets. A worker can have plugins from multiple roles or assigned directly.

### Flow
DAG of jobs triggered by events. Active only when the worker is running.

```yaml
name: my-flow
version: 1
description: "What this flow does"
params:
  worker_name:
    type: worker
on:
  linear:                    # at least one automatic trigger required
    event: issue.create
    filters:
      labels: [my-label]
  workflow_dispatch: true    # optional, for debugging only
jobs:
  step1:
    type: agent
    task: my-task
    worker: $worker_name
    timeout: 30m
  step2:
    type: agent
    task: another-task
    worker: $worker_name
    needs: [step1]
    timeout: 30m
```

**Trigger types:** `schedule` (cron), `linear` (webhook events with filters), `webhook` (external), `workflow_dispatch` (manual/debug only).

**Triggers:** Flows should ideally have at least one automatic trigger (schedule, linear, or webhook). `workflow_dispatch` alone is valid but means the flow can only be dispatched manually or via team-mcp. Exception: meta-configurator flows like design-agent are dispatch-only (workflow_dispatch) since agent design is always initiated by a human or another agent, not by events.

**Data flow between jobs:** Job outputs (from task YAML `outputs`) are uploaded as Linear documents. Next job receives dependency documents injected at session start. Only direct `needs` are injected.

### Task
Defines what an agent does within a single session. The most important entity for agent behavior quality.

```yaml
name: my-task
description: "What this task accomplishes"
complexity: medium           # simple | medium | complex | critical
outputs:                     # REQUIRED, runner validates at session end
  - result_name
  - another_output

hard_gates:                  # optional, non-negotiable rules
  - before: phase-name
    rule: "Condition that must be true"

anti_rationalization:        # required for medium+ complexity
  title: "Common shortcuts"
  entries:
    - thought: "What the agent will think to skip a step"
      reality: "Why that's wrong and what to do instead"

phases:
  first-phase:
    goal: "What this phase achieves"
    actions:
      - "Plain text action"
      - text: "Action with metadata"
        why: "Why this step matters"
        type: verify          # verify | decide | write_result | dispatch_subagents | context_template
      - type: verify
        check: "What to verify"
        on_fail: "What to do if verification fails"
    gate: "Condition to pass before moving on"
    next: second-phase

  second-phase:
    goal: "Next phase goal"
    actions:
      - type: write_result
        status: done
        summary: "What was accomplished"
    gate: "Final verification"
    next: null                # null = terminal phase

terminal_state:
  description: "What the world looks like when this task is done"
  invokes:                   # optional: tasks this one dispatches
    - sub-task-name
  invoked_by:                # optional: tasks/flows that call this one
    - parent-flow-name

hooks:                       # optional, deterministic shell checks
  SessionStart:
    - name: "Load test fixtures"
      command: "cat task/fixtures.md 2>/dev/null || true"
  Stop:
    - name: "Run tests"
      command: "npm test"
      blocking: true         # blocks session end on failure
```

**Key rules:**
- `outputs` is required (min 1 item). Runner validates at session end and uploads as documents.
- `complexity` is required. Anti-rationalization is mandatory for medium and above.
- `phases` is required. Each phase has goal, actions, gate, next.
- `terminal_state` is required. Describes the end state.
- Names: lowercase alphanumeric + hyphens only.
- `hooks` is optional. SessionStart hooks inject context; Stop hooks with `blocking: true` prevent completion on failure.
- Hook commands are deterministic — they run as shell commands, not agent instructions. Visible in the task phase graph.

**Action types:**
- Plain string: simple instruction
- `verify`: check + on_fail fields. Agent must verify something.
- `decide`: high_confidence + low_confidence paths.
- `write_result`: status + summary. Produces result.md.
- `dispatch_subagents`: invoke other agents.
- `context_template`: structured sections for content generation.

**Phase-level references:**
- `skills` — array of skill names. Compiled into HARD-GATE requiring `/skill-name` invocation before the phase starts. Use when a phase needs specific knowledge to execute correctly.
- `tools` — array of tool names the phase uses (native tools like `Bash`, `Read`, `Glob`, or MCP tools). Compiled into a reference list shown before the phase goal. Reminds the agent which tools the phase relies on.

```yaml
phases:
  check-existing:
    skills: [system-model]
    tools: [Read, Glob]
    goal: "Check what exists"
    actions: [...]
```

## Binding Chain

```
MCP Entity  <--  Plugin  -->  Role  -->  Worker
(tools)      references    preset     execution
```

1. MCP entities define available tool servers
2. Plugins reference MCP entities + add skills, hooks, flows
3. Roles group plugins into presets (optional)
4. Workers get plugins assigned (directly or via role)

## Two-Tier Store

- **Shared repo** (`data/flows/`): system-level entities, read-only to workspace agents
- **Workspace repo**: your workspace's entities, fully editable

System plugins (prefixed `system-` or `core-`) live in the shared repo. Custom plugins live in workspace repos.

## Directory Structure

Your workspace repo has this structure at its root:

```
<workspace-repo>/
  mcp/                    MCP entity YAMLs (flat)
  plugins/                Plugin directories
    {name}/
      plugin.yaml         Plugin definition (required)
      skills/             SKILL.md files (optional)
      flows/              Flow YAMLs (optional)
      tasks/              Task YAMLs (optional)
  roles/                  Role YAMLs (flat)
```

All entities belong to plugins. There is no shared/ directory — tasks, flows, and skills are always owned by a plugin.

You work in your workspace repo (cloned as your git_repo). Paths are relative to the repo root — `plugins/my-plugin/plugin.yaml`, not `data/flows/plugins/...`. The shared repo has the same structure but is read-only to you.

## Naming Conventions

All entity names use kebab-case (lowercase + hyphens): `create-plugin`, `review-validation`.

**Task prefixes** group tasks by purpose:
- `create-*` — entity creation processes (create-plugin, create-role, create-flow, create-task, create-mcp)
- `review-*` — review and audit processes (review-plugin, review-validation, review-reliability, ...)
- `find-*` — search and research processes (find-external-skills)
- `adapt-*` — adaptation processes (adapt-external-skill)
- `test-*` — testing processes (test-agent)
- `design-*` — design and planning processes (design-agent)

Use these prefixes consistently. They enable UI grouping and make capabilities scannable.

**Plugin prefixes:**
- `system-*` — always-attached system plugins (system-result, system-autocommit, system-kick-start)
- `core-*` — optional core plugins (core-state, core-memory, core-summary)
- `workers-*` — worker collaboration/orchestration/config plugins (workers-collaboration, workers-orchestra, workers-configurator)
- No prefix — domain-specific plugins (telegram, project-management)

**MCP entity names** match their service name: `journal-mcp`, `linear-mcp`, `orchestra-api`, `telegram-inbox`.

## Runner Assembly Process

When a worker starts and picks up a task, the runner:

1. **Plugin resolution** — reads worker's plugin list from config-service
2. **MCP registration** — collects all `session.mcp` references across plugins, resolves to MCP entity definitions, registers sessions with each tool service
3. **Skill injection** — copies all `session.skills` files into `.claude/skills/` with plugin-name prefix
4. **Hook merge** — merges all `session.hooks` across plugins into `.claude/settings.json`
5. **Flow trigger activation** — registers triggers from all plugin flows, scoped to this worker instance
6. **Session start** — injects task context (flow contract or Linear ticket) via SessionStart hook
7. **Session end** — Stop hook validates result.md exists and outputs are present (for flow tasks)
8. **Output upload** — extracts outputs from result.md frontmatter, uploads as Linear documents

## Inspecting & Validating Entities

You author entities by editing YAML files in the repo and validating them with the repo-local `flows` CLI (run via `Bash` from the repo root) plus your native filesystem tools. There is no entity-management MCP server — reading and validation are local operations.

**Reading / listing entities** — inspect the repo filesystem directly with `Read`/`Glob`/`Grep`:
- Plugins: `plugins/<name>/plugin.yaml`
- Tasks: `plugins/<name>/tasks/*.yaml`
- Flows: `plugins/<name>/flows/*.yaml`
- Roles: `roles/*.yaml`; MCP entities: `mcp/*.yaml`

**Validate** — `flows lint` (Bash, from the repo root):
- Whole-repo validation — schema + cross-references + reliability for every entity at once.
- Exit 0 = clean; exit 1 = findings, each tagged `filepath:line`. Focus on findings for the file you just edited.

**Reliability** — `flows reliability <task>`: reliability score + recommendations for a task (most useful for complexity >= medium).

**Preview** — `flows preview-skill <task>`: render the SKILL.md the runner will compile from a task, so you can read what the agent will actually see.

**orchestra-api** (MCP — worker/task operations):
- `create_worker`, `start_worker`, `stop_worker`, `get_worker_status` — worker lifecycle
- `create_task`, `get_task_result`, `get_task_logs` — task operations

## External Skills Marketplace

The skills.sh marketplace (https://skills.sh) provides open-source agent skills that can be adapted for Orchestra:

- **Search:** `npx skills find [query]` — finds skills by keyword
- **NEVER install:** `npx skills add` is blocked — skills must be adapted to Orchestra format
- **Adaptation:** External skills are either converted to task YAMLs (if algorithmic) or kept as SKILL.md files (if knowledge-based)
- **Quality signals:** Prefer skills with 1K+ installs from reputable sources (vercel-labs, anthropics, google-labs)

## Workflow Order

1. Inspect existing entities (`Glob`/`Read` over `plugins/*/{tasks,flows}`, `roles/`, `mcp/`) before creating anything
2. Create MCP entities before plugins that reference them
3. Create tasks before flows that reference them
4. Create plugins after their dependencies exist
5. Validate after creation with `flows lint`
6. Run `flows reliability <task>` on medium+ complexity tasks
7. Test with real tasks before declaring done
