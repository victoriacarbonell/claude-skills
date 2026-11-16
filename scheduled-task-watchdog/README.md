# scheduled-task-watchdog

A Claude skill that sets up a watchdog for your scheduled tasks. If a task fails because you had run out of usage when it fired, the watchdog runs it again once your usage is back.

## Why

Scheduled tasks fire at a fixed time. If that time falls when you have reached your usage limit, the run fails and nothing retries it. For a daily task (a briefing, a digest, a daily lesson) that means a missed day you may not even notice.

There is no event for "usage has been reset", so the watchdog checks a few times a day instead.

## How it works

1. **Asks for the essentials**: your time zone, when to check (default: 4 times a day, about 4 hours apart) and which model to use (default: the cheapest, e.g. Haiku).
2. **Creates one extra scheduled task**, the watchdog. On each run it lists your recurring tasks and fires again only those whose last run:
   - failed,
   - because the usage limit was reached,
   - within the last 24 hours.
3. **Stays quiet unless something happened.** It messages you only when it re-runs a task, or when a task failed for a different reason (which it reports instead of retrying in a loop).

## Requirements

- A Claude environment with scheduled tasks and tools to list, create, update and fire them (for example, `list_triggers`, `create_trigger`, `update_trigger` and `fire_trigger`).
- The watchdog's own runs use a little of your usage. Running it on the cheapest model keeps that small.

## Installation

**Claude Code**: copy the `scheduled-task-watchdog` folder into your skills directory:

```
~/.claude/skills/scheduled-task-watchdog/SKILL.md
```

or into `.claude/skills/` inside a project to enable it for that project only.

**Claude apps**: upload the `scheduled-task-watchdog` folder as a custom skill.

## Usage

| Say | Effect |
| --- | --- |
| "set up the scheduled-task watchdog" | Creates the watchdog (or updates it if one exists) |
| "change the watchdog times" / "use another model for the watchdog" | Updates the existing watchdog |
| "remove the watchdog" | Deletes or disables it, after confirming |

Any language works.

## Limitations

- **A check can also hit the limit.** If the watchdog runs while you are still out of usage, it fails too; the next check picks things up.
- **Recovery runs are late.** A task re-run by the watchdog runs at the next check after your usage resets, not at its usual time.
- **Only recurring tasks with a recorded last run.** One-shot tasks and tasks that have never run are ignored.
- **Failure reasons depend on the platform.** The skill looks for a usage-limit failure reason (e.g. `USAGE_LIMIT_REACHED`); if your platform names it differently, adjust the prompt.
