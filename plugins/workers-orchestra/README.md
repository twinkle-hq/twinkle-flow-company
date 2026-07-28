# workers-orchestra

`tier: company` · per-workspace **library** plugin (attached to a workspace via
the company library — **not** force-attached).

A thin **connector** plugin: a `plugin.yaml` and the one skill it points at. It
ships no capabilities, no hooks, and no flows. Its single job is to teach the
agent the worker-lifecycle commands of the **`twinkle` CLI**:

> *Workers lifecycle — list workers, check status, start and stop them, list and
> dispatch flows, through the twinkle CLI.*

It teaches no `worker create`. That command prints `scope: admin`, and it is a
configurator operation rather than an executive one — see
[`workers-configurator`](../workers-configurator/README.md), which carries the
`--git_repo` rule that goes with it.

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
`twinkle` CLI rather than through an MCP tool — and to treat both "is there a
`twinkle` command here" and "is it pointed at a deployment" as open questions
answered by the four shapes below (never `twinkle auth login`).

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

Four details the skill spells out and this list cannot:

- **The workspace is spelled two ways.** It is a positional on `worker
  start`/`worker stop`/`worker status`/`worker capabilities` and a
  `--workspace` flag on `worker list`/`flow list`/`run list`/`flow dispatch`.
  Adding that flag to `twinkle worker start` exits 2 with
  `Unknown option '--workspace'`.
- **Omitting `--workspace` is harmless on `worker list` and wrong on `flow
  list`.** Measured on a live stand: `worker list` without it answered
  fleet-wide (4 workers across 3 workspaces); `flow list` without it answered
  `[]`, and `--workspace sandbox` answered 14 flows. Always pass `--workspace`
  to `flow list`; read an empty answer that came back without it as "the flag
  was omitted", never as "the workspace has no flows".
- **A stopped worker's `active` list is not necessarily empty.** Measured on a
  live stand over three workers all reading `"state":"stopped"`: two answered
  `"active":[]` and `qa/qa-worker` answered two entries. The skill's
  instruction is to run `worker capabilities` before `flow dispatch` and to
  dispatch only what it lists — and never to start or stop a worker in order to
  change the answer.
- **Three of the eight print `scope: admin`** — `worker list`, `worker status`
  (the same operation) and `flow dispatch`.
- **An answer comes back in one of four shapes, not two.** It answers; or it is
  a `403` with a JSON body, which is scope; or it is a connection error or a
  message naming `http://localhost`, and the CLI is not pointed at a deployment
  on this machine; or it is a `command not found` naming `twinkle`, and there
  is no CLI on this machine at all. The skill teaches the agent to read the
  last two as "report it and fall back" rather than as a broken platform to
  retry.

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
