# project-scout-subagents — Claude Code skill

Parallel-subagent edition of project-scout. Before you start a project, it fans
out subagents to discover the latest libraries, trending repos, and what's new
in the Claude/AI ecosystem, then synthesizes a single recommendation.

## Install

```bash
mkdir -p ~/.claude/skills/project-scout-subagents && curl -sf \
  https://raw.githubusercontent.com/mitchelfletcher11/project-scout-subagents/main/SKILL.md \
  -o ~/.claude/skills/project-scout-subagents/SKILL.md
```

Then invoke it by asking Claude to scout a project, or with `/project-scout-subagents`.

The skill runs without any credentials, but a few **optional** API tokens make
its searches faster and more precise — see **[SETUP.md](SETUP.md)**.
