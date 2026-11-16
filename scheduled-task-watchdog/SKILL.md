---
name: scheduled-task-watchdog
description: Use when the user wants their scheduled tasks re-run after they fail because the usage limit was reached, or asks to set up, change or remove the scheduled-task watchdog.
---

# Scheduled-task watchdog

Problem: a scheduled task that fires while the user is out of usage fails, and nothing runs it again. The user only notices days later, if at all.

Solution: one extra, lightweight scheduled task (the watchdog) that runs a few times a day, finds recurring tasks whose last run failed because of the usage limit, and fires them again.

## 1. Check what is available

You need the scheduling tools: one that lists scheduled tasks with their last run (e.g. `list_triggers`), one that creates them (e.g. `create_trigger`), one that updates them (e.g. `update_trigger`) and one that fires them on demand (e.g. `fire_trigger`). Load them with ToolSearch if they are deferred. If they are not available, tell the user in one line that this environment cannot schedule tasks and stop.

List the user's scheduled tasks first. If a watchdog already exists, update it instead of creating a second one.

## 2. Ask only what you need

Ask in a single message, with sensible defaults the user can accept as is:
- **Time zone**: use it if you already know it.
- **Check times**: default to 4 checks a day spread about 4 hours apart, starting 2-3 hours after the user's latest scheduled task (usage limits usually reset within a few hours). Use an off-the-hour minute (e.g. :47).
- **Model**: default to the cheapest available model (e.g. Haiku). The job is mechanical, so a cheaper model does it just as well and spends less of the budget.

## 3. Create the watchdog

Create a recurring scheduled task named "Scheduled-task watchdog" with:
- the cron expression in the user's time zone (`CRON_TZ=<zone> <min> <hours> * * *`),
- the chosen model,
- push notifications off (the watchdog sends its own message only when something happened),
- this prompt, translated into the user's language:

```
You are the watchdog for the user's scheduled tasks. Your only job is to re-run tasks that failed because the usage limit was reached. Use as few tokens as possible and do nothing else.

1. Get the current date and time (a time tool if you have one; otherwise the date in your context).
2. List scheduled tasks with enabled=true and recurring=true.
3. Skip this task ("Scheduled-task watchdog").
4. Fire a task again ONLY if all three hold:
   - its last run status is FAILED,
   - its failure reason is the usage limit (e.g. USAGE_LIMIT_REACHED),
   - that run fired within the last 24 hours.
   Pass this text with it: "Recovery run: the scheduled run of [last run date] failed because the usage limit was reached. Run it now as usual."
   In every other case (succeeded, running, pending, failed for another reason, failed more than 24 hours ago, or no last run) do nothing.
   Fire each task at most once per watchdog run.
5. If you fired any task, send the user one short message: "🔁 Re-ran: [names]". If any task failed for ANOTHER reason in the last 24 hours, add a line: "⚠️ [name] failed: [reason]; not re-run". If none of this happened, finish without sending anything.
```

Why the rules matter:
- **Only usage-limit failures**: a task that fails for another reason (a broken prompt, a missing connector) would fail again on every check. The watchdog reports it instead of retrying it in a loop.
- **Only the last 24 hours**: an old failure has usually been replaced by a newer run, and re-running a stale one makes no sense for daily tasks.
- **Silent when nothing happened**: the watchdog runs several times a day; it should not create noise.

## 4. Confirm

Tell the user in a few lines: when the watchdog runs, which model it uses, and how its runs handle approvals (say whether they were set to approve automatically). Mention that if a check itself lands while the user is still out of usage, it fails too, and the next check picks it up.

## Changing or removing it

- "Change the watchdog times / model": update the existing task.
- "Remove the watchdog": delete or disable the task, after confirming with the user.
