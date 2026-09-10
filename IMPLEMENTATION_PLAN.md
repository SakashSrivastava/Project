# Implementation Plan

Three people, one codebase. This plan splits the work into a short shared foundation, three tracks that can run at the same time without blocking each other, then an integration pass. Track names are roles, not people — assign them however fits.

The product is a Claude Code skill, not a hosted service: no API to deploy, no frontend to host. Everything lives under `.claude/skills/cost-autopilot/` and runs locally, inside whoever's Claude Code session invokes it.

## Sequencing at a glance

| Phase | What | Who | Rough length |
|---|---|---|---|
| 0 | Foundations — shared contracts everyone codes against | All three, together | 1–2 days |
| 1 | Three parallel build tracks | One person per track | 4–6 days |
| 2 | Wire the tracks together | All three | 2 days |
| 3 | Run the bootstrap phase, collect real data | All three, passive | Until the cutover threshold is hit |
| 4 | Turn the classifier on, watch it, polish | All three | Ongoing |

The order matters more than the day count. Don't start Phase 1 before Phase 0 is settled — every track in Phase 1 codes against the schemas and table definitions Phase 0 produces, and redoing those mid-flight costs far more than a few extra days up front.

## Phase 0 — Foundations

Everyone works on this together; nobody starts their own track until it's done.

