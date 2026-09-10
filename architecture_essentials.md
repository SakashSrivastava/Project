# Architecture Essentials

A quick-reference checklist. See [architecture.md](architecture.md) for the full picture and [PRD.md](PRD.md) for why each choice was made.

## Environment variables

- `ANTHROPIC_API_KEY` — billed separately from any Claude plan; needed for every execute call
- `GROQ_API_KEY` — needed for the classify call

Both must be present on **each teammate's own machine** — the skill runs client-side inside their own Claude Code session, not against a shared server.

## Models

- Anthropic tiers, in escalation order: `claude-haiku-4-5-20251001` → `claude-sonnet-5` → `claude-opus-5`
- Groq classifier: a small, fast model such as an 8B Llama on Groq — pick one at build time, it is a config value, not an architectural decision

## Core packages

- `anthropic`, `groq` — model clients
- `langgraph` — orchestration, run as a local library, not a service
- `sqlite3` (standard library) or `sqlalchemy` — storage

No `fastapi`, no `uvicorn`, no `streamlit` — there is no hosted API and no separate frontend. The skill's own instructions (`SKILL.md`) are the interface.

## Orchestration graph nodes

1. `classify` — Groq call, skipped during bootstrap
2. `execute` — Anthropic call at the current tier
3. `validate` — run generated code against its tests (inline tests in standalone mode, the project's real test suite in real-repo mode) in a subprocess
4. `decide` — pass → done; fail → escalate one tier and loop, or fail the task at Opus

Which task mode applies (standalone vs. real-repo) is decided by the Claude Code session itself while reading the task — it isn't a graph node, and it isn't a Groq call.

## Storage tables

- `tasks`
- `classifier_predictions`
- `executions`
- `cost_ledger`

All stored in a SQLite file under `.claude/skills/cost-autopilot/data/`.

## Config that must exist before v1 runs

- Per-tier pricing table (input/output cost per token, for Haiku, Sonnet, Opus, and the Groq classifier model)
- Subprocess timeout and memory limit for validation runs
- Bootstrap-to-routed cutover rule (how many logged tasks before the classifier takes over the first guess)
- The `/cost-autopilot` invocation contract in `SKILL.md` — what inline input looks like, and what marks a task as real-repo instead of standalone

## v1 constraints, restated

- Anthropic only for the tiers doing the work; Groq only for classification
- One task type: coding, either a standalone spec or a reference to an existing feature in an open project
- Standalone tasks require tests — no tests, no run. Real-repo tasks validate against the project's own test suite instead
- No multi-turn, back-and-forth sessions — one task in, one result out, escalation aside
- Validation runs in a local subprocess, not a container
- No hosted service, no web frontend — the Claude Code skill is the whole product
