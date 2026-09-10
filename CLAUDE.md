# CLAUDE.md

Guidance for Claude Code when working in this repository.

## What this project is

A Claude Code skill, not a hosted service. Called with `/cost-autopilot` from inside a coding session, it picks the cheapest Anthropic model tier likely to solve a given task, runs the task, checks the result by running the code's own tests, and escalates to a pricier tier only on failure. Read [PRD.md](PRD.md) for why, and [architecture.md](architecture.md) for how.

## Ground truth

Treat `PRD.md`, `architecture.md`, and `architecture_essentials.md` as the source of truth for scope and design. If a change would contradict one of them, flag the conflict rather than quietly picking a side.

## Working here

- Python throughout. Match the folder map in `architecture.md` — everything lives under `.claude/skills/cost-autopilot/`: instructions in `SKILL.md`, logic in `scripts/`, the SQLite file in `data/`. Don't blur those boundaries for convenience.
- There is no API and no frontend to build. The skill's own instructions are the interface — don't reintroduce a FastAPI or Streamlit layer without the user asking for it back.
- The project is currently in the bootstrap phase by design (see `architecture.md`): the classifier is meant to be bypassed until enough logged task outcomes exist. Don't wire the classifier into the routing decision as a default without that data existing first.
- Two task modes exist — standalone (inline spec + tests) and real-repo (reads and writes an actual open project, validates against its real test suite). Deciding which mode applies is the invoking session's own judgment call, made while reading the task — it isn't a script, and it isn't the Groq call.
- `scripts/validate.py` runs model-generated code on the local machine. Any change there must keep a timeout and a resource limit in place — that subprocess is running code an LLM just wrote, not code a person reviewed.
- This project's entire point is spending less on model calls. When touching orchestration or model-client code, notice if a change adds an LLM call that wasn't there before, and say so — an extra call anywhere in this codebase works against the project's own goal.
- Tests a task is validated against (inline, or a project's own suite in real-repo mode) are not the same thing as this project's own test suite in `.claude/skills/cost-autopilot/tests/`. Don't conflate the two.

## Running it

- Invoke it inside a Claude Code session with `/cost-autopilot`.
- To test a script directly without going through the skill: `python .claude/skills/cost-autopilot/scripts/cascade.py` (once it exists).
- The SQLite file lives under `.claude/skills/cost-autopilot/data/`.

## Secrets

`ANTHROPIC_API_KEY` and `GROQ_API_KEY` are required, set locally in each person's own environment — there's no shared server to hold them centrally. Never hardcode either; never log a raw key or a full request payload that might contain one.
