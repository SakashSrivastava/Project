# LLM Cost Autopilot — Product Requirements

## Problem

Most tools send every task to one fixed model, usually the strongest one available. Nobody wants to guess wrong on a task that turns out hard, so the safe choice becomes the only choice. That means a one-line function costs the same as a gnarly bug fix, because both pay the price of the hardest task the system might ever see. Spend scales with the ceiling, not the actual need.

## Solution

A routing layer sits between the caller and the model — not as a separate website, but as a Claude Code skill, called directly from inside a coding session. It reads a coding task, picks the cheapest model tier likely to succeed, runs the task, checks whether the result actually works, and only moves up to a pricier model if the cheap one fails. Across many tasks, the goal is the smallest total spend that still gets every task done.

## Who this is for

The author's own coding-task workflow, plus two teammates working from the same Claude Code setup. A personal/internal tool, not a product for other people, at least for now.

## Scope for v1

One task type: coding, called with an explicit command inside Claude Code. A task arrives one of two ways:

- **Standalone** — a self-contained spec, such as a function or class description, with test cases attached, for a task with no existing project behind it.
- **Real-repo** — a request that references or extends something already in an open project. The skill reads the relevant files, writes generated code into the project, and validates against the project's own test suite instead of inline tests.

Which mode applies is a judgment call the Claude Code session itself makes while reading the task — no separate detection step. No provider besides Anthropic supplies the model tiers that do the work; Groq handles the classification step that picks a starting tier.

## What "done right" means

A task passes when its code passes its tests — the ones submitted with it in standalone mode, or the project's own test suite in real-repo mode. That is the only judge: run the code, pass or fail. No model grades another model's work.

## What "cheapest" means

Not the lowest cost on any single task, but the lowest total spend across every task the system runs, even if a handful of individual tasks end up costing more than a person guessing by hand would have spent.

## Non-goals for v1

- No routing across providers other than Anthropic for the model tiers doing the work
- No multi-turn, back-and-forth sessions — one task in, one result out, escalation aside
- No container isolation for running code
- No support for a standalone task that ships without tests
- No separate hosted service or web frontend — the skill is the entire product

## The cold-start problem

The classifier starts with no data on which tasks need which tier. Early tasks skip the classifier and run through the full staged cascade instead: cheapest tier first, one step up on failure. Every run through that cascade becomes a labeled example — a real record of which tier a given task actually needed. Once enough examples exist, the classifier starts making the first guess instead of the system always starting at the bottom.

Classification stays a separate Groq call rather than something the Claude Code session judges for itself, on purpose: a fixed external model gives a stable, reproducible signal to calibrate the cutover against, instead of one that drifts depending on which Claude model happens to be running the session that day.

## Success metric

Total dollars spent across a batch of tasks, measured against a baseline of running every task on the top model tier.
