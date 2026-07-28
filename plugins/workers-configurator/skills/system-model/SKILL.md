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
# internal — Orchestra core services. RESERVED + RUNTIME-SEEDED.
# The four builtins (linear-mcp, journal-mcp, team-mcp,
# telegram-inbox) are seeded directly into mcp-registry (the runtime source of
# truth) — do not redeclare them in a library repo. Any mcp/*.yaml mirror in a
# library repo is runtime-inert; keep it only as a cross-reference anchor for
# session.mcp lint checks.
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
- **Library repos** cannot use `connection_type: internal` and cannot shadow any of the five reserved builtin names. Use `sidecar-http` / `sidecar-stdio` for a repo-local container.
- `remote-pat` and `remote-oauth` URLs must be `https://`.
- `tool_filter` only works on sidecar types (enforced by `mcp-runtime /proxy`; remote paths have no proxy layer).
- `internal.registration_payload_fields` entries must be in the context-bag vocabulary: `task_id`, `task_identifier`, `team_id`, `scope_labels`, `visible_workers`, `role`. `workspace` and `worker` are always sent implicitly — do not list them.
- For sidecar `env_schema`, mark every credential field with `secret: true`.

MCP entity declarations live in a library repo's `mcp/` directory (flat, at the repo root). The five reserved internals are seeded into mcp-registry at the platform level; a library repo only declares its own `sidecar-http` / `sidecar-stdio` / `remote-pat` / `remote-oauth` providers. Create MCP entities before plugins that reference them.

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

**SessionStart hook as capability introduction:** Plugins that provide meaningful capabilities should include a SessionStart hook that briefly orients the agent — what abilities this plugin gives and which skill to invoke first for context. Keep it concise: 2-3 sentences, no lists of individual capabilities. Point to the primary knowledge skill (e.g., `/system-model`).

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
  workflow_dispatch: true    # enables manual/team-mcp dispatch (valid alone for dispatch-only flows)
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

**Trigger types:** `schedule` (cron), `linear` (webhook events with filters), `webhook` (external), `workflow_dispatch` (manual / team-mcp dispatch).

**Triggers:** Give an event-driven flow at least one automatic trigger (schedule, linear, or webhook). `workflow_dispatch` alone is also fully valid — `flows lint` accepts it — and is the correct choice for a dispatch-initiated process. Meta-configurator flows like `design-agent`, and the executive's `task-decomposition`, declare `workflow_dispatch` only. Use an automatic trigger when the process should react to events; use `workflow_dispatch`-only when it is always explicitly dispatched.

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

## Packaging & Distribution (repos + manifests + MVS)

Entities are distributed as **git repos with a `flows.toml` manifest** — there is
no `data/flows/` shared repo and no two-tier store. Each repo is a self-contained
**library**: a `flows.toml` at the root declares what it is, what it exposes, and
what it depends on.

```toml
# flows.toml — repo manifest
[package]
name = "twinkle-hq/twinkle-flow-company"   # canonical id (owner/repo)
version = "v0.1.0"                          # semver-ish git tag

[exports]
plugins = "*"        # which plugins this repo exposes ("*" = all, or a list)
roles = "*"          # which roles this repo exposes

[dependencies]
"twinkle-hq/twinkle-flow-core" = "v0.1.0"    # other repos this one needs
```

**MVS resolution.** A workspace declares an ordered list of attached libraries.
flows-core's **MVS (Minimal Version Selection)** resolver walks each library's
`[dependencies]`, clones + pins every transitive repo at the minimum version that
satisfies all constraints, and merges the exported entities into a single
**workspace build list**. Cross-repo references (e.g. a company role listing the
core plugin `core-memory`) resolve at runtime against this merged view.
`flows lint` is strictly repo-local. It emits a benign "Referenced plugin not
found" warning for a cross-repo ref that MVS resolves fine — read that warning
as benign, and suppress it in `.flows-lint.yaml` rather than editing the ref.

### How a repo reaches a worker — two attach mechanisms

1. **Per-workspace library** (opt-in, per workspace). The admin attaches a repo
   to a specific workspace (`POST /api/config/workspaces/<ws>/libraries`).
   flows-service clones it, MVS-resolves the build list, and serves the merged
   view to that workspace's workers. **This repo (`twinkle-flow-company`) is a
   library** — attach it to a workspace; it is NOT force-attached.

