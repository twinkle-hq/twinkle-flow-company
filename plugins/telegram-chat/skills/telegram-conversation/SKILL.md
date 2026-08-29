---
name: telegram-conversation
description: The task you are working on is one Telegram thread with one person, and your session result is the message they read. Use when a task carries messages from a person in its feedback and you are about to write result.md — it decides what goes in `summary` and what goes in `questions`.
---

# You Are In A Conversation

This worker has a Telegram bot attached to it. When it does, a task is not a
ticket someone filed and walked away from — **it is one thread with one person,
and that person is waiting**. The thread does not close: they can reply to
anything you say, and their reply brings the task back to you.

There is **no Telegram tool**. Do not look for one, and do not report that you
could not find one. Both directions of the conversation travel through the task:

| direction | where it is |
|---|---|
| what they said | `task/context.json` → `feedback[]` |
| what you say | `task/result.md` → `summary`, or `questions` |

## Reading what they said

`feedback[]` is every comment on the task, oldest first. Each entry:

```json
{ "id": "…", "parent_id": "…", "author": "…", "date": "…", "body": "…" }
```

`parent_id` is what makes it a conversation rather than a list. An entry whose
`parent_id` matches something you said earlier is **a reply to that**, not a new
subject — read the pair together before you answer.

Two things in `feedback[]` are not a person talking to you:

- **Machine records.** Some comments are a single line of JSON (`{"event":
  "completed", …}`). That is the runner's own bookkeeping. Skip them.
- **Your own earlier messages**, which appear here the same as theirs. `author`
  tells you which is which.

## Writing what they read

**On `status: done`, your `summary` IS the message.** It is posted to the task
and delivered to the thread verbatim — nobody edits it, summarises it, or wraps
it in anything. Write it as a reply to a person:

- Answer what they actually asked, first. Not what you did first.
- Second person. "I've moved the deploy to Friday" — not "The deploy has been
  rescheduled" and not "Task completed successfully."
- Telegram length. A few sentences. They are reading it on a phone.
- No run statistics, no durations, no file lists, no headings, no status
  banners. If they want the detail they will ask, and asking is cheap now.
- Do not replay the thread back at them. They can scroll.
- **Never leave `summary` empty when a person is waiting.** An empty summary
  sends nothing at all, and from their side that is silence, not completion.

**On `status: needs_human`, your `summary` is NOT what reaches them — your
`questions` are.** Each entry in `questions[]` becomes its own message. Put the
whole ask in the question itself; a question that only makes sense next to a
summary they cannot see reads as a non-sequitur.

```yaml
---
status: needs_human
summary: Blocked on which environment to deploy to.
questions:
  - Should this go to staging first, or straight to production?
---
```

**Ask one thing at a time.** The conversation comes back to you only when
**every** question you asked has been answered. Three questions is three
messages the person must all reply to before you hear anything; the third one
they skip holds the other two hostage. If you genuinely need three answers, ask
for the one that unblocks you and ask the rest later — there is a later now.

## What happens next

You reply and the task closes. That is not the end of the thread: when they
answer, the task comes back to Ready and you pick it up again with their message
in `feedback[]`. So a reply of "let me know if you want me to also do X" is a
real offer, not a formality — they can take you up on it, and you will see it.
