# workers-orchestra

`tier: company` · per-workspace **library** plugin (attached to a workspace via
the company library — **not** force-attached).

A thin **connector** plugin: `plugin.yaml` only. It ships no capabilities, no
hooks, and no flows — one skill. Its single job is to teach the agent the
worker-lifecycle commands of the **`twinkle` CLI**:

> *Workers lifecycle — create, start, stop workers, dispatch flows, list system
> state.*

## What it wires

```yaml
# plugin.yaml
session:
  skills:
    - skills/worker-lifecycle/SKILL.md
```

That one line compiles
[`skills/worker-lifecycle/SKILL.md`](skills/worker-lifecycle/SKILL.md) into the
session's `SKILL.md`, teaching the agent to reach the platform through the
`twinkle` CLI (already on `PATH`, already authenticated as the worker — no
`twinkle auth login`) rather than through an MCP tool.

### The `worker-lifecycle` skill

Six commands, taken verbatim from the skill:

```bash
twinkle worker list --workspace <ws>
twinkle worker status <ws> <w>
twinkle worker start <ws> <w>
twinkle worker stop <ws> <w>
twinkle flow list --workspace <ws>
twinkle flow dispatch <name> --workspace <ws> --worker <w>
```

`worker start`/`worker stop` are fire-and-forget (a `202` means the command
reached the bus, not that the worker has actually changed state — confirm with
`worker status` afterward). There is no server-side workspace binding, so every
invocation needs an explicit `--workspace`.

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
