# workers-orchestra

`tier: company` · per-workspace **library** plugin (attached to a workspace via
the company library — **not** force-attached).

A thin **connector** plugin: `plugin.yaml` only. It ships no skills, no
capabilities, no hooks, and no flows. Its single job is to wire the
**`orchestra-api`** MCP into the session so an agent can drive the worker
lifecycle:

> *Workers lifecycle — create, start, stop workers, dispatch flows, list system
> state.*

## What it wires

```yaml
# plugin.yaml
session:
  mcp:
    - orchestra-api
```

That one line adds the `orchestra-api` server to the session's `.mcp.json`,
giving the agent tools to create/start/stop workers, dispatch flows, and list
system state.

### The `orchestra-api` MCP

The MCP entity is defined in [`mcp/orchestra-api.yaml`](../../mcp/orchestra-api.yaml):

- `connection_type: internal` — a **builtin-internal** provider, served by the
  in-cluster `orchestra-mcp` service (`http://orchestra-mcp:4300`).
- Registered **on-demand**: the runner registers a session against the
  provider's `registration_endpoint` (`/register`, carrying `task_id`) and writes
  the returned `token` into the worker's `.mcp.json` before the session starts.
- `auth.type: internal_secret` — authenticated with the shared internal MCP
  secret; no per-workspace OAuth or credentials are required.

Because it is builtin-internal, the runner's MCP dispatch resolves and attaches
it without an explicit per-workspace integration row.

## Which role includes it

This plugin is part of the **`executive`** role
([`roles/executive.yaml`](../../roles/executive.yaml)) — the PM agent that manages
projects, coordinates workers, and handles escalations:

```yaml
# roles/executive.yaml
name: executive
plugins:
  - project-management
  - workers-orchestra
  - core-memory
  - core-summary
  - workers-collaboration
```

The `workers-orchestra` connector is what gives the executive its
worker-lifecycle and flow-dispatch reach.

## How to attach

`twinkle-flow-company` is a per-workspace library, not a global force-attach.
Attach it to a workspace and select the plugin (directly, or by selecting the
`executive` role which already pulls it in):

```bash
curl -X POST http://localhost/api/config/workspaces/<ws>/libraries \
  -H 'Content-Type: application/json' \
  -d '{"git_url": "https://github.com/twinkle-hq/twinkle-flow-company.git", "branch": "main"}'
```

See the repo [README](../../README.md) for the company-tier model and the full
plugin/role catalog.
