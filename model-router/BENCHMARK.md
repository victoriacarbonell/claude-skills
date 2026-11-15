# model-router benchmark

An A/B test of the `model-router` skill: the same task run twice, once without the skill and once with it, comparing cost and quality.

**Status: protocol written, not run yet.** The task, the measurements and the quality checklist below were fixed before any run, so the results cannot shape the criteria.

## Question

On a large task that can be split into parts, does `model-router` reduce cost without lowering the quality of the result?

## Setup

| | Run A (control) | Run B (skill) |
| --- | --- | --- |
| Skill installed | No | Yes |
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
# Run A first, before the skill is installed
git clone https://github.com/victoriacarbonell/nutri-app run-a
cd run-a && claude
#   /model opus  ->  paste the prompt  ->  when it finishes: /usage
cd ..

# Install the skill, then Run B
mkdir -p ~/.claude/skills && cp -r claude-skills/model-router ~/.claude/skills/
git clone https://github.com/victoriacarbonell/nutri-app run-b
cd run-b && claude
#   /model opus  ->  paste the same prompt  ->  when it finishes: /usage
cd ..
```

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

_Not run yet._

| | Run A (control) | Run B (skill) |
| --- | --- | --- |
| Date and Claude Code version | | |
| Skill activated by itself | n/a | |
| Split (model: parts) | n/a | |
| Opus tokens (in / out / cache read / cache write) | | |
| Sonnet tokens | | |
| Haiku tokens | | |
| Estimated cost (USD) | | |
| Wall-clock time | | |
| 1. File coverage (/35) | | |
| 2. Endpoint coverage | | |
| 3. File summary errors (5 sampled) | | |
| 4. Endpoint errors (5 sampled) | | |
| 5. Spec mismatches (real / false) | | |
| 6. Architecture document (1-5) | | |
| 7. Rule compliance | | |

## Limitations

- **One run per condition.** Model output varies between runs, so a single A/B pair shows what happened once, not what happens on average. A difference of a few percent means nothing at this sample size.
- **One task, one repository.** The task was chosen because it suits the skill. Tasks that depend on conversation context are expected to benefit less or not at all.
- **The control is not "no subagents".** Claude Code may launch subagents by itself in Run A. The comparison is "with the skill" against "what Claude Code does by default".
- **Cost is an estimate** computed locally at list price. On a subscription plan what is saved is usage allowance, not money.
- **The reviewer wrote the skill.** The blind shuffle reduces bias in the quality scores but does not remove it.
