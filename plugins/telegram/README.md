# telegram

A thin **connector** plugin in the company tier. Its only job is to wire the
**Telegram inbox** into a worker session so the agent can receive and process
Telegram messages over MCP. It carries no flows, tasks, skills, capabilities,
hooks, or gates — just a single `session.mcp` entry.

```yaml
# plugin.yaml
name: telegram
description: "Telegram inbox for receiving messages via MCP"

session:
  mcp:
    - telegram-inbox
```

## What it wires

| Mechanism | Value | Effect |
|---|---|---|
| `session.mcp` | `telegram-inbox` | Registers the **telegram-inbox** MCP server into the session's `.mcp.json`, exposing the Telegram inbox tools to the agent |

`telegram-inbox` is a **builtin, internal** MCP provider (it ships with the
platform; the runner registers a session against it and writes the server entry
into the worker's `.mcp.json`). It is wired **on-demand** — the inbox tools reach
the agent only in sessions of workers that actually attach this plugin. A worker
that never attaches `telegram` gets no Telegram tools.

## Tier & attach model

This plugin lives in the **company tier**, which is a **per-workspace library**,
not a force-attach. It is not injected into every session. To make it available
to a worker you attach it explicitly, by either:

- **A role** — add `telegram` to a role's `plugins` list, then assign that role
  to the worker. **None of the bundled roles include it by default.** The two
  roles shipped in this repo — [`executive`](../../roles/executive.yaml) and
  [`configurator`](../../roles/configurator.yaml) — do **not** list `telegram`,
  so neither pulls it in automatically.
- **The worker plugin list** — add `telegram` directly to the worker's `plugins`
  array in config-service.

Either way, the company library must first be attached to the workspace
(`workspace_libraries`) so the plugin source is part of the merged build view.
