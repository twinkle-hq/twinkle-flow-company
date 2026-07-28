---
name: worker-lifecycle
description: Drive worker lifecycle and flow dispatch through the twinkle CLI — list workers, check status, start or stop one, read a worker's capabilities and recent runs, list flows, dispatch a flow. Use when you need to see or change what a worker is doing.
---

# Worker Lifecycle

You reach the platform through the `twinkle` CLI, not through an MCP tool.
Whether the command exists on this machine, and whether it is POINTED at a
deployment, are two separate things and neither is a given — an answer comes
back in one of four shapes, and one of them is that there is no `twinkle`
command at all. Read "Four shapes an answer can take" below before you diagnose
any error from it. **Never run `twinkle auth login`**, however strongly the
CLI's own error text suggests it.

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
  filter**. But what OMITTING the flag does is different for `worker list` and
  for `flow list`, and only one of the two is harmless:
  - **`worker list` without it answers fleet-wide** — every worker in every
    workspace the platform knows about (measured on a live stand: 4 workers
    across 3 workspaces). The flag narrows an answer you would otherwise still
    get.
  - **`flow list` without it does NOT answer fleet-wide.** Measured on the same
    stand: `[]` with no flag, 14 flows with `--workspace sandbox`. **Always
    pass `--workspace` to `flow list`.** Read an empty answer that came back
    without the flag as "the flag was omitted" — never as "this workspace has
    no flows to dispatch". Re-run it with `--workspace` before you report that
    there is nothing to run.
- `worker status <ws> <w>` takes both. Its `--help` prints
  `operation: worker.list`, `GET /api/bff/workers` — the two positionals are
  the CLI's own client-side filter over that answer. Passing `--workspace`
  narrows what the server sends; the positionals are what pick the row.
- `flow dispatch <name>` takes the flow name as a positional and `--workspace` /
  `--worker` as flags.

When in doubt run `twinkle <group> <action> --help`: it prints the command's
signature, its flag list and its `scope:` line. **With one exception, and it is
`worker status`.** Its help prints `twinkle worker status [--flags]` and offers
only `--workspace`; the two positionals appear nowhere in that output. Type
them anyway — they are what selects the row.

Five of the eight are `scope: 'worker'` — `worker start`, `worker stop`,
`worker capabilities`, `flow list`, `run list`. Three are **`scope: 'admin'`** —
`worker list`, `worker status` (whose help prints `operation: worker.list`) and
`flow dispatch`. Run the `--help` and read the `scope:` line rather than
trusting this paragraph.

## Four shapes an answer can take

Place every answer in one of these four before you diagnose it.

**1 — It answers.** All eight commands are available to you in this session.

**2 — `403` with a JSON body.** That is this session's scope: not a broken
command, not a transient failure, and no retry or flag changes it. The three
`scope: 'admin'` commands above are the ones that can come back this way. Say
so and use what you have.

**3 — A connection error, or any message naming `http://localhost`.** The CLI
is not pointed at a deployment on this machine. A read warns on stderr first —
measured verbatim: `twinkle: no --url, --profile, TWINKLE_URL or stored profile
— reading from the default vhost http://localhost`.

**Never run `twinkle auth login` and never invent a `--url`**, however strongly
the CLI's own error text suggests them: there is no credential to log in with
and no address you can supply that reaches the platform from here. This is not
a broken deployment and not something a retry fixes. Report exactly that — "the
twinkle CLI is unconfigured on this machine" — and fall back to the non-CLI
path for whatever you were asked to do.

**4 — A `command not found` error naming `twinkle`.** Measured in a shell with
no such binary: `bash: twinkle: command not found`, exit status 127. There is
no `twinkle` command here. **Do not install one, do not build one, do not
retry, and do not run `twinkle auth login`.** Report "there is no twinkle
command on this machine" and fall back to the non-CLI path — the same fallback
shape 3 gets, on a different fact.

**Telling the four apart is the point of this section.** Reading shape 3 or
shape 4 as a broken platform and retrying it is the exact misdiagnosis this
section exists to prevent.

`twinkle <group> <action> --help` prints each operation's `scope:` line.

## What each one does

- **`worker list --workspace <ws>`** — every worker in the workspace, with its
  state. Start here when you need to know what exists before acting on one.
- **`worker status <ws> <w>`** — one worker's state, selected from the fleet
  list after it arrives (see above). Add `--json` for `.state` /
  `.machine_online` (measured on a live stand: `{"name":"executive",
  "workspace":"sandbox","state":"stopped", …,"machine_online":true}`). One read
  can lag what is actually true — re-read it rather than treating a single
  answer as final.
- **`worker start <ws> <w>` / `worker stop <ws> <w>`** — fire-and-forget.
  Confirm the worker actually changed state with `worker status` afterward;
  never treat the call returning as the worker being started or stopped.
- **`worker capabilities <ws> <w>`** — **run this before `flow dispatch`, and
  dispatch to a worker only a flow this command lists for it.** It answers
  `{worker, description, active}`. Measured on a live stand, over three workers
  that all read `"state":"stopped"` in `worker list`: `sandbox/executive`
  answered `{"worker":"executive","description":"","active":[]}`,
  `sandbox/configurator` answered `"active":[]`, and `qa/qa-worker` answered two
  entries — a flow `qa-review-gate` and a task `qa-parallel-review`. **Do not
  start or stop a worker in order to change this answer.** An empty `active` is
  an answer, not an error and not a retry: report it and do not dispatch.
- **`flow list --workspace <ws>`** — flows defined for the workspace, by name.
  The flag is not optional in practice — see above.
- **`flow dispatch <name> --workspace <ws> --worker <w>`** — trigger a run of a
  flow that already exists, addressed to one worker. It is not a way to hand a
  worker a one-off task from free text.
- **`run list --workspace <ws> --worker <w> --limit <n>`** — recent run rows for
  a worker. Measured on a live stand, a row carries `status`, `started_at`,
  `completed_at`, `duration_seconds`, `input_tokens`, `output_tokens`,
  `cost_usd`, `summary`, `machine` — and no step-by-step behavioural trace. Use
  it to see whether recent work completed and what it cost, not to reconstruct
  how an agent worked.

## Before dispatching or starting

Check `worker status` (or `worker list`) before `flow dispatch`, and
`worker start` first if the worker is not already running. A task addressed to
a stopped worker is not picked up.

## Error shapes

Every failure sorts into the four shapes above, and none of the four is a
statement about the worker:

- **`403` with a JSON body** — scope. Report it; do not retry.
- **A connection error, or any message naming `http://localhost`** — the CLI is
  unconfigured here. Report it; do not retry.
- **A `command not found` naming `twinkle`** — there is no CLI here. Report it;
  do not retry and do not install one.
- **Anything else** — do not read a failed `worker start`, `worker stop` or
  `flow dispatch` as "this worker does not exist". Re-read `worker status`
  before concluding anything about the worker itself.
