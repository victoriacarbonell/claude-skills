---
name: model-router
description: "Use only when the user sends a large task that can be split into parts, or says 'split it'. The current model breaks it down, delegates each part to the cheapest model that can do it well, and reviews the result."
---

# Model router

Goal: use less of the usage budget without losing quality. The conversation's model acts as orchestrator: it splits the task, delegates each part to the cheapest model able to do it well, then reviews and merges the results.

Having this skill installed is an explicit request from the user to use subagents whenever the activation conditions are met.

## 1. Activation: large tasks ONLY

Activate only if both conditions hold:
- **It is large**: it can be split into at least 2-3 parts, each involving substantial work (several files or documents to process, many searches, or a lot of text or code to produce).
- **It can be split**: each part can be described in a self-contained way, without relying on what was said earlier in the conversation.

Do not activate (just do the task yourself, without mentioning this skill) if:
- It is a question, a message, a small fix, or something that takes only a few steps.
- It depends heavily on the conversation so far, and explaining that to a subagent would cost more than doing the work.
- It is ambiguous, high-impact or hard to undo, or it is a medical or legal matter.
- There is no model cheaper than the current one to delegate to.

When in doubt, do not activate.

## 2. Split and assign

Models from cheapest to most expensive: Haiku < Sonnet < Opus < higher tiers. Delegate only to models cheaper than the conversation's model; delegating at the same tier saves nothing.

Give each part to the cheapest model that will do it well:

- **Haiku**: mechanical, well-defined work: extracting data, reformatting, classifying, renaming, batch conversion, summarizing simple documents, simple translations.
- **Sonnet**: most coding, text drafts, medium-complexity research, analysis with clear criteria, tests.
- **Current model (keep it yourself)**: anything ambiguous, the overall design, judgment calls, and the final merge.

Rules:
- One task can be spread across several models. Do not force everything onto the same one.
- If you are torn between two models for a part, pick the higher one: redoing work costs more than getting it right.
- Do not over-split: each part must be big enough for delegation to be worth the brief.
- Run independent parts in parallel; run dependent parts afterwards.

## 3. Brief for each subagent

The subagent cannot see this conversation. Every brief includes:
- The goal of that part and how it fits into the whole.
- Concrete inputs (paths, data, links) and only the context it needs.
- Constraints and the exact output format (so merging is easy).
- A "done" criterion.
- How to deliver: "Save the full result to <file path> and return only a 5-10 line summary: what you did, where it is, and any doubts or problems. Do not paste the full content in your reply."
- "Do not make commits, pushes or pull requests." (unless the user explicitly asked for them in this task).

Call the Agent tool with `subagent_type: "general-purpose"` and `model: "haiku"` or `"sonnet"`. Subagents never delegate further: the orchestrator always does the splitting.

## 4. Final review (proportionate)

- Start from the summaries. Open the result files only to check the "done" criterion and to look closely at anything doubtful or critical; do not reread everything line by line.
- If a part comes back weak, move it up one tier, once (Haiku → Sonnet → do it yourself).
- Merge the parts and deliver the result.
- Tell the user in one line how the work was split (which model did what).

## 5. If there is no Agent tool

Without subagents, the model cannot be changed from inside the session. Do the task and, if it was clearly Haiku- or Sonnet-level work, suggest in one line switching models in the model picker (or with /model in Claude Code).

## 6. Manual commands

These count only when the user gives them as an instruction about the task, not when a model is mentioned in passing. They work in any language (the equivalent phrase in the user's language counts).

- "split it": activate even if the task does not look large.
- "do it with haiku" / "do it with sonnet": send everything to that model, without evaluating.
- "no delegation": do it yourself.
