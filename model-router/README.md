# model-router

A Claude skill that saves usage on large tasks. The model you are talking to acts as an orchestrator: it splits the task, hands each part to the cheapest model that can do it well (Haiku or Sonnet) through subagents, and then reviews and merges the results.

## Why

Running a whole task on your most capable model is wasteful when much of it is mechanical: extracting data, reformatting, batch conversions, routine code. This skill keeps the expensive model for the parts that need judgment and pushes the rest down to cheaper ones.

## How it works

1. **Decides whether to activate.** Only for large tasks that split into at least 2-3 substantial, self-contained parts. Small tasks, context-heavy tasks and ambiguous or high-impact ones are done directly.
2. **Splits and assigns.** Mechanical work goes to Haiku, most coding and drafting to Sonnet, and the orchestrator keeps design, judgment calls and the final merge.
3. **Briefs each subagent.** Every brief is self-contained: goal, inputs, constraints, output format and a "done" criterion.
4. **Keeps the orchestrator's context small.** Subagents save their full output to files and return only a short summary.
5. **Reviews proportionately.** Checks each part against its "done" criterion, escalates a weak part one tier (once), and reports in one line which model did what.

## Requirements

- A Claude environment with a subagent tool that accepts a model parameter (for example, Claude Code).
- Without subagents the skill cannot switch models; it does the task and suggests changing the model manually when that would have been enough.

## Installation

**Claude Code**: copy the `model-router` folder into your skills directory:

```
~/.claude/skills/model-router/SKILL.md
```

or into `.claude/skills/` inside a project to enable it for that project only.

**Claude apps**: upload the `model-router` folder as a custom skill.

## Usage

The skill activates on its own when a task is large enough. You can also steer it:

| Say | Effect |
| --- | --- |
| "split it" | Activate even if the task does not look large |
| "do it with haiku" / "do it with sonnet" | Send the whole task to that model |
| "no delegation" | The current model does everything |

The commands work in any language.

## Limitations

- **Savings are not guaranteed.** Each subagent starts with no context, so delegation has overhead. The skill pays off on large, mechanical, well-defined work, and can break even or cost more on tasks that depend heavily on context.
- **Activation is a judgment call.** Some environments are conservative about launching subagents, so the skill may not trigger by itself. "split it" is the reliable way to force it.
- **Model tiers change.** The Haiku < Sonnet < Opus ordering is written into the skill; update it if the lineup changes.
- By default subagents are told not to make commits, pushes or pull requests unless you ask for them.
