# Architecture

## Tech stack

| Layer | Choice | Role |
|---|---|---|
| Delivery | Claude Code skill (`SKILL.md` + Python scripts) | Invoked directly inside a coding session with `/cost-autopilot` — no separate service, no deployment |
| Routed models | Anthropic (Haiku, Sonnet, Opus) | The three tiers that actually do the coding work |
| Classifier | Groq (a small, fast open-weight model) | Guesses simple vs. complex before any Anthropic call runs |
| Orchestration | LangGraph, as a local library | The classify → execute → validate → escalate loop, run in-process by the skill's scripts |
| Storage | SQLite + JSON | SQLite for structured records; JSON columns for raw, shape-varying data such as model responses and test output. Lives inside the skill's own folder |

There is no API layer and no frontend. The skill's output — the answer plus its routing trace — is delivered straight into the Claude Code conversation that invoked it.

## Two operating phases

**Bootstrap phase.** The classifier is bypassed. Every task starts at Haiku. If validation fails, the task escalates one tier at a time up to Opus. Each run logs which tier finally passed, and that log is the classifier's future training data. This phase runs until there is enough logged data to trust a prediction.

**Routed phase.** The classifier makes the first guess (Haiku or Sonnet) from the task spec, via a Groq call. Cascade escalation stays on as the safety net: if the guess was wrong, the task still climbs one tier at a time to the tier that works. Opus is never a direct classifier prediction — only an escalation target.

## Two task modes

**Standalone mode.** The task has no existing project behind it — a spec plus test cases, both supplied inline in the message that invokes the skill. Validation runs those tests directly.

**Real-repo mode.** The task references or extends something in a project already open in the session. The skill reads the relevant files for context, writes generated code into the project, and validates using the project's own test suite instead of inline tests.

Which mode applies is decided by the Claude Code session itself, as part of reading the invoked task — not a separate classification step, and not the Groq call. If the task points at existing files or an existing feature, it's real-repo mode; otherwise standalone.

## Request flow

1. The user invokes `/cost-autopilot` inside a Claude Code session, with either an inline spec + tests (standalone) or a reference to an existing feature (real-repo).
2. The session decides standalone vs. real-repo mode, then the skill's scripts run the orchestration loop:
   - **Classify** — in the routed phase, calls Groq, predicts simple/complex, and sets the starting tier. In the bootstrap phase, this step is skipped and the starting tier is always Haiku.
   - **Execute** — calls the chosen Anthropic tier with the task (plus file context, in real-repo mode), gets back code.
   - **Validate** — runs the code against its tests — inline tests in standalone mode, the project's own test suite in real-repo mode — in a local subprocess, under a timeout and resource limit, and returns pass/fail with detail.
   - **Decide** — pass: mark the task done. Fail, with a tier remaining above the current one: escalate to the next tier (Haiku → Sonnet → Opus) and loop back to Execute. Fail at Opus: mark the task failed.
3. Every outcome — tier tried, pass/fail, cost, latency — is written to the local SQLite file.
4. The skill reports back into the conversation: the result, plus which tier ran, whether it escalated, and what it cost.

## Data model

**tasks**
- `id`, `spec` (text), `tests` (JSON, empty in real-repo mode), `mode` (standalone / real-repo), `status` (pending / done / failed), `created_at`

**classifier_predictions**
- `id`, `task_id`, `predicted_tier` (simple / complex), `raw_response` (JSON), `latency_ms`, `cost_usd`, `phase` (bootstrap / routed)

**executions**
- `id`, `task_id`, `tier` (haiku / sonnet / opus), `code_output` (text), `passed` (bool), `validation_detail` (JSON), `cost_usd`, `latency_ms`, `escalated_from_execution_id` (nullable), `created_at`

**cost_ledger**
- `id`, `task_id`, `total_cost_usd`, `baseline_cost_usd` (cost if the task had gone straight to Opus), `savings_usd`

## Folder map

| Path | Holds |
|---|---|
| `.claude/skills/cost-autopilot/SKILL.md` | The skill's instructions and invocation contract — the `/cost-autopilot` command, how it reads inline vs. real-repo tasks, how it reports results |
| `.claude/skills/cost-autopilot/scripts` | The classify, execute, validate, and cascade logic; Anthropic and Groq client wrappers; the pricing table; the bootstrap-to-routed cutover check |
| `.claude/skills/cost-autopilot/data` | The SQLite database file, local to this skill |
| `.claude/skills/cost-autopilot/tests` | Tests for the skill's own scripts, distinct from the tests a task is validated against |
| `PRD.md`, `architecture.md`, `architecture_essentials.md`, `IMPLEMENTATION_PLAN.md` | Planning docs, at the repo root |