1. **Shared schemas** (`.claude/skills/cost-autopilot/scripts/schemas.py`) — data models for `TaskSubmission` (spec + tests, or a real-repo reference), `Execution`, `ClassifierPrediction`, `CostLedgerEntry`. These are the contract every track below writes against, so get agreement here before anyone writes track code.
2. **SQLite table definitions** (`scripts/storage.py`) — the four tables from `architecture.md`: `tasks`, `classifier_predictions`, `executions`, `cost_ledger`. Write the `CREATE TABLE` statements and a small init script that creates `data/cost_autopilot.db` on first run.
3. **Config and secrets loading** — where `ANTHROPIC_API_KEY` and `GROQ_API_KEY` come from (each teammate's own shell environment, never committed), plus a `.env.example` with placeholder names so everyone sets theirs up the same way.
4. **Pricing table format** — a config file (JSON or Python dict) with cost-per-input-token and cost-per-output-token for Haiku, Sonnet, Opus, and the chosen Groq model. Doesn't need real numbers finalized yet, just the shape.
5. **The `SKILL.md` invocation contract** — agree on what `/cost-autopilot` expects: how a standalone task's spec + tests are written inline, and what signals a real-repo task (referencing existing files/features) instead. This is the one new piece of Phase 0 that didn't exist in a web-service version of this project — get it settled early since Track C's `SKILL.md` and Track A's `classify`/`execute` logic both depend on the same shape.
6. **A shared sample-task set** — 5–10 small coding tasks (spec + tests), spanning obviously-easy to obviously-hard, all standalone. Every track needs something to run against during development, and using the same set means bugs found by one person are reproducible by the others.

Output of this phase: schemas, table definitions, config loading, the invocation contract, and sample tasks all merged to `main` before Phase 1 starts.

## Phase 1 — Three tracks

### Track A — Model clients and orchestration

The routing brain: the part that decides which model runs and when to escalate.

- Anthropic client wrapper (`scripts/execute.py`) — one function per tier that takes a task (plus file context, in real-repo mode), returns generated code plus token counts and computed cost.
- Groq client wrapper (`scripts/classify.py`) — calls the classifier model, returns a simple/complex prediction, raw response, latency, and cost.
- LangGraph graph (`scripts/cascade.py`) — four nodes: `classify`, `execute`, `validate`, `decide`.
  - `classify` must support a bootstrap-bypass mode (always predict nothing, starting tier is always Haiku) and a routed mode (call Groq, predict a starting tier).
  - `decide` implements one-tier-at-a-time escalation: Haiku → Sonnet → Opus, stop and mark the task failed if Opus fails.
- Unit tests for the graph's control flow using mocked model and validator calls — no real API spend needed to verify the escalation logic is correct. This is the track where a wrong test is expensive to discover late, since every other track depends on this behavior being right.

**Depends on:** Phase 0 schemas. **Blocks:** the `execute`/`decide` calls Track B's validation result feeds back into.

### Track B — Sandbox and storage

The part that runs the generated code and remembers what happened.

- Sandbox runner (`scripts/validate.py`) — takes generated code and a set of tests, runs them in a subprocess with a timeout and memory limit, returns pass/fail plus stdout/stderr detail. Needs two modes: run against inline tests (standalone), and run against a project's existing test suite (real-repo). Handle syntax errors and crashes as a plain fail, not an exception that kills the task.
- File I/O for real-repo mode — reading the relevant project files for context before `execute`, and writing generated code back into the project after validation passes.
- Storage repositories (`scripts/storage.py`) — insert/read functions for all four tables, built against Phase 0's table definitions.

**Depends on:** Phase 0 table definitions and schemas. Can build the sandbox and storage layers independently of Track A — stub the code-to-validate with a fixed sample so this track's tests don't need a real model call, then wire it to Track A's real output once both sides exist.

### Track C — Skill authoring, bootstrap tooling, and test harness

The part a human actually invokes, plus the machinery for the cold-start problem described in the PRD.

- `SKILL.md` — the skill's instructions: the `/cost-autopilot` command, how it tells standalone from real-repo tasks, how it calls into Track A/B's scripts, and how it reports the result back into the conversation (the answer, plus which tier ran, whether it escalated, and what it cost — see Q7 in the design notes).
- Bootstrap-to-routed cutover script (`scripts/cutover.py`) — reads the logged `executions`/`classifier_predictions` data, checks it against a cutover rule (for example, "at least N tasks logged with a clear tier split"), and reports whether the system is ready to flip from bootstrap to routed mode.
- Expanded sample-task library — grow Phase 0's 5–10 tasks into 15–20, covering a real spread of difficulty, for use as the initial bootstrap batch and as a standing regression check later.
- Skill test suite (`.claude/skills/cost-autopilot/tests`) — end-to-end tests that submit a sample task through the whole pipeline and check it lands correctly in storage. These can run against Track A's mocked graph early, then against the real thing once Phase 2 wires everything together.

**Depends on:** Phase 0's sample tasks, schemas, and invocation contract. Can draft `SKILL.md` and the cutover script against fake/mocked script output before Track A/B's real scripts exist.

## Phase 2 — Integration

All three together:

1. Point `SKILL.md`'s instructions at Track A's real graph and Track B's real sandbox/storage (remove the stubs).
2. Run the full sample-task library from Track C through the real pipeline in bootstrap mode, invoking `/cost-autopilot` for real inside a Claude Code session.
3. Test both modes explicitly — a standalone task and a real-repo task against a small sample project — since they exercise different code paths in `validate.py`.
4. Fix the seams: schema mismatches, sandbox timeout edge cases, cost numbers that don't add up.

Definition of done for this phase: every sample task runs end to end through the actual `/cost-autopilot` invocation, and the numbers reported back match what the SQLite tables actually hold.

## Phase 3 — Bootstrap run

Passive, ongoing. Each of the three uses `/cost-autopilot` for real coding tasks day to day. Every task runs the full staged cascade and logs which tier actually worked. Watch the cutover script from Track C; once it says there's enough data, move to Phase 4.

## Phase 4 — Routed phase and polish

1. Flip the system to routed mode: the classifier makes the first guess instead of always starting at Haiku.
2. Watch the escalation rate closely for the first batch of routed tasks — a high rate means the classifier's guesses are often wrong, and it's worth checking why before trusting it further.
3. Revisit the non-goals list in the PRD to confirm nothing crept into scope during the build.

## Cross-cutting notes

- **Git workflow** — one branch per track during Phase 1, merged to `main` only after the other two have looked at it. Since all three tracks touch shared schemas and the invocation contract from Phase 0, a change to either after Phase 1 starts needs a heads-up to the other two before merging.
- **Secrets** — neither API key ever gets committed. Each teammate sets `ANTHROPIC_API_KEY` and `GROQ_API_KEY` in their own shell environment; `.env.example` holds the placeholder names only.
- **No shared server** — because the skill runs client-side, "it works on my machine" is the actual deployment target for every teammate, not a stepping stone to one. Each of the three needs Python plus the `anthropic` and `groq` packages installed locally before Phase 1 work is usable.
- **Cost awareness during development** — Track A's unit tests and Track C's early `SKILL.md` drafting should run against mocked model calls, not real Anthropic/Groq calls, so building the system doesn't itself run up a bill before it's finished.
