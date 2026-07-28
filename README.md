# twinkle-flow-company

The **company / project-running** tier of the Orchestra plugin catalog. Where
[`twinkle-flow-system`](https://github.com/twinkle-hq/twinkle-flow-system) is the
locked life-support layer and
[`twinkle-flow-core`](https://github.com/twinkle-hq/twinkle-flow-core) makes a
single worker session *more effective*, this tier turns the platform into a
**project-running tool**: it gives you executive PM agents that coordinate worker
agents through Linear, a configurator agent that designs new agent setups, plus
the connective tissue (Telegram inbox, team discovery, worker lifecycle) that
lets agents talk to each other and to the orchestrator.

```toml
# flows.toml
[package]
name = "twinkle-hq/twinkle-flow-company"
version = "v0.1.0"

[exports]
plugins = "*"
roles   = "*"

[dependencies]
"twinkle-hq/twinkle-flow-core" = "v0.1.0"
```

## Tier & attach model

This repo is **not** force-attached. Unlike the system and core tiers (which are
injected globally into every worker session), the company tier is a
**per-workspace library**: a workspace admin opts in by attaching this repo to a
specific workspace. It is the **first entry in the attach-library preset
dropdown** in the UI.

The separation is the point:

- **system** — the platform can't run without it (global, locked).
- **core** — optional baseline effectiveness (global, toggleable).
- **company** — turns the platform into a project-running tool (opt-in per
  workspace, via this library).

## Plugins

Five plugins, exported wholesale (`[exports] plugins = "*"`):

| Plugin | Job | Wires |
|---|---|---|
| [`project-management`](plugins/project-management/README.md) | Executive PM agent — coordinates worker agents through Linear, manages project lifecycle, monitors health, reviews results, escalates blockers | `session.mcp: [linear-mcp]`; the `executive-overview` + `twinkle-executive-framework` skills; the `create-worker-task` + `defer-and-delegate` capabilities; 12 flows |
| [`workers-configurator`](plugins/workers-configurator/README.md) | Create, validate, review, and test Orchestra agent configurations (plugins, roles, flows, tasks, MCP entities) | the `system-model` skill; 14 design/test capabilities; a SessionStart intro hook + a blocking PreToolUse `skills add` guard; the `design-agent` + `flow-creation` flows |
| [`telegram`](plugins/telegram/README.md) | Telegram inbox for receiving messages via MCP | `session.mcp: [telegram-inbox]` |
| [`workers-collaboration`](plugins/workers-collaboration/README.md) | Linear task management + team discovery and flow dispatch | `session.mcp: [linear-mcp, team-mcp]` |
| [`workers-orchestra`](plugins/workers-orchestra/README.md) | Workers lifecycle — start and stop workers, dispatch flows, list system state | the `worker-lifecycle` skill (which teaches the `twinkle` CLI's worker and flow commands) |

No plugin in that table wires the `twinkle` CLI. The runner bind-mounts the
binary into every worker container regardless of which plugins are attached;
what these two plugins add is a skill that teaches the commands. Whether the
command is present, and whether it is pointed at a deployment, are two separate
questions their skills answer.

## Roles

Two roles ship with the tier (`[exports] roles = "*"`). A role is a named bundle
of plugins applied to a worker.

### `executive`

> Executive PM agent — manages projects, coordinates workers, handles escalations.

Composes: `project-management`, `workers-orchestra`, `workers-collaboration` (all
from this repo) **plus** `core-memory` and `core-summary` from
[`twinkle-flow-core`](https://github.com/twinkle-hq/twinkle-flow-core). The result
is a worker that can drive the project lifecycle in Linear, spin up and steer
other workers, discover its team, and carry persistent memory + a structured
execution summary across sessions.

### `configurator`

> Agent configuration designer — creates, validates, and tests agent setups.

Composes: `workers-configurator` (from this repo) **plus** `core-memory` and
`core-summary` from
[`twinkle-flow-core`](https://github.com/twinkle-hq/twinkle-flow-core). The result
is a worker that can design and test new agent capabilities (plugins, roles,
flows, tasks, MCP entities) while remembering past design decisions and emitting
a summary.

### Cross-repo dependency (the `[dependencies]` block)

Both roles reference `core-memory` / `core-summary`, which live in a **separate
repo** (`twinkle-flow-core`). `flows.toml` declares
`[dependencies] "twinkle-hq/twinkle-flow-core" = "v0.1.0"` so that the
flows-service's MVS resolver clones and pins `twinkle-flow-core` into the merged
workspace view — the core plugins resolve correctly even though they are authored
elsewhere. (The dependency declaration does **not** silence the repo-local
`flows lint` cross-repo warnings — `.flows-lint.yaml` handles those.)

## MCP connectors

Three MCP connectors are declared under `mcp/`. All three are
**`connection_type: internal`** builtin services wired **on-demand** — the runner
registers a session against each one (using `auth.type: internal_secret`) only
for the worker sessions whose attached plugins declare them in `session.mcp`, and
writes the resulting token URL into the worker's `.mcp.json`.

| Connector | Backing service | Purpose | Used by |
|---|---|---|---|
| [`linear-mcp`](mcp/linear-mcp.yaml) | `linear-mcp-proxy` | Scoped Linear access via the internal MCP proxy | `project-management`, `workers-collaboration` |
| [`team-mcp`](mcp/team-mcp.yaml) | `flows` | Team discovery and manual flow dispatch | `workers-collaboration` |
| [`telegram-inbox`](mcp/telegram-inbox.yaml) | `telegram-inbox` | Receive Telegram messages as MCP tool calls | `telegram` |

## How to attach

The company tier is a per-workspace library, so attaching is two steps: attach
the repo to a workspace, then apply a role to a worker in that workspace.

1. **Attach the library** to the target workspace (or pick it from the
   attach-library preset dropdown — it's the first entry):

   ```bash
   curl -X POST http://localhost/api/config/workspaces/<ws>/libraries \
     -H 'Content-Type: application/json' \
     -d '{
       "git_url": "https://github.com/twinkle-hq/twinkle-flow-company.git",
       "branch": "main",
       "github_token_id": "sec_..."
     }'
   ```

   The flows-service resolves the workspace build list via MVS, which also clones
   and pins the declared `twinkle-flow-core` dependency.

2. **Apply a role to a worker** — set the worker's plugins to the role's
   composition (e.g. apply `executive` to a PM worker, `configurator` to a
   config-designer worker). On the next session the runner injects the role's
   plugins, registers the MCP connectors each plugin declares, and compiles the
   skills/capabilities/flows into the session.

## Per-plugin docs & autonomy ramp

- Each plugin has its own README under `plugins/<name>/README.md`
  (linked in the table above).
- [`docs/autonomy-ramp.md`](docs/autonomy-ramp.md) — how to roll up from a single
  configured worker to a fully autonomous, executive-coordinated project team.
