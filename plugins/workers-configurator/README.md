# workers-configurator

`tier: company` · per-workspace **library** plugin (attached to a workspace, NOT
force-attached).

The **meta-configurator**: an agent that designs, creates, validates, reviews,
and tests *other* agents' Orchestra configurations. Where the rest of the catalog
makes a worker session run, this plugin makes a worker session that **builds the
catalog itself** — it authors the very entity types Orchestra is made of
(plugins, roles, flows, tasks, MCP entities) by editing YAML in its own git repo
and validating with the repo-local `flows` CLI.

It is the configurator role's core plugin: `roles/configurator.yaml` composes
`core-memory`, **`workers-configurator`**, and `core-summary`.

## Tier & attach model

This repo (`twinkle-flow-company`) is the **company** tier — a per-workspace
**library**, attached opt-in to a specific workspace
(`POST /api/config/workspaces/<ws>/libraries`), not globally force-attached.
`workers-configurator` reaches a worker only when an admin attaches this library
to that workspace. Contrast the force-attach tiers:
[`twinkle-flow-system`](https://github.com/twinkle-hq/twinkle-flow-system) (locked
life-support) and [`twinkle-flow-core`](https://github.com/twinkle-hq/twinkle-flow-core)
(toggleable baseline effectiveness).

## Session config

From `plugin.yaml` `session`:

| Field | Value | Purpose |
|---|---|---|
| `skills` | `skills/system-model/SKILL.md` | The single knowledge skill — the system model the agent must internalize before configuring anything (see below). |
| `capabilities` | 14 tasks (listed below) | Process guides compiled into SKILL.md at session start, each activated by its `trigger`. |
| `flows` | `design-agent`, `flow-creation` | The two owned DAGs. |

### Hooks

- **SessionStart** — *"Introduce configurator capabilities"*: a static `echo`
  block orienting the agent that it can design/configure Orchestra
  meta-processes, and instructing it to invoke
  `/workers-configurator:system-model` first.
- **PreToolUse** — *"Block direct skill installation"*: reads the tool call
  Claude Code sends on stdin (`.tool_input.command`, with `jq`), and **exits
  2** when a command runs `skills add` (`npx skills add`, `npx -y skills
  add`, on any line), with the message to use `npx skills find` to *search*
  and then adapt content into workspace-repo format instead of installing.
  This guardrail is the enforcement behind the `find-external-skills` /
  `adapt-external-skill` workflow. It used to read `$CLAUDE_TOOL_INPUT`,
  which Claude Code never sets, so until 2026-10 it never fired. Every
  non-Bash tool passes (it has no `command`).

### The `system-model` skill

`skills/system-model/SKILL.md` is the configurator's foundational knowledge: it
defines every entity type (MCP entity, plugin, role, flow, task), the five MCP
`connection_type` values and their per-type schema, the **binding chain**
(MCP → plugin → role → worker), packaging/distribution (repos + `flows.toml`
manifests + MVS resolution), the directory structure, naming conventions, the
runner assembly process, and the `flows` CLI verbs (`lint`, `reliability`,
`preview-skill`). Nearly every task names `skills: [system-model]` on its first
substantive phase — it is the shared reference all the create/review/test tasks
lean on.

## The two flows

The plugin owns two end-to-end DAGs, both `on.workflow_dispatch: true`
(dispatch-only — a configurator process is always initiated by a human or another
agent, never by a real-world event), parameterized by a single `configurator`
worker param.

### `flow-creation` — the gated, human-in-the-loop pipeline

A four-stage agent pipeline, each stage separated by a **`type: human`** review
gate. The human gates are the autonomy ramp: a human inspects and approves the
artifact of each stage before the next agent stage begins.

```
capture-and-design ──▶ [review-brief] ──▶ draft-entities ──▶ [review-draft]
   (Brief)              human gate          (YAML files)        human gate
        │
        └──▶ validate-and-review ──▶ [review-validation] ──▶ test-and-iterate ──▶ [review-tests]
                (review report)         human gate              (test/benchmark)     human gate
```

| Agent job | Task | Produces | Following human gate |
|---|---|---|---|
| `capture-and-design` | `capture-and-design` | `brief` | **review-brief** — "Review the captured requirements and entity-design brief before drafting begins." |
| `draft` | `draft-entities` | `entities_created`, `validation_results` | **review-draft** — "Review the drafted YAML entities before semantic validation and review." |
| `validate-and-review` | `validate-and-review` | `review_report`, `critical_count`, `total_findings` | **review-validation** — "Review the validation and multi-lens review results before sandbox testing." |
| `test-and-iterate` | `test-and-iterate` | `test_results`, `benchmark_report`, `improvements_applied` | **review-tests** — "Review the sandbox test results and iterations before the configuration is considered production-ready." |

`test-and-iterate` declares `needs: [review-validation, capture-and-design]`;
the test stage receives the original Brief (its test cases) alongside the
reviewed configuration. Job timeouts: 60m / 60m / 45m / 90m.

### `design-agent` — the autonomous variant

The "think before you build" path: a single complex task that runs the whole
loop in one session without human gates, followed by a sandbox test.

```
design (design-agent) ──▶ test (test-agent)
```

`design-agent` (the task) internally walks analyze → inventory →
`find-external-skills` → write design doc → `create-entities` (loops, `flows
lint` after each) → `compose-role` → `test-agent` → a `type: parallel` **review**
fan-out (`review-validation`, `review-coherence`, `review-design`,
`review-coverage`) → a `type: decision` `quality-check` that loops back to
re-create on Critical findings (capped at 2 iterations) → `evaluate` (assess
against requirements, clean up the test worker). Its hard gates demand a written
design document before any YAML and require all role plugins to exist + validate
before the role is composed.

## Task catalog

14 of the 21 task YAMLs are passive **capabilities** (listed in
`session.capabilities`); the rest are review lenses and sub-tasks invoked by the
capability tasks. Grouped by purpose:

### Capture & design

- **`capture-and-design`** (complex) — capture requirements, inventory existing
  entities, collect human input (invokes the sibling `gather-input` task — see
  *Source notes*), design the entity map with a dependency graph and creation
  order, formulate ≥2 test cases, and compose a self-contained **Brief**.
  Hard-gated: must have human input before designing; the entity map must trace
  every entity to a requirement; test cases are part of requirements.
- **`design-agent`** (complex, task) — holistic single-session agent design:
  analyze → inventory → marketplace search → design doc → create all entities →
  compose role → test → parallel review → decision loop → evaluate. The task that
  backs the `design-agent` flow's first job.

### Drafting

- **`draft-entities`** (medium) — take an approved Brief and create all YAML in
  strict dependency order (MCP → skills → tasks → flows → plugin), running
  `flows lint` after each entity, then cross-check against the Brief's entity map
  and commit. Carries Task-YAML and Flow-YAML checklists as `sections`.

### Review lenses

Seven single-purpose `review-*` lens tasks (all `simple`, all output `findings`
scored Critical / Important / Suggestion), plus the complex `review-plugin`
aggregator. Two callers fan the lenses out as `type: parallel` phases —
`review-plugin` (all 7 lenses) and the `validate-and-review` flow stage (a
4-lens subset):

- **`review-plugin`** (complex) — comprehensive plugin review through **all 7**
  lenses, then consolidate/deduplicate/categorize with a decision loop.
- **`review-validation`** — schema compliance, broken references, structural
  integrity (runs `flows lint`).
- **`review-reliability`** — anti-rationalization quality, hard gates, review
  loops, gate specificity on medium+ tasks (runs `flows reliability`).
- **`review-coherence`** — components form a unified capability; no orphan
  entities; capability/flow don't duplicate each other.
- **`review-coverage`** — gaps: missing triggers, undeclared phase tool/skill
  refs, undeclared outputs, missing skill frontmatter, flows lacking triggers.
- **`review-design`** — architecture: active (flows) / passive (session)
  separation, single responsibility, correct hook lifecycle, oversized-plugin
  detection.
- **`review-duplication`** — duplicated processes, reinvented marketplace
  wheels, extraction/reuse opportunities across the repo.
- **`review-usability`** — descriptions, triggers, naming conventions, gate
  verifiability, anti-rationalization realism (developer experience).

The `validate-and-review` flow stage fans out a **4-lens** subset (coherence,
coverage, design, duplication) on top of automated reliability + preview-skill
inspection.

### Testing & iteration

- **`validate-and-review`** (complex) — semantic validation + multi-lens review:
  `flows reliability`, `flows preview-skill`, the 4-lens parallel fan-out,
  aggregate/categorize findings, then a `type: decision` fix-cycle that re-runs
  reliability until Critical/Important findings are resolved.
- **`test-and-iterate`** (medium) — sandbox testing with **baseline comparison**:
  run each test case with-config and baseline in parallel, draft
  **artifact-based** assertions, grade the produced entity files + their
  `flows lint --format=json` result, aggregate a benchmark, and iterate. Adapted
  from the skill-creator evaluation cycle.
- **`test-agent`** (medium) — single sandbox run via the `twinkle` CLI: create a
  test worker, assign a test task, run it, grade the produced **artifacts** (not
  the execution log), and clean up.

> **Honest grading boundary** (documented in both test tasks): `twinkle run
> list --task_id <id> --limit 1` returns only the run row — measured on a live
> stand, its fields are status, timings, tokens, cost, summary and machine,
> with **no** JSONL behavioral trace. So these tasks grade on what is
> repo-locally verifiable — terminal status, produced entity files + their
> structure, the `flows lint` result, and declared outputs — and explicitly do
> **not** assert "the agent followed phases in order." Behavioral grading is
> named as a deferred follow-up requiring a platform tool that exposes the
> execution trace.

### External-skill adaptation

- **`find-external-skills`** (complex) — search the skills.sh marketplace
  (`npx skills find`, multiple keyword strategies), read full SKILL.md content,
  score relevance/quality/source reputation, compare against existing config, and
  curate a prioritized list. Hard-gated to **never** `npx skills add`. Reviews
  its curation via parallel `review-coverage` + `review-usability`.
- **`adapt-external-skill`** (medium) — adapt one curated skill into
  workspace-repo format: a `type: decision` branches on algorithm-vs-knowledge —
  algorithms become a task YAML (validated through `create-task`), knowledge stays
  a SKILL.md with proper frontmatter and a pushy trigger. Also hard-gated against
  `npx skills add`.

### Entity creators

Five `create-*` tasks, one per entity type, all `invoked_by: design-agent` (and
some by `create-flow`/`create-plugin`):

- **`create-flow`** (complex) — author a flow DAG: understand the process, check
  for duplicates, design the job graph, choose the right trigger (automatic vs
  dispatch-only), create any missing tasks, write the flow YAML, validate, review
  (`review-validation` + `review-coverage`), and commit. Carries Parameter-Model
  and Flow-YAML-Reference `sections`.
- **`create-task`** (complex) — the most important entity for agent quality: map
  the **human** workflow, simulate ≥3 rationalization failures (hard-gated),
  design phases, write YAML, validate, check reliability, 4-lens review, preview,
  commit. Carries a Params-vs-Context `section`.
- **`create-mcp`** (simple) — declare an MCP entity: pick one of the five
  `connection_type` values via a decision tree (rejecting `internal` in a library
  repo), copy the matching skeleton from `system-model` verbatim, mark credential
  fields `secret: true`, validate, commit.
- **`create-plugin`** (medium) — assemble a plugin: identify needed
  MCP/skills/hooks, check existing inventory, create missing dependencies, write
  `plugin.yaml` (with an orienting SessionStart hook), validate, commit.
- **`create-role`** (medium) — compose existing plugins into a role preset:
  verify every referenced plugin exists, write the identity prompt, write the
  role YAML, validate, commit.

## How entities are authored & validated

The configurator has **no entity-management MCP server** — authoring is purely
local: it edits YAML in its own git repo (its `git_repo`) and validates with the
repo-local `flows` CLI (`flows lint`, `flows reliability <task>`,
`flows preview-skill <task>`) run via `Bash` from the repo root. Commits/pushes
are handled by the runner-side git guards.

The `twinkle` CLI is the platform-side half. This plugin does not wire it — the
runner mounts the binary into every worker container regardless of attached
plugins; what this plugin adds is the skill that teaches it. The sandbox-testing
tasks use it to spin up and grade throwaway test workers, and the system-model
skill teaches the reads and writes that reconcile the repo against what the
platform answers — `plugin list --workspace`, `role list --workspace`,
`worker capabilities`, `library sync` (all `scope: 'worker'`) and
`worker update` (`scope: 'admin'`). The `--workspace` flag is not optional on
the two list commands: measured on a live stand, `plugin list` answered 6
plugins with no flag and 11 with `--workspace sandbox`, and `role list` answered
`[]` with no flag and 2 with it. See `skills/system-model/SKILL.md` for the full
entity model and for the four shapes an answer from the CLI can take — the
fourth being a `command not found` naming `twinkle`.

## Source notes

- `capture-and-design` invokes `gather-input` as a `type: task` sub-task, but
  `gather-input` is **not** owned by `workers-configurator` — it lives in the
  sibling `project-management` plugin
  (`plugins/project-management/tasks/gather-input.yaml`). This is a valid
  cross-plugin reference within the same library repo, but worth knowing when
  reading `capture-and-design` in isolation.
- `review-duplication` references `shared/tasks/` ("Check against shared/tasks/")
  in its `analyze` phase; the repo has no `shared/` directory (the system-model
  documents that there is no `shared/` and entities are always plugin-owned). The
  reference reads as a holdover from the legacy `shared/` layout — its intent
  (check for duplicates against existing repo tasks) is still served by the
  surrounding `plugins/*/{tasks,flows}` inventory step.
