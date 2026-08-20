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
  conflicts after a `git pull`/`merge`/`clone`, and runs the right
  install/migrate commands directly — the tool's own permission prompt is the
  confirmation. Only asks first when a decision genuinely can't be inferred
  (which branch to sync, `migrate dev` vs `migrate deploy`).
