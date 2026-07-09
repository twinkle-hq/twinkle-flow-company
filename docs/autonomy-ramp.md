# Autonomy ramp — the company tier

This is the operating philosophy of `twinkle-flow-company`, the **company /
project-running** tier of the Orchestra plugin catalog. Where
[`twinkle-flow-system`](https://github.com/twinkle-hq/twinkle-flow-system) is
life-support and [`twinkle-flow-core`](https://github.com/twinkle-hq/twinkle-flow-core)
is optional effectiveness, this tier is what turns the platform into a tool that
*runs projects*: it ships the **executive** role (`plugins/project-management`)
that drives the Linear project lifecycle, and the **configurator** role
(`plugins/workers-configurator`) that designs new agents and flows. Both roles
are per-workspace **library** attachments — opt-in, not force-attached.

This document is about the one thing that matters most when you hand work to
those roles: **where the human stays in the loop, and how that loop shrinks over
time.**

## 1. Philosophy — the human is the direction; the agents are the machinery

The thesis of this tier is deliberately un-magical:

> **The human is the source of direction and the ultimate validator. The agents
> run the machinery between human decision points.**

Nothing in this tier decides *what is worth doing* or *whether the result is
good enough* on its own authority. An executive agent does not invent
initiatives; it reacts to an initiative a human marked **Active**. It does not
ship a project plan on its own say-so; it drafts one and waits for a human to
move the project to **Approved**. When it gets stuck in a way it cannot
honestly resolve, it does not guess — it escalates and waits.

What the agents *do* own is the machinery in between: decomposing an approved
project into well-formed worker tasks, reviewing finished work against the
original requirements, diagnosing and unblocking stuck tasks, keeping status
updated, and noticing when a human decision is actually needed. That is real,
high-volume work — and it is exactly the work that is safe to automate, because
every consequential *judgement* still routes back through a human touchpoint.

The "ramp" in this document's title is the path from **supervised** operation
(a human reviews every intermediate artifact) to **autonomous** operation (the
human only sets direction and validates outcomes, and is pulled in only when the
machinery genuinely cannot proceed). The ramp is real and gradual, but two
touchpoints **never** retract, because they *are* the principle above:
**initiative activation** (the source of direction) and the **escalation
channel** (the safety valve that lets the machinery hand a genuine decision
back to its validator).

The rest of this document is an honest, structural map. Every flow, trigger,
filter, and gate named below exists in this repo's YAML — nothing is rounded up
or invented.

## 2. The executive ramp

The executive lifecycle is **Linear-status-mediated**: the human and the agents
communicate almost entirely by moving Linear entities between statuses and
attaching labels. There are exactly **8 Linear-triggered flows** in
`plugins/project-management/flows/` (each carries an `on.linear` block). The
table classifies each by *who initiates the trigger* and whether it is a true
human gate.

| Flow | Trigger (event + filter) | Who initiates | Human gate? | Permanent / retractable | Retract condition |
|---|---|---|---|---|---|
| `initiative-kickoff` | `initiative.create` / `initiative.update`, `status: Active` | **Human** (activates an initiative) | **Yes** — entry / source of direction | **Permanent** | — (this *is* the source of direction; never retracts) |
| `review-gate-flow` | `issue.create`, `status: Ready`, `labels: [executive, create-project]` | **Human or agent** (files a create-project request) | **Hybrid** entry point | Permanent as a channel | — (the request channel stays; who files it can shift) |
| `on-project-approved` | `project.update`, `status: Approved` | **Human** (approves a drafted project) | **Yes** — approval | **Retractable** | N consecutive clean, human-approved projects on an initiative → per-initiative auto-approve trust flag |
| `on-project-feedback` | `project.update`, `status: Changes Requested` | **Human** (requests changes) | **Yes** — validator / rejection path | **Retractable** (twin of approval) | Retracts together with `on-project-approved` once auto-approve trust is granted |
| `on-escalation-ready` | `issue.update`, `status: Ready`, `labels: [escalation]` | **Human** (answers an escalation) | **Yes** — escalation answer / safety valve | **Permanent** | — (channel is permanent; only its *frequency* is retractable) |
| `on-task-review` | `issue.update`, `status: In Review`, `labels: [review_required]` | **Worker / system** (a worker finished a review-required task) | **No** — autonomous | Already autonomous | — (no gate to retract) |
| `on-task-blocked` | `issue.update`, `status: Blocked` | **Worker / system** (a task blocked) | **No** — autonomous, with escalation fallback | Already autonomous | — (no gate to retract; may *itself* invoke the escalation channel) |
| `project-completion-flow` | `project.update`, `status: Completing` | **System** (the projects health-check dispatches it once all tasks are terminal) | **No** — autonomous | Already autonomous | — (no gate to retract) |

Plus **3 schedule-triggered flows** that are **fully autonomous** — no human
in the loop at all:

- `initiatives-health-check-flow` — `cron: 0 9 * * *` → `initiatives-health-check`
- `projects-health-check-flow` — `cron: 0 */4 * * *` → `projects-health-check`
- `project-progress-review-flow` — `cron: 0 12 * * *` → `project-progress-review`

Note that `project-completion-flow` is **Linear-triggered**, not scheduled: it
fires on `project.update` with `status: [Completing]`. It reaches the executive
autonomously because the `projects-health-check` task **dispatches** it once it
confirms every task in a project is in a terminal status — no human is involved,
but the trigger itself is a Linear event, so it belongs with the Linear-triggered
flows above, not with the cron flows.

So the verifiable structural count is: **8 Linear-mediated touchpoints — of
which 4 are true human gates, 1 is a hybrid entry point, and 3 are autonomous
(`on-task-review`, `on-task-blocked`, `project-completion-flow`) — plus 3
fully-autonomous schedule flows.** Resist the urge to miscount.

### The project lifecycle, end to end

Read top-to-bottom, the executive lifecycle is a chain of agent machinery
punctuated by exactly the human gates above:

1. **Initiative → Active** *(human gate #1)*. A human activates an initiative.
   `initiative-kickoff` fires; the executive analyses the initiative and **drafts
   projects**. This is the source of direction — the agent is reacting to a human
   decision about what is worth doing.

2. **Create-project request** *(hybrid entry #2)*. Independently, a create-project
   request can be filed as a Ready issue labeled `executive, create-project` — by
   a human *or* by another agent process. `review-gate-flow` runs `review-gate`,
   which drafts a project and sends it for human review. This is why it is hybrid:
   the *request* may originate either side, but the **review** still lands on a
   human.

3. **Project → Approved** *(human gate #3)*. A drafted project sits until a human
   moves it to **Approved**. That is the validator gate for the *plan*.
   `on-project-approved` then fires `task-decomposition`, which decomposes the
   approved project into worker tasks (creating each via the `create-worker-task`
   capability — the canonical, mandatory route for filing a worker task; the
   `task-decomposition` flow itself is intentionally `workflow_dispatch`-only to
   avoid double-firing on the same Approved event).

4. **Project → Changes Requested** *(human gate #4)*. If the human is *not*
   satisfied, they move the project to **Changes Requested** instead.
   `on-project-feedback` fires `handle-project-feedback`, which revises the
   project and resubmits for review. This is the rejection path — the other half
   of the same approval gate.

5. **Worker tasks run.** Workers pick up the decomposed tasks. Two task outcomes
   route back to the executive *without* a human:

   - **review-required → In Review** *(autonomous, #6)*. When a worker finishes a
     task labeled `review_required` and moves it to **In Review**,
     `on-task-review` fires `review-worker-task` and the executive **auto-reviews**
     it — accepting (→ Done), returning with feedback (→ Ready), or, after a
     bounded number of returns, escalating. This is the closed review loop; no
     human gate.

   - **Blocked** *(autonomous with escalation fallback, #7)*. When a task moves to
     **Blocked**, `on-task-blocked` fires `unblock-worker`, which diagnoses and
     attempts to resolve (rewrite the description, change the model, split the
     task). After its bounded attempt limit, or when confidence is too low, it
     **escalates to a human** through the channel described in §4. So the *machine*
     tries first; the *human* is pulled in only when the machine honestly cannot
     proceed.

6. **Health checks run in the background** *(fully autonomous)*. The three schedule
   flows scan initiatives, projects, and progress; detect completion; and feed the
   completion/cleanup paths — `projects-health-check`, once it confirms every task
   in a project is terminal, **dispatches** the Linear-triggered
   `project-completion-flow`. None of these gate on a human — but any of the
   underlying tasks can still *open* an escalation if it finds a decision only a
   human can make (see §4).

## 3. The configurator ramp

The configurator role (`plugins/workers-configurator`) ships **two** flows, and
together they *are* the ramp made visible — one supervised, one autonomous,
side by side.

### `flow-creation` — 4 explicit `type: human` gates

`flow-creation.yaml` interleaves four `type: human` review jobs between the
agent jobs. In dependency order:

| Gate | After agent job | Message |
|---|---|---|
| **`review-brief`** | `capture-and-design` | "Review the captured requirements and entity-design brief before drafting begins." |
| **`review-draft`** | `draft` (`draft-entities`) | "Review the drafted YAML entities before semantic validation and review." |
| **`review-validation`** | `validate-and-review` | "Review the validation and multi-lens review results before sandbox testing." |
| **`review-tests`** | `test-and-iterate` | "Review the sandbox test results and iterations before the configuration is considered production-ready." |

This is the **supervised** end of the ramp: a human eyeballs every intermediate
artifact — the requirements brief, the drafted YAML, the validation/review
results, and the sandbox test results — before the next stage of machinery runs.

### `design-agent` — 0 human gates (the target state)

`design-agent.yaml` runs `design` → `test` (`design-agent` → `test-agent`) with
**no `type: human` gate anywhere**. It is fully autonomous: the configurator
designs an agent, then tests it, end to end, with no human checkpoint.

`design-agent` is not a different *kind* of work from `flow-creation` — it is
what `flow-creation` **converges on** once the gates are earned away. It is the
concrete picture of the target state.

### How the configurator gates retract — late to early

The configurator gates retract **back to front** (LATE → EARLY), because trust
accrues from the outside of the pipeline inward. The order is:

1. **Drop `review-tests` first.** Once sandbox testing is itself trusted to catch
   regressions, the human no longer needs to re-read the test results — the test
   stage is the validator.
2. **Then drop `review-validation`.** Once the validate-and-review stage is trusted
   to surface real semantic problems.
3. **Then drop `review-draft`.** Once the drafted YAML is reliably well-formed.
4. **Finally drop `review-brief`.** The requirements brief is the *last* gate to
   go, because misunderstanding the requirements is the most expensive failure —
   it is the configurator's analogue of the executive's initiative-activation
   gate.

When all four are gone, `flow-creation` has converged on the `design-agent`
model. **Retract condition for each gate: N consecutive clean, human-approved
runs through that gate** — the same trust-by-track-record rule the executive's
approval gate uses.

## 4. The escalation safety valve

The escalation channel is the one piece of machinery whose whole purpose is to
*hand control back to the human*. It is not a structural `on:` gate on any single
flow — it is a **dynamic** channel that any executive task can open at runtime:

- `gather-input` writes a `task/result.md` with `status: needs_human` and a
  `questions:` block, ending the session; the human's answers come back in
  `task/context.json → feedback[]` and the session resumes.
- `defer-and-delegate` creates an `[Escalation]`-titled issue on the executive
  label in **Blocked** status when the helper work is a human decision.
- `unblock-worker` and `review-worker-task` both fall through to
  `defer-and-delegate` after their bounded attempt/return limits — so a task the
  machine genuinely cannot resolve becomes an escalation rather than a silent
  failure or an infinite retry.

Every one of those round-trips back through the **`on-escalation-ready`** flow
(gate #5): the human answers by moving the escalation issue back to **Ready**,
which fires `handle-escalation` to verify the answer is sufficient and either
resolve (unblocking the continuation) or ask a focused follow-up.

The classification is sharp and worth stating plainly:

- **The channel is permanent.** You may never remove the ability for the
  machinery to hand a genuine decision back to its validator. Removing it would
  break the core principle — the agents would have to *guess* at decisions that
  are the human's to make.
- **The frequency is retractable.** A well-trusted, well-journaled executive on a
  familiar initiative will escalate *less* — it has seen similar decisions before
  (`journal_search` over past escalations is a first-class step in
  `handle-escalation`, `unblock-worker`, and `defer-and-delegate`). Lowering the
  escalation rate is the *desired* outcome of the ramp. Removing the channel is
  not.

## 5. Today vs gated-today vs retractable-next

A consolidated, honest summary of where each touchpoint stands.

### Autonomous today — no human in the loop, nothing to retract

- `on-task-review` (#6) — the executive auto-reviews `review_required` tasks.
- `on-task-blocked` (#7) — the executive auto-unblocks Blocked tasks (escalating
  only on genuine impasse).
- `project-completion-flow` (#8) — Linear-triggered (`status: Completing`) but
  dispatched autonomously by `projects-health-check`; no human in the loop.
- The 3 schedule flows — `initiatives-health-check-flow`,
  `projects-health-check-flow`, `project-progress-review-flow`.
- The configurator's `design-agent` flow — design → test, no gate.

### Gated today — a human touchpoint, by design

- **Initiative activation** (`initiative-kickoff`, #1) — *permanent*.
- **Create-project request** (`review-gate-flow`, #2) — hybrid entry; the review
  still lands on a human.
- **Project approval** (`on-project-approved`, #3) and its **feedback twin**
  (`on-project-feedback`, #4) — *retractable*.
- **Escalation answer** (`on-escalation-ready`, #5) — *permanent channel*.
- The configurator's **4 `flow-creation` gates** — `review-brief`,
  `review-draft`, `review-validation`, `review-tests` — *retractable*.

### Retractable next — the path the ramp walks

- **Project approval + feedback** → auto-approve once an executive proves a
  consistently-high human-approval rate on an initiative (a **per-initiative**
  trust flag, so trust is earned per body of work, not granted globally).
- **Configurator gates** → drop **late → early** (`review-tests`, then
  `review-validation`, then `review-draft`, then `review-brief`), converging on
  the gate-free `design-agent` model. Retract condition: **N consecutive clean,
  human-approved runs** through each gate.
- **Escalation frequency** → lowers naturally as the executive's journal of past
  decisions deepens. The channel itself does not retract.

### Never retracts — the principle, restated

- **Initiative activation** (#1) — the human remains the source of direction.
- **The escalation channel** (#5 + the dynamic `needs_human` / `[Escalation]`
  paths) — the human remains the ultimate validator for any decision the
  machinery cannot honestly make on its own.

Everything else is machinery, and machinery is exactly what this tier is here to
run.