2. **Force-attach** (admin-global — injected into every worker session of every
   workspace, independent of attached libraries). Two tiers:
   - **system tier** (`twinkle-flow-system`: `system-result`,
     `system-autocommit`, `system-kick-start`) — **locked**: non-removable, every
     plugin always on. The platform cannot run without these.
   - **core tier** (`twinkle-flow-core`: `core-state`, `core-memory`,
     `core-summary`) — default-on but **toggleable** per plugin.

   The official force-attach catalog is seeded from
   `config-service/preset-repos.js` (`SHIPPED_CATALOG`).

A worker's effective entity set = force-attached system + core plugins
(global) + the plugins/roles MVS-resolved from its workspace's attached
libraries. The runner reads the worker's flat `plugins` list (the `workers.role`
column was dropped — applying a role just copies the role's plugins into the
worker's `plugins` at creation).

## Directory Structure

Every library repo (this one included) has this structure at its root:

```
<repo>/
  flows.toml              Manifest: [package] / [exports] / [dependencies] (required)
  mcp/                    MCP entity YAMLs (flat)
  plugins/                Plugin directories
    {name}/
      plugin.yaml         Plugin definition (required)
      skills/             SKILL.md files (optional)
      flows/              Flow YAMLs (optional)
      tasks/              Task YAMLs (optional)
  roles/                  Role YAMLs (flat)
  .flows-lint.yaml        Optional: lint suppress config (e.g. silence benign
                          cross-repo role warnings)
```

All entities belong to plugins — tasks, flows, and skills are always owned by a
plugin. There is no `shared/` directory and no `data/flows/...` prefix. Paths are
relative to the repo root — `plugins/my-plugin/plugin.yaml`. You author and edit
entities in your repo (cloned as your git_repo); the runner-side git guards
commit and push your changes. Force-attached system/core repos are resolved
read-only by the platform — you don't edit them.

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

**MCP entity names** match their service name: `journal-mcp`, `linear-mcp`, `telegram-inbox`.

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

**Preview** — `flows preview-skill <task>`: render the SKILL.md the runner will compile from a task. Read it to see what the agent will actually see.

**twinkle** (CLI — worker/task and platform-config operations). Whether the
command exists on this machine, and whether it is POINTED at a deployment, are
two separate things and neither is a given — see "Four shapes an answer can
take" below before you diagnose any error from it. Never run
`twinkle auth login`.

- `twinkle worker start <ws> <w>`, `twinkle worker stop <ws> <w>` — `scope: 'worker'`
- `twinkle task create`, `twinkle task get <id>`, `twinkle run list` — `scope: 'worker'`
- `twinkle plugin list --workspace <ws>`, `twinkle role list --workspace <ws>`,
  `twinkle worker capabilities <ws> <w>`, `twinkle library sync <ws> <namespace>`
  — `scope: 'worker'`
- `twinkle worker create <ws> <w>`, `twinkle worker update <ws> <w>`,
  `twinkle worker delete <ws> <w>`, `twinkle worker status <ws> <w>`,
  `twinkle worker list [--workspace <ws>]`,
  `twinkle tracker worker-config <ws> <w>`, `twinkle library list <ws>`
  — **`scope: 'admin'`**

The four `scope: 'worker'` reads/writes on the third line are how you check the
platform's view against the repo's, and they are the ones the configurator's job
actually needs:

- **`twinkle plugin list --workspace <ws>` / `twinkle role list --workspace <ws>`**
  — the plugins and roles the platform answers for that workspace, which is not
  the same set as the files in your checkout. **The `--workspace` flag is not
  optional.** Measured on a live stand: `plugin list` with no flag answered 6
  plugins (`core-memory`, `core-state`, `core-summary`, `system-autocommit`,
  `system-kick-start`, `system-result`), and with `--workspace sandbox` answered
  11; `role list` with no flag answered `[]`, and with `--workspace sandbox`
  answered 2. Read an empty or short answer that came back without the flag as
  "the flag was omitted", and re-run the command with `--workspace` before you
  report what the platform holds.
