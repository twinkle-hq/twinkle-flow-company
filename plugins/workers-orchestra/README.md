# workers-orchestra

`tier: company` · per-workspace **library** plugin (attached to a workspace via
the company library — **not** force-attached).

A thin **connector** plugin: a `plugin.yaml` and the one skill it points at. It
ships no capabilities, no hooks, and no flows. Its single job is to teach the
agent the worker-lifecycle commands of the **`twinkle` CLI**:

> *Workers lifecycle — list workers, check status, start and stop them, list and
> dispatch flows, through the twinkle CLI.*

It teaches no `worker create`: creating a worker is `scope: 'admin'` and, with
no `--git_repo` and a GitHub token the BFF can resolve, creates a real private
GitHub repository. That is a configurator operation, not an executive one — see
[`workers-configurator`](../workers-configurator/README.md).

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
`twinkle` CLI (already on `PATH`, already pointed at the deployment by the
runner's `TWINKLE_URL` — no `twinkle auth login`) rather than through an MCP
tool.

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
`worker status` afterward).

Two details the skill spells out and this list cannot:

- **The workspace is spelled two ways.** It is a positional on `worker
  start`/`worker stop`/`worker status` and a `--workspace` flag on `worker
  list`/`flow list`/`flow dispatch`. Adding that flag to `twinkle worker start`
  exits 2. There is no server-side workspace binding, so the list commands
  answer fleet-wide without the flag.
- **Three of the six are `scope: 'admin'`** — `worker list`, `worker status`
  (the same operation) and `flow dispatch`. On a deployment where the runner
  points the CLI at the worker vhost, those answer `403` at the edge; on a
  single-host deployment, where it points at the default vhost, all six work.

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
