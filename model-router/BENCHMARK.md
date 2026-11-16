# model-router benchmark

An A/B test of the `model-router` skill: the same task run twice, once without the skill and once with it, comparing cost and quality.

**Status: run on 15 and 16 November 2026.** The task, the measurements and the quality checklist were fixed before any run. What was run departs from the plan in several ways; the [Results](#results) section says how.

**Short version:** the skill never activated by itself. When forced with `split it`, it gave the cheapest and fastest run ($2.66 against $3.15 to $4.24 for Opus working alone) with complete coverage, but it missed two of four bugs that only show up when files are read together. One run per condition, so this is what happened once, not an average.

## Question

On a large task that can be split into parts, does `model-router` reduce cost without lowering the quality of the result?

## Setup

| | Run A (control) | Run B (skill) |
| --- | --- | --- |
| Skill installed | No | Yes, at project level |
| Prompt | Identical | Identical |
| Main model | Opus | Opus |
| Repository | Fresh clone of [nutri-app](https://github.com/victoriacarbonell/nutri-app) | Fresh clone of nutri-app |
| Session | New, no prior context | New, no prior context |

Run B uses the same prompt with no extra hint, so it also tests whether the skill activates by itself. If it does not, that is recorded as a result and a third run (B2) repeats the task with `split it` added at the end of the prompt.

## Task prompt

Paste exactly this in both runs:

```text
Document this repository. Create a docs/ folder containing:

1. docs/files/ - one Markdown file per source file in these folders:
   artifacts/api-server/src/routes, lib/db/src/schema,
   artifacts/calorie-tracker/src/pages and
   artifacts/calorie-tracker/src/components (skip components/ui).
   Each one: purpose, what it exports, what it depends on, and anything
   surprising in the code. 10 lines at most.

2. docs/api-reference.md - every HTTP endpoint the server exposes: method,
   path, parameters, request body, response and error cases. Derive it from
   the route code, compare it with lib/api-spec/openapi.yaml, and list every
   mismatch between the code and the spec in a final section.

3. docs/architecture.md - how the pieces fit together, the data flow for
   logging a meal from a photo, three design decisions with their
   trade-offs, and three concrete risks or improvements.

Do not modify any file outside docs/. Do not commit anything.
```

Why this task: it mixes mechanical work (35 file summaries), medium work (the API reference) and judgment (the architecture document), which is the split the skill is designed for.

## How to run it

```bash
git clone https://github.com/victoriacarbonell/claude-skills
git clone https://github.com/victoriacarbonell/nutri-app run-a
git clone https://github.com/victoriacarbonell/nutri-app run-b

# Install the skill for Run B only, at project level
mkdir -p run-b/.claude/skills
cp -r claude-skills/model-router run-b/.claude/skills/

# Run A, in one terminal
cd run-a && claude
#   /model opus  ->  paste the prompt  ->  when it finishes: /usage

# Run B, in another terminal
cd run-b && claude
#   /model opus  ->  paste the same prompt  ->  when it finishes: /usage
```

The skill is copied into `run-b/.claude/skills/` instead of the user-level skills directory, so Run A cannot see it and the two runs can go in parallel. The skill must not be installed at user level while the test runs. Send no other message in either session until the task finishes, since any extra turn changes the usage.

After each run, copy the Session block that `/usage` prints (total cost, durations and "Usage by model") into the results below.

If `/usage` does not show a per-model breakdown on your plan, run the task non-interactively instead; the JSON output includes the total cost and a per-model breakdown _(command not run yet)_:

```bash
claude -p "$(cat prompt.txt)" --model opus --permission-mode acceptEdits \
  --output-format json > ../result-a.json
```

## Measurements

**Cost**

- Tokens per model: input, output, cache read, cache write.
- Estimated cost in USD at list price, as Claude Code reports it. Tokens from different models are not comparable with each other, so cost is the common unit.
- Wall-clock time.
- Run B only: whether the skill activated by itself, and how it split the work (which model did what).

**Quality**

Blind review: shuffle the two `docs/` folders so the reviewer does not know which run produced which.

```bash
if [ $((RANDOM % 2)) -eq 0 ]; then a=1; b=2; else a=2; b=1; fi
mkdir -p blind && cp -r run-a/docs blind/$a && cp -r run-b/docs blind/$b
echo "a=$a b=$b" > blind/.key   # open only after scoring both
```

Checklist, scored for each folder:

| # | Check | How |
| --- | --- | --- |
| 1 | File coverage | Files in `docs/files/` out of 35 expected (10 routes, 8 schema, 10 pages, 7 components) |
| 2 | Endpoint coverage | Endpoints documented out of those defined in the route code (about 40 handlers by a text search; count the exact number once, by hand) |
| 3 | File summary accuracy | Pick 5 summaries at random; count factual errors against the code |
| 4 | Endpoint accuracy | Pick 5 endpoints at random; count errors in method, path, body or response |
| 5 | Spec mismatches | Real mismatches found, and false ones reported (the spec has about 34 operations by a text search, so real ones exist) |
| 6 | Architecture document | 1 to 5: correct, specific to this repository, useful to a new developer |
| 7 | Rule compliance | Any file touched outside `docs/`, or any commit: yes or no |

The same 5 summaries and 5 endpoints are checked in both folders.

## Results

### What was actually run

Five runs completed, all with the same prompt, on Opus 5.5, in auto permission mode. Claude Code 2.1.325 was recorded on 15 November and not re-checked on the 16th.

| Run | Date | Skill | Extra line in prompt | What happened |
| --- | --- | --- | --- | --- |
| A | 15 Nov | Not installed | None | Opus did everything itself |
| B1 | 16 Nov | Installed, broken header | None | Skill not used; Opus did everything itself |
| F1 | 16 Nov | Installed, broken header | `split it` | Skill not used; 4 subagents, all on Opus |
| B2 | 16 Nov | Installed, fixed | None | Skill listed but not used; Opus did everything itself |
| F2 | 16 Nov | Installed, fixed | `split it` | Skill loaded; 3 subagents on Sonnet, Opus wrote the architecture document and reviewed |

**The broken header.** As first published, the `description` line in `SKILL.md` contained an unquoted colon, which is invalid YAML. Claude Code listed the skill by its title only, with no description, so runs B1 and F1 did not test the skill: they tested a skill Claude could not tell when to use. This was found after F1 by asking the session what it saw, and fixed in commit `8abeb7a`. B1 and F1 are kept because they turned out to be useful comparisons.

**Discarded runs.** Two other attempts stopped partway when the account's usage limit ran out. They were not resumed, because continuing adds turns and changes the usage.

### Cost and time

Figures are the Session block of `/usage` at the end of each run.

| | A | B1 | B2 | F1 | F2 |
| --- | --- | --- | --- | --- | --- |
| Estimated cost (USD) | 4.24 | 3.15 | 3.17 | 4.81 | **2.66** |
| Wall-clock time | 16m 17s | 9m 35s | 10m 26s | 8m 58s | **7m 36s** |
| API time | 9m 27s | 5m 53s | 7m 9s | 12m 13s | 10m 27s |
| Opus output tokens | 61.6k | 40.5k | 49.5k | 83.3k | 10.7k |
| Opus cache read | 6.5M | 4.5M | 3.2M | 3.3M | 1.2M |
| Opus cache write | 212.8k | 178.5k | 191.1k | 473.0k | 52.2k |
| Sonnet output tokens | 0 | 0 | 0 | 0 | 76.3k |
| Sonnet cache read / write | 0 | 0 | 0 | 0 | 2.1M / 323.8k |
| Cost on Opus / Sonnet (USD) | 4.24 / 0 | 3.15 / 0 | 3.17 / 0 | 4.81 / 0 | 0.87 / 1.79 |
| Subagents | None | None | None | 4 on Opus | 3 on Sonnet |

Haiku appears in every run for well under one cent (background work, not documentation).

How F2 compares:

- Against A, the planned control: 37% cheaper.
- Against the mean of the three runs where Opus worked alone (A, B1, B2: $3.52): 24% cheaper.
- Against the cheapest of those three ($3.15): 16% cheaper.
- Against F1, the same kind of split with every subagent on Opus: 45% cheaper.

The three runs where Opus worked alone differ by 26% from cheapest to most expensive with nothing changed between them. A 16% gap to the cheapest one is inside that spread. The F1 to F2 comparison is the cleaner one: same shape of work, different models.

### Activation

- **The skill did not activate by itself**, with the header broken (B1) or fixed (B2). Asked afterwards, the B2 session quoted the description correctly, agreed the task fitted it, and said it had gone straight to the work without considering the skill.
- **`split it` with the header broken (F1)** was taken literally: the session split the work across four general-purpose subagents and passed no model, so all four ran on Opus. That was the most expensive run.
- **`split it` with the header fixed (F2)** loaded the skill and followed it: Sonnet for the per-file docs and the API reference, Opus for the architecture document and the review, and a closing line saying which model did what.

### Quality

Four of the five sets were reviewed: A, B2, F1 and F2. B1 was left out as a second Opus-alone run close to B2.

The review did not follow the plan above. The author had already seen which session produced which output, so a blind review by her was no longer possible. Instead the four `docs/` folders were copied under shuffled neutral names and scored by a fresh model session that was told nothing about how they were produced, only the original task and the checks. Its full evidence is in [BENCHMARK-REVIEW.md](BENCHMARK-REVIEW.md).

Ground truth established by the reviewer from the code: 40 endpoints, 34 operations in the spec, and 22 distinct real mismatches across the four sets.

| Check | A | B2 | F1 | F2 |
| --- | --- | --- | --- | --- |
| Files covered (of 35) | 35 | 35 | 35 | 35 |
| File docs over 10 lines | 0 | 0 | 0 | 0 |
| Endpoints documented (of 40) | 40 | 40 | 40 | 40 |
| Errors in 5 sampled file docs | 0 | 1 | 2 | 1 |
| Errors in 5 sampled endpoints | 0 | 0 | 0 | 0 |
| Spec mismatches: claimed | 21 | 18 | 28 | 25 |
| Spec mismatches: verified real | 14 | 13 | 27 | 19 |
| Spec mismatches: unsupported or false | 7 | 5 | 1 | 6 |
| Real mismatches missed (of 22) | 7 | 7 | 0 | 2 |
| Cross-file bugs found (of 4) | 4 | 4 | 2 | 2 |
| Architecture document (1-5) | 5 | 5 | 4 | 4 |
| Factual errors in architecture document | 0 | 0 | 2 | 1 |

Reading it:

- **Coverage does not separate the sets.** All four are complete and within the line limit.
- **The split runs found more spec mismatches.** F1 missed none of the 22 and F2 missed two; the two Opus-alone runs missed seven each.
- **The split runs missed cross-file bugs.** Both F1 and F2 missed that the Home page deletes a log entry with a field name the generated client does not expect, and both described the two halves of the recipe bug in separate docs without connecting them. Both Opus-alone runs found all four. Since F1 ran entirely on Opus, this comes from splitting the work, not from using a cheaper model.
- **The architecture document scored 4 in both split runs** against 5, each with a factual error.

So F2 is not "the same quality for less". It is cheaper and faster, better at the spec comparison, and worse at findings that need several files in view at once.

### Departures from the protocol

- The planned comparison was two runs. Five were needed to separate "skill present" from "skill used" and "split" from "split and routed".
- The skill under test changed once, to fix the broken header. Nothing else in it changed.
- The quality review was done by a model session, not by the author, and on four sets, not two.
- Three checks were added after the runs: the line limit, the real mismatches missed out of the union, and the four cross-file bugs. The cross-file check was prompted by a remark from the B2 session that splitting might lose such findings. It was not fixed in advance.
- Rule compliance (nothing changed outside `docs/`, nothing committed) rests on each session's own `git status` report and was not checked independently.

## Limitations

- **One run per condition.** Three runs with Opus working alone already differ by 26% in cost. One routed run cannot show that its saving is stable.
- **One task, one repository.** The task was chosen because it suits the skill. Tasks that depend on conversation context are expected to benefit less or not at all.
- **The skill was only ever used when forced.** Nothing here shows it activating by itself.
- **The control is not "no subagents".** Claude Code may launch subagents by itself. In practice it did not in A, B1 or B2.
- **Cost is an estimate** computed locally at list price. On a subscription plan what is saved is usage allowance, not money.
- **The reviewer is a model**, it ran once, and its accuracy checks cover a sample of 5 file docs and 5 endpoints. It applied a strict rule for what counts as a real mismatch; with a looser one A, B2 and F2 would each gain 5 or 6 and F1 would gain 1.
- **The person who set up the review wrote the skill.** The reviewer did not know which set was which, but the choice of checks was not blind.
