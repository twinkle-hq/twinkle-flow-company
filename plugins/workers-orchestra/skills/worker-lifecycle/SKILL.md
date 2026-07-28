---
name: worker-lifecycle
description: Drive worker lifecycle and flow dispatch through the twinkle CLI — list workers, check status, start or stop one, read a worker's capabilities and recent runs, list flows, dispatch a flow. Use when you need to see or change what a worker is doing.
---

# Worker Lifecycle

You reach the platform through the `twinkle` CLI, not through an MCP tool. It is
already on `PATH` in this session. Whether it is also POINTED at a deployment
depends on how this machine is wired, and there are three cases that fail in
three different ways — read "Where this CLI is pointed" below before you
diagnose any error from it. Never run `twinkle auth login`: nothing in a worker
session holds a credential to log in with, and the CLI's own error text will
suggest it anyway.

## The eight commands this plugin gives you

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

**The workspace is spelled two different ways above, and they are not
interchangeable.**

- `worker start`, `worker stop` and `worker capabilities` take the workspace
  and the worker as **positionals**. None of them declares a `--workspace`
  flag, and the parser is strict: write `twinkle worker start qa w1` and it
  runs, add that flag to it and the whole command exits 2 with
  `twinkle: worker start: Unknown option '--workspace'.`
- `worker list`, `flow list` and `run list` take `--workspace` as a **query
  filter**. There is no server-side binding to the workspace you are running
  in. But what OMITTING the flag does is different for `worker list` and for
  `flow list`, and only one of the two is harmless:
  - **`worker list` without it answers fleet-wide** — every worker in every
    workspace the platform knows about (measured on a live stand: 4 workers
    across 3 workspaces). The flag narrows an answer you would otherwise still
    get.
  - **`flow list` without it does NOT answer fleet-wide.** It answers the
    force-attach SHARED view, and a workspace's flows arrive through an
    attached *library*, not through force-attach — so on a real deployment it
    returns `[]` (measured: `[]` with no flag, 14 flows with
    `--workspace sandbox`). **Always pass `--workspace` to `flow list`.** An
    empty answer without the flag means "you omitted the flag", never "this
    workspace has no flows to dispatch" — and acting on the second reading is
    how an executive concludes there is nothing to run.
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

## Where this CLI is pointed — three deployments, three failure shapes

The runner decides this before your session starts, in `resolve_twinkle_env()`
(`services/controllers/runner/runner.sh`). What it emits sorts into three
observable shapes, and an error you cannot place in one of them you cannot
diagnose.

**1 — Single-host.** `CORE_API_URL` is empty, so the runner sets
`TWINKLE_URL=http://gateway`: the gateway's DEFAULT vhost, which carries no
auth subrequest. **All eight commands work**, admin-scoped ones included.

**2 — Remote machine, CLI wired.** `CORE_API_URL` and `WORKER_API_URL` are both
set. The runner points the CLI at the WORKER vhost and mints a
`scope: 'worker'` token, and that vhost admits only operations the registry
declares `scope: 'worker'`:

- `worker start`, `worker stop`, `worker capabilities`, `flow list` and
  `run list` are `scope: 'worker'` — they work.
- `worker list`, `worker status` (the same operation) and `flow dispatch` are
  `scope: 'admin'` — they answer **403 at the edge**, before reaching any
  service.

A 403 here is this session's scope: not a broken command, not a transient
failure, and no retry or flag changes it. Say so and use what you have.

**3 — Remote machine, CLI NOT wired.** `CORE_API_URL` is set and
`WORKER_API_URL` is empty. The runner logs `WORKER_API_URL unset on a remote
machine; the twinkle CLI is mounted but unconfigured` and then emits
**nothing** — no URL, no token. The CLI falls back to its compiled-in default
`http://localhost`, which inside this container is nothing at all. **This is
the state a remote machine stays in until an operator adds worker-vhost DNS and
`WORKER_API_URL` — an operator-only change no session can make.** It shows up
in two shapes:

- a read (`worker list`, `worker status`, `flow list`, `worker capabilities`,
  `run list`) prints `twinkle: no --url, --profile, TWINKLE_URL or stored
  profile — reading from the default vhost http://localhost` on stderr, then
  fails with a **connection error**;
- a write (`worker start`, `worker stop`, `flow dispatch`) is refused before
  any network call, with `no target named, and POST … would go to the default
  vhost http://localhost … Name it: --url <base>, --profile <name>,
  TWINKLE_URL, or twinkle auth login`.

**Do none of what that second message suggests.** There is no credential to log
in with, and no `--url` you can invent reaches the platform from here. This is
not a broken deployment and not something a retry fixes: the CLI is simply not
wired up on this machine. Report exactly that — "the twinkle CLI is unconfigured
on this machine" — and fall back to the non-CLI path for whatever you were
asked to do.

**Telling the three apart is the point of this section.** A `403` with a JSON
body is case 2 and means scope. A connection error, or any message naming
`http://localhost`, is case 3 and means unwired. Anything that answers is case
1. Reading case 3 as a broken platform and retrying it is the exact misdiagnosis
this section exists to prevent — and it is the case every remote machine is in
today.

`twinkle <group> <action> --help` prints each operation's `scope:` line.

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
- **`worker capabilities <ws> <w>`** — the dispatchable flows and capability
  tasks the platform resolved from this worker's own plugins. `flow dispatch`
  performs no capability check of its own: it dispatches any flow that exists
  in the workspace, whether or not it is listed here.
- **`flow list --workspace <ws>`** — flows defined for the workspace, by name.
  The flag is not optional in practice — see above.
- **`flow dispatch <name> --workspace <ws> --worker <w>`** — trigger a flow run
  addressed to one worker. This is dispatch, not delegation: it starts a DAG
  the flow itself defines, it does not create a one-off task from free text.
- **`run list --workspace <ws> --worker <w> --limit <n>`** — recent run rows for
  a worker: tokens, cost, duration, status, machine. This is the run row and
  only the run row — never the step-by-step behavioural trace — so use it to
  see whether recent work completed and what it cost, not to reconstruct how an
  agent worked.

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
- `403`, a connection error, or a message naming `http://localhost` — none of
  these is about the worker. They are deployment cases 2 and 3; go read "Where
  this CLI is pointed" before you retry anything.
