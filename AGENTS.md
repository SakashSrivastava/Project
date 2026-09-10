# AGENTS.md

Guidance for any coding agent working in this repository, regardless of tool.

## Project summary

A Claude Code skill, invoked with `/cost-autopilot`, not a hosted service. Given a coding task — either a standalone spec with tests, or a reference to something in an already-open project — it picks the cheapest Anthropic model tier likely to solve it, runs the task, validates the result by executing the code against its tests, and escalates to the next tier only when validation fails. The full reasoning is in `PRD.md`; the design is in `architecture.md`; a condensed checklist is in `architecture_essentials.md`.

## Before making changes

Read `PRD.md` and `architecture.md` first. Both documents record deliberate choices — for example, that a standalone task requires tests, that validation is deterministic rather than judged by another model, that escalation moves one tier at a time rather than jumping straight to the top tier, and that this ships as a Claude Code skill rather than a website with its own API. Don't reverse one of these choices without calling it out; assume it was decided on purpose.

## Structure

- `.claude/skills/cost-autopilot/SKILL.md` — the skill's instructions and the `/cost-autopilot` invocation contract
- `.claude/skills/cost-autopilot/scripts` — the classify/execute/validate/decide logic, Anthropic and Groq client wrappers, pricing table, bootstrap-to-routed cutover check
- `.claude/skills/cost-autopilot/data` — SQLite database file, bootstrap logs
- `.claude/skills/cost-autopilot/tests` — tests for the skill's own scripts
- `PRD.md`, `architecture.md`, `architecture_essentials.md`, `IMPLEMENTATION_PLAN.md` — planning docs, at the repo root

There is no `app/api` or `frontend` — this project doesn't run as a service.

## Two phases, one codebase

The system runs in a bootstrap phase (classifier bypassed, every task starts at the cheapest tier, staged escalation gathers labeled data) before it switches to a routed phase (classifier picks the starting tier, escalation remains as a fallback). Check `architecture.md` for the cutover condition before assuming the classifier is live.

## Two modes, one codebase

Standalone tasks (inline spec + tests) and real-repo tasks (reads/writes an actual open project, validates against its real test suite) both exist. Which one applies is decided by the invoking Claude Code session itself, as part of reading the task — there's no separate detection script to look for.

## Boundaries to respect

- Anthropic is the only provider for the model tiers doing the work; Groq is the only provider for classification. Don't add another provider's client without the user's say-so.
- A standalone task without tests should be rejected, not silently passed through with a weaker check.
- Sandbox execution stays a local subprocess with a timeout and memory limit — don't add container isolation as a "nice to have" without checking that the threat model has actually changed.
- Every added LLM call has a cost, and this project's job is to minimize total LLM cost. Treat a new model call the same way you'd treat a new dependency — justify it.
- Don't reintroduce a hosted API or a separate frontend. The skill's own instructions are the entire interface, by design.
