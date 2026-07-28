---
name: worker-lifecycle
description: Drive worker lifecycle and flow dispatch through the twinkle CLI — list workers, check status, start or stop one, list flows, dispatch a flow. Use when you need to see or change what a worker is doing.
---

# Worker Lifecycle

You reach the platform through the `twinkle` CLI, not through an MCP tool. It is
already on `PATH` in this session and the runner has already pointed it at this
deployment — it sets `TWINKLE_URL` in the container, and on a remote machine a
`scope: 'worker'` token with it. Do not run `twinkle auth login`.

## The six commands this plugin gives you

```bash
twinkle worker list --workspace <ws>
twinkle worker status <ws> <w>
twinkle worker start <ws> <w>
twinkle worker stop <ws> <w>
twinkle flow list --workspace <ws>
twinkle flow dispatch <name> --workspace <ws> --worker <w>
```

**The workspace is spelled two different ways above, and they are not
interchangeable.**

- `worker start` and `worker stop` take the workspace and the worker as
  **positionals**. Neither declares a `--workspace` flag, and the parser is
  strict: write `twinkle worker start qa w1` and it runs, add that flag to it
  and the whole command exits 2 with
  `twinkle: worker start: Unknown option '--workspace'.`
- `worker list` and `flow list` take `--workspace` as a **query filter**.
  Without it they answer for every workspace the platform knows about — there
  is no server-side binding to the workspace you are running in.
- `worker status <ws> <w>` takes both: the two positionals are applied
  **client-side**, because no per-worker status endpoint exists anywhere in the
  system and the command is an alias onto the fleet-wide worker list. Passing
  `--workspace` narrows what the server sends; the positionals are what pick
  the row.
- `flow dispatch <name>` takes the flow name as a positional and `--workspace` /
  `--worker` as flags.

When in doubt run `twinkle <group> <action> --help`: it prints the command's
signature, its flag list and its `scope:` line. **With one exception, and it is
`worker status`.** Because that command is an alias onto `worker.list`, whose
path has no parameters, its help prints `twinkle worker status [--flags]` and
offers only `--workspace` — the two positionals are the CLI's own client-side
filter and appear nowhere in that output. Type them anyway; they are what
selects the row.

## Three of these six need more than a worker token

Your CLI is authenticated **as this worker**. Where the runner reaches the
platform over the network it points the CLI at the worker vhost with a
`scope: 'worker'` token, and that vhost admits only operations the registry
declares `scope: 'worker'`. Of the six above:

- `worker start`, `worker stop`, `flow list` are `scope: 'worker'` — they work.
- `worker list`, `worker status` (same operation) and `flow dispatch` are
  `scope: 'admin'` — on such a deployment they answer **403 at the edge**,
  before reaching any service.

On a single-host deployment the runner points the CLI at the default vhost,
which has no auth subrequest, and all six work. So a 403 here is not a broken
command and not a transient failure: it is this session's scope, and no retry
or flag changes it. Say so and use what you have — `worker start`/`worker stop`
still work, and `flow list` still tells you what exists. `twinkle <group>
<action> --help` prints each operation's `scope:` line.

## What each one does

- **`worker list --workspace <ws>`** — every worker in the workspace, with its
  state. Start here when you need to know what exists before acting on one.
- **`worker status <ws> <w>`** — one worker's state, selected from the fleet
  list after it arrives (see above). Add `--json` for
  `.state` / `.machine_online`. `machine_online` is derived from the machine's
  heartbeat (every 15s, stale past 45s), so a status read can lag reality by up
  to that much.
- **`worker start <ws> <w>` / `worker stop <ws> <w>`** — fire-and-forget: the
  202 response means the command reached the bus, not that the worker is
  already running or stopped. There is no poll-by-command endpoint — confirm
  with `worker status` afterward, not by assuming the call succeeded.
- **`flow list --workspace <ws>`** — flows defined for the workspace, by name.
- **`flow dispatch <name> --workspace <ws> --worker <w>`** — trigger a flow run
  addressed to one worker. This is dispatch, not delegation: it starts a DAG
  the flow itself defines, it does not create a one-off task from free text.

## Before dispatching or starting

A task addressed to a stopped worker sits Ready forever — nothing in this
system starts a worker on task arrival. Check `worker status` (or `worker
list`) before `flow dispatch`, and `worker start` first if it is not already
running.

## Error shapes

- `409 {"error":"machine_offline"}` — the worker's machine has not sent a
  heartbeat in the last 45 seconds.
- `404` — unknown workspace or worker.
- `503 {"error":"command_bus_unavailable"}` — the command bus is down; retry
  later, this is not a "worker doesn't exist" answer.
