# skills

A collection of Claude Code skills for efficient, high-quality development workflows.

## Skills

| Skill | Description |
|-------|-------------|
| [tech-lead](./tech-lead/) | Principal-level tech lead: TDD authoring, task breakdown, and implementation review |
| [dev-expert](./dev-expert/) | Principal-level developer: implements tasks end-to-end with review loop, commit, and PR |
| [csvkit-feature-review](./csvkit-feature-review/) | Audits a csvkit tool against design guidelines and emits an HTML quality report |

## Installation

### Install a single skill

```bash
cp -r tech-lead ~/.claude/skills/tech-lead
```

### Install all skills

```bash
git clone https://github.com/nadavhdev/skills.git
cp -r skills/tech-lead ~/.claude/skills/
cp -r skills/dev-expert ~/.claude/skills/
cp -r skills/csvkit-feature-review ~/.claude/skills/
```

Then start a new Claude Code session — skills are loaded at session startup.

### Project-specific installation

To install a skill for a single project only:

```bash
cp -r tech-lead /path/to/your/project/.claude/skills/tech-lead
```

## Structure

Each skill follows the [Agent Skills](https://agentskills.io/specification) standard:

```
<skill-name>/
├── SKILL.md          # Skill definition (name, description, instructions)
├── evals/            # Eval test cases (evals.json + input files)
└── references/       # Reference docs loaded on demand by the skill
```

The `tech-lead-workspace/` directory contains iteration history from skill development runs (benchmarks, eval outputs, grading results).
