---
name: worker-lifecycle
description: Drive worker lifecycle and flow dispatch through the twinkle CLI — list workers, check status, start or stop one, list flows, dispatch a flow. Use when you need to see or change what a worker is doing.
---

# Worker Lifecycle

You reach the platform through the `twinkle` CLI, not through an MCP tool. It is
already on `PATH` in this session and already authenticated as this worker — do
not run `twinkle auth login`.

## The six commands this plugin gives you

```bash
twinkle worker list --workspace <ws>
twinkle worker status <ws> <w>
twinkle worker start <ws> <w>
twinkle worker stop <ws> <w>
twinkle flow list --workspace <ws>
twinkle flow dispatch <name> --workspace <ws> --worker <w>
```

Always pass `--workspace`. Every one of these without it answers for **every
workspace the platform knows about**, not just yours — there is no server-side
binding to the workspace you happen to be running in.

## What each one does

- **`worker list --workspace <ws>`** — every worker in the workspace, with its
  state. Start here when you need to know what exists before acting on one.
- **`worker status <ws> <w>`** — one worker's state. Add `--json` for
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
