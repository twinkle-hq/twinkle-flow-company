# telegram-chat

`tier: company` · per-workspace **library** plugin (attached to a workspace via
the company library — **not** force-attached).

A **framing** plugin: a `plugin.yaml` and the one skill it points at. It wires
**no MCP**, no capabilities, no hooks, and no flows. Its single job is to tell
the agent that the task in front of it is a conversation with a person on
Telegram, so that its session result reads as a reply rather than a work report.

```yaml
# plugin.yaml
session:
  skills:
    - skills/telegram-conversation/SKILL.md
```

## It replaces the `telegram` plugin, and the difference is the whole point

The old `telegram` plugin was a connector: `session.mcp: [telegram-inbox]`, five
`inbox_*` tools, and an agent that had to remember to call one. Telegram is no
longer a tool the agent reaches for — it is the shape of the task itself:

| | old `telegram` | this plugin |
|---|---|---|
| inbound | `inbox_list` / `inbox_read` MCP calls | `task/context.json` → `feedback[]` |
| outbound | `inbox_reply` MCP call | `result.md` → `summary` (or `questions`) |
| "handled" | an `inbox_mark_handled` call | the task's state |
| what the plugin wires | an MCP server | a skill, and nothing else |

Nothing in this plugin is load-bearing for delivery. The thread ↔ task binding
lives in the platform's `telegram-chat` service; a worker with a bot attached
holds a conversation whether or not this plugin is on. What the plugin changes
is **how the reply is written** — without it the agent writes "Task completed
successfully. Modified 3 files." to someone who asked when the deploy is.

## The `telegram-conversation` skill

[`skills/telegram-conversation/SKILL.md`](skills/telegram-conversation/SKILL.md).
Four things it tells the agent, each of which it otherwise gets wrong:

- **There is no Telegram tool.** Both directions travel through the task files.
  Without this, an agent hunts for one and then reports that it is missing.
- **On `status: done`, `summary` IS the message** — delivered verbatim, read on
  a phone. So: answer the question first, second person, a few sentences, no run
  statistics. An empty `summary` sends nothing at all, which from the person's
  side is silence rather than completion.
- **On `status: needs_human`, `summary` is NOT what reaches them — `questions`
  are.** Each question becomes its own message, so a question that only makes
  sense beside an unseen summary reads as a non-sequitur.
- **Ask one thing at a time.** The conversation returns to the agent only when
  *every* question it asked has been answered; the third question the person
  skips holds the first two hostage.

It also names the two things in `feedback[]` that are not a person talking: the
runner's own single-line JSON event comments, and the agent's own earlier
messages, which appear there identically and are told apart by `author`.

## Tier & attach model

Company tier — a **per-workspace library**, not a force-attach. It is not
injected into every session. To make it available to a worker, attach the
company library to the workspace and then either add `telegram-chat` to the
worker's `plugins` array in config-service, or add it to a role the worker
carries. **None of the bundled roles include it** — neither
[`executive`](../../roles/executive.yaml) nor
[`configurator`](../../roles/configurator.yaml) lists it, and neither pulls it
in automatically.

Attaching this plugin does **not** give the worker a bot. The bot is configured
on the worker row in config-service (`telegram_bot_secret_id` +
`telegram_owner_chat_id`, with the token in secret-store). A worker with a bot
and without this plugin still holds conversations, just less well; a worker with
this plugin and no bot has a skill about a surface it does not have.
