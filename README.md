# agent-skills

A collection of Claude Code skills for git workflows, database patterns, and beyond.

## Install

Install all skills into your local `.claude/skills/` directory:

```bash
npx skills add winens/agent-skills
```

Or install a specific skill:

```bash
npx skills add winens/agent-skills rebase-worktree-conflicts
```

## Skills

| Skill | Description |
|---|---|
| [rebase-worktree-conflicts](skills/rebase-worktree-conflicts/skill.md) | Rebase a worktree branch onto latest main and resolve merge conflicts |
| [sqlc-jsonb-joins](skills/sqlc-jsonb-joins/SKILL.md) | Compose relational JOIN results into Go model types using PostgreSQL `to_jsonb()` with sqlc |

## License

MIT
