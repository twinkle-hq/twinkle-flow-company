# workers-collaboration

`tier: company` · per-workspace **library** (attach to a workspace; not
force-attached).

A thin **connector** plugin. It ships no skills, hooks, capabilities, or
gates — its entire job is to wire two MCP services into the session so the agent
can manage Linear tasks and discover its team / dispatch flows. The whole plugin
is a single `plugin.yaml`:

```yaml
name: workers-collaboration
description: "Linear task management + team discovery and flow dispatch"

session:
  mcp:
    - linear-mcp
    - team-mcp
```

## What it wires

`session.mcp` lists two MCP definitions from this repo's `mcp/` directory. Both
are `connection_type: internal` — they resolve to Orchestra's own internal
services and are registered on-demand by the runner (per task, with a scoped
session token) before the worker container starts; the plugin does not embed
any credentials.

| MCP | Source | What it gives the agent |
|---|---|---|
| `linear-mcp` | [`mcp/linear-mcp.yaml`](../../mcp/linear-mcp.yaml) | Scoped Linear access via the internal MCP proxy (`linear-mcp-proxy`). The runner registers a per-task session (`task_id`, `task_identifier`, `team_id`, `scope_labels`) and writes the scoped token + `mcp_url` into the worker's `.mcp.json`. |
| `team-mcp` | [`mcp/team-mcp.yaml`](../../mcp/team-mcp.yaml) | Team discovery and manual flow dispatch via the `flows` service. The runner registers a session with the worker's `role`, `task_id`, `visible_workers`, and `team_id`. |

Because both are `internal`, the actual Linear API key (held by
`linear-mcp-proxy`) and the flows dispatch surface never reach the worker
directly — the agent only ever sees a short-lived, scoped session token.

## Which role includes it

The repo's **executive** role
([`roles/executive.yaml`](../../roles/executive.yaml)) lists
`workers-collaboration` among its plugins — the executive PM agent manages
projects, coordinates workers, and handles escalations, so it needs both Linear
task management and team discovery / dispatch. The **configurator** role does
**not** include it.

## How to attach

This repo is a **company-tier library**, attached per workspace (not
force-attached). Two ways to get `workers-collaboration` into a worker's session:

1. **Via the role** — give the worker the `executive` role and the plugin comes
   in automatically (it is one of that role's plugins).
2. **Directly** — list `workers-collaboration` in the worker's `plugins[]` once
   this library is attached to the workspace.

Attach the library to a workspace (it is the first entry in the company preset
dropdown) and the runner resolves the merged view — including the two MCP
definitions this plugin references.

See the repo [README](../../README.md) for the company tier and the full plugin
list.
