# claude-skills

Personal Claude Code skills.

## Installing a skill

Copy the skill's folder into your `~/.claude/skills/` directory, e.g.:

```
cp -r sync-repo ~/.claude/skills/sync-repo
```

It will show up automatically in Claude Code's available-skills list on your
next session — no extra install step needed.

## Skills

- **[sync-repo](sync-repo/SKILL.md)** — post-pull/clone environment sync
  assistant. Detects dependency changes, needed migrations, and merge
  conflicts after a `git pull`/`merge`/`clone`, and suggests (never silently
  runs) the right install/migrate commands.
