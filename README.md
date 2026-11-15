# claude-skills

A collection of skills for Claude that I build and use in my own work. Each skill lives in its own folder with a `SKILL.md` (the instructions Claude follows) and a `README.md` (what it does and how to use it).

## Skills

| Skill | What it does |
| --- | --- |
| [model-router](model-router/) | Saves usage on large tasks: the current model splits the work, delegates each part to the cheapest model that can do it well, and reviews the result. |

## Installation

Each skill is self-contained, so you only need the folder of the one you want.

**Claude Code**: copy the skill's folder into your skills directory:

```
~/.claude/skills/<skill-name>/
```

or into `.claude/skills/` inside a project to enable it for that project only.

**Claude apps**: upload the skill's folder as a custom skill.

See each skill's README for requirements and usage.

## Feedback

Issues and suggestions are welcome. If a skill does not behave as described, please open an issue with the task you tried and what happened.

## License

[MIT](LICENSE)