- **`twinkle worker capabilities <ws> <w>`** — read this back after
  `twinkle worker update --plugins …` and a `library sync`, to see what the
  platform answers for that worker. It answers `{worker, description, active}`.
  Measured on a live stand, over three workers that all read `"state":"stopped"`
  in `worker list`: `sandbox/executive` answered
  `{"worker":"executive","description":"","active":[]}`, `sandbox/configurator`
  answered `"active":[]`, and `qa/qa-worker` answered two entries — a flow
  `qa-review-gate` and a task `qa-parallel-review`. **Do not start or stop a
  worker in order to change this answer.** When `active` comes back empty after
  an update, re-check the plugin names you passed and re-run `library sync`.
- **`twinkle library sync <ws> <namespace>`** — run it after you commit and push
  entity edits. The `<namespace>` comes from `twinkle library list <ws>` —
  admin-scoped, listed above with the other six.

**`twinkle worker update <ws> <w>`** is the admin-scoped write for changing an
existing worker — `--plugins` (repeatable), `--model`, `--machine`,
`--visible_workers`, `--description` and the rest.

**Before `twinkle worker create <ws> <w>`, confirm the name is not already in
`twinkle worker list --workspace <ws>`.** If it is there, do not create over it:
pick a name that is free, or change the existing worker with
`twinkle worker update` instead. Creating over an existing name risks
reconfiguring a worker that is not yours, and no error the command returns is
worth trusting to tell you that is what happened.

Three things to know before using the admin seven:

1. **They can be refused, and the refusal is not the only failure shape.** See
   "Four shapes an answer can take" immediately below — a `403`, a connection
   error and a missing binary are three different facts, and only one of them
   is about scope.
2. **Always pass `--git_repo` to `twinkle worker create`, naming a repository
   that already exists.** With no `--git_repo` the command can create a real
   private GitHub repository, and no twinkle command removes it. That rule is
   the protection — do not go looking for a `--help` line that softens it. When
   you also pass `--git_branch`, the branch must ALREADY EXIST on that remote:
   confirm it with `git ls-remote --heads <repo URL> <branch>`, which prints
   one line for a branch that is there and nothing at all (still exit 0) for
   one that is not.
3. **`twinkle worker delete` reaches outside the platform's own config.** Run
   `twinkle worker delete --help` and read it before you use the command: it
   prints that the call removes the worker's configuration in config-service
   and in the tracker, and deletes **the Linear label the worker filed tasks
   under — the label disappears from every issue that carried it**; the worker
   directory and its git repo are left alone, and workspace-keyed history and
   journal entries survive. Delete a worker you created. Do not delete one you
   merely found. On a worker you merely found, the label a delete removes is on
   issues that are not yours.

### Four shapes an answer can take

Place every answer in one of these four before you diagnose it.

1. **It answers.** Everything above is available to you in this session.
2. **`403` with a JSON body.** That is this session's scope: not a broken
   command and not a transient failure. Report it, do not retry, do not go
   looking for another flag.
3. **A connection error, or any message naming `http://localhost`.** The CLI is
   not pointed at a deployment here. A read warns on stderr first — measured
   verbatim: `twinkle: no --url, --profile, TWINKLE_URL or stored profile —
   reading from the default vhost http://localhost`. **Never run
   `twinkle auth login` and never invent a `--url`**, whatever the CLI's own
   error text suggests: there is no credential to log in with and no address
   you can supply that reaches the platform. Report "the twinkle CLI is
   unconfigured on this machine" and use the non-CLI path. Retrying this shape
   as though the platform were broken is the misdiagnosis this section exists
   to prevent.
4. **A `command not found` error naming `twinkle`.** Measured in a shell with
   no such binary: `bash: twinkle: command not found`, exit status 127. There
   is no `twinkle` command here. **Do not install one, do not build one, do not
   retry, and do not run `twinkle auth login`.** Report "there is no twinkle
   command on this machine", author and validate entities with the repo-local
   `flows` CLI as usual, and record the platform-side reads and writes you
   could not make.

`twinkle <group> <action> --help` prints the command's signature, its flag list
and its `scope:` line. Run it; the lists above are a summary of it.

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
