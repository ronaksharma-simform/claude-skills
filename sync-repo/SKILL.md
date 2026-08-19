---
name: sync-repo
description: Use this skill whenever the user has just pulled, merged, rebased, switched branches, or freshly cloned a git repo and wants to catch up their local environment — trigger on phrases like "sync repo", "just pulled", "pulled latest", "merged main", "after merge", "catch up my env", "what do I need to run after this pull", "just cloned this repo", "set up this repo", "get this project running", or when the user asks "do I need to install/migrate anything". Also invoke proactively right after you (Claude) perform a git clone/pull/merge/checkout on the user's behalf, or before running a project's basic setup/start commands for the first time. Handles both a normal post-pull diff AND a first-time-setup/fresh-clone case (no prior reflog history) — it does not just describe the scenario, it actually runs `git diff --name-only HEAD@{1} HEAD` (or, on a fresh clone, scans the full file tree instead of diffing) plus `git status --porcelain=v1` to inspect real files, cross-references them against dependency manifests (package.json/package-lock.json/yarn.lock/pnpm-lock.yaml, requirements.txt/pyproject.toml/Pipfile) and migration paths (prisma/schema.prisma, **/migrations/*.py), and detects unresolved merge conflicts (opening them in VS Code via `code <file>`). It then proposes the exact install/migration commands and executes only the ones the user confirms.
---

# Sync Repo (post-pull assistant)

Personal workflow skill: after the user pulls/merges changes, scan what changed
and tell them what they need to do to catch up their local environment —
installs, migrations, and merge conflicts. **Never run installs or migrations
without explicit confirmation** — always suggest first, then ask.

## 0. Confirm which branch to work on

Never assume a branch — not the currently checked-out one, not `develop`, not
`main`. Different repos default to different branches, and silently working
off whatever happens to be checked out (e.g. a leftover `develop` from a
previous clone/checkout) is exactly the kind of mistake this step exists to
prevent.

- Run `git branch --show-current` and `git branch -a` (or
  `git remote show origin` for the remote's default) to see what's available.
- Ask the user explicitly which branch they want to sync/work on — list the
  current branch and the other local/remote branches as options, don't just
  proceed on the current one silently.
- If the user picks a branch other than the one currently checked out, confirm
  before running `git checkout <branch>` (and `git fetch`/`git pull` first if
  it's a remote-only branch) — this mutates the working tree.
- Only after the right branch is checked out do you move to step 1.

## 1. Find what changed

First rule out a **first-time setup** (fresh clone, or an existing repo where
dependencies were never installed). Signs of this, any one of which is enough:
- `git reflog` has only one entry (`clone: from ...`), or
  `git diff --name-only HEAD@{1} HEAD` errors with
  `fatal: log for 'HEAD' only has 1 entries` / `ambiguous argument 'HEAD@{1}'`.
- A dependency manifest exists at the repo root (or in a subdirectory) but its
  install output directory doesn't: `package.json` with no `node_modules/`,
  `pyproject.toml`/`Pipfile` with no active venv, etc.

Do **not** fall back to `git diff --name-only main...HEAD` in this case — if
`HEAD` already points at `main`/`master` that diff is empty against itself and
will falsely report "nothing changed" even though nothing has ever been
installed. Instead switch to first-time-setup mode: treat *every* manifest,
prisma schema, and migrations directory that exists in the working tree as
"relevant" (skip the diff entirely) and walk straight into steps 2–4 against
the full file listing (`git ls-files` or a directory scan), noting to the user
that this looks like a first-time setup, not a post-pull sync.

Otherwise (normal case — this isn't a fresh clone), determine the commit range
to diff, in order of preference:

- If mid-merge-conflict (see step 4), diffing doesn't matter yet — jump to step 4 first.
- Otherwise use the pull's reflog entry: `git diff --name-only HEAD@{1} HEAD`
  (this is "before this pull" → "after this pull").
- If `HEAD@{1}` doesn't look like a pull (e.g. user ran this skill much later)
  but a reflog with real history exists, fall back to asking the user for a
  comparison ref, or use `git diff --name-only main...HEAD` / `git status` as
  a sanity check, and say which range you used.

Run:
```
git diff --name-only HEAD@{1} HEAD
```
Keep this file list — steps 2 and 3 filter it, don't re-diff per stack.

## 2. Detect dependency changes and suggest installs

Check the changed-files list for these manifests and map to a suggested command:

**Node:**
- `package-lock.json` changed → suggest `npm install`
- `yarn.lock` changed → suggest `yarn install`
- `pnpm-lock.yaml` changed → suggest `pnpm install`
- Only `package.json` changed (no lockfile in the list) → still suggest an
  install with whichever lockfile exists in the repo root; note the mismatch.

**Python:**
- `requirements.txt` changed → suggest `pip install -r requirements.txt`
  (mention `-r requirements-dev.txt` etc. if such files also changed)
- `pyproject.toml` changed → check if `poetry.lock` exists → suggest
  `poetry install`; else suggest `pip install -e .`
- `Pipfile` / `Pipfile.lock` changed → suggest `pipenv install`

If a monorepo has multiple manifests in different subdirectories, group
suggestions by directory instead of assuming repo root.

## 3. Detect likely-needed migrations

- Any path matching `prisma/schema.prisma` in the changed list → suggest
  `npx prisma migrate dev` (local/dev DB) or `npx prisma migrate deploy`
  (shared/staging DB) — ask the user which applies, don't guess.
- Any new/changed file under a path matching `**/migrations/*.py` (Django
  convention) → suggest `python manage.py migrate`. List which app(s) got new
  migration files.

If neither pattern matches but a manifest changed in a way that smells
DB-related (e.g. an ORM model file changed with no matching migration file),
flag it as "possible missing migration" rather than silently skipping.

## 4. Detect merge conflicts

Run `git status --porcelain=v1` and look for `UU`, `AA`, `DD` (or similar
unmerged) markers, or grep tracked files for `<<<<<<<` conflict markers.

If any are found:
- List every conflicted file.
- For each, run `code <file>` to open it in VS Code (the user confirmed VS
  Code as their editor). If `code` isn't on PATH, say so and give the file
  paths instead.
- Do not attempt to auto-resolve conflicts.

## 5. Summarize and confirm before acting

Present one consolidated summary: what changed, what command(s) you'd run and
why, and what migrations/conflicts need attention. Then ask the user which
suggested commands (if any) to actually run — run only what they confirm, one
at a time, showing output. If nothing changed in a category, say so briefly
rather than omitting it silently.

## 6. If follow-up code changes are needed, write them to minimize future conflicts

Once the environment is caught up, this skill is often immediately followed by
adding code on top of what was just synced (a new feature, a fix, filling in
something the pull/merge exposed as missing). Before writing anything:

- Read enough of the current state first — the relevant files, the branch's
  existing patterns, naming, and structure — so the change fits what's
  already there instead of guessing.
- Prefer additive changes over restructuring: new functions/files/modules
  rather than reshaping shared ones, appending to existing lists/exports/config
  blocks rather than reordering them, and touching only the lines that need to
  change.
- Match existing conventions exactly (formatting, import style, naming) so the
  diff stays minimal and reviewable — gratuitous reformatting of untouched
  code is what turns a small change into a conflict-prone one.
- If a shared/central file must be touched, keep the edit as small and
  localized as possible, and say so in the summary so the user knows which
  files carry the higher conflict risk when they merge this branch back.
