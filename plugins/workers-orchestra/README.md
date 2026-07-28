# workers-orchestra

`tier: company` · per-workspace **library** plugin (attached to a workspace via
the company library — **not** force-attached).

A thin **connector** plugin: a `plugin.yaml` and the one skill it points at. It
ships no capabilities, no hooks, and no flows. Its single job is to teach the
agent the worker-lifecycle commands of the **`twinkle` CLI**:

> *Workers lifecycle — list workers, check status, start and stop them, list and
> dispatch flows, through the twinkle CLI.*

It teaches no `worker create`: that command prints `scope: admin`, and with no
`--git_repo` it can create a real private GitHub repository that nothing in the
CLI ever deletes. It is a configurator operation, not an executive one — see
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
`twinkle` CLI (already on `PATH`; whether it is also pointed at a deployment
depends on how the machine is wired — never `twinkle auth login`) rather than
through an MCP tool.

### The `worker-lifecycle` skill

Eight commands, taken verbatim from the skill:

```bash
twinkle worker list --workspace <ws>
twinkle worker status <ws> <w>
twinkle worker start <ws> <w>
twinkle worker stop <ws> <w>
twinkle worker capabilities <ws> <w>
twinkle flow list --workspace <ws>
twinkle flow dispatch <name> --workspace <ws> --worker <w>
twinkle run list --workspace <ws> --worker <w> --limit <n>
```

`worker start`/`worker stop` are fire-and-forget — the skill's instruction is to
confirm with `worker status` afterward rather than treat the call returning as
the worker having changed state.

Three details the skill spells out and this list cannot:

- **The workspace is spelled two ways.** It is a positional on `worker
  start`/`worker stop`/`worker status`/`worker capabilities` and a
  `--workspace` flag on `worker list`/`flow list`/`run list`/`flow dispatch`.
  Adding that flag to `twinkle worker start` exits 2 with
  `Unknown option '--workspace'`.
- **Omitting `--workspace` is harmless on `worker list` and wrong on `flow
  list`.** Measured on a live stand: `worker list` without it answered
  fleet-wide (4 workers across 3 workspaces); `flow list` without it answered
  `[]`, and `--workspace sandbox` answered 14 flows. Always pass `--workspace`
  to `flow list`; an empty answer without it means the flag was omitted, not
  that the workspace has no flows.
- **Three of the eight print `scope: admin`** — `worker list`, `worker status`
  (the same operation) and `flow dispatch` — and an answer comes back in one of
  three shapes, not two. It answers; or it is a `403` with a JSON body, which
  is scope; or it is a connection error or a message naming `http://localhost`,
  which means the CLI is not pointed at a deployment on this machine. The skill
  teaches the agent to read that third shape as "unwired, report it" rather
  than as a broken platform to retry.

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
