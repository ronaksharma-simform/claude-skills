---
name: sync-repo
description: Use this skill whenever the user has just pulled, merged, rebased, switched branches, or freshly cloned a git repo and wants to catch up their local environment — trigger on phrases like "sync repo", "just pulled", "pulled latest", "merged main", "after merge", "catch up my env", "what do I need to run after this pull", "just cloned this repo", "set up this repo", "get this project running", or when the user asks "do I need to install/migrate anything". Also invoke proactively right after you (Claude) perform a git clone/pull/merge/checkout on the user's behalf, or before running a project's basic setup/start commands for the first time. Handles both a normal post-pull diff AND a first-time-setup/fresh-clone case (no prior reflog history) — it does not just describe the scenario, it actually runs `git diff --name-only HEAD@{1} HEAD` (or, on a fresh clone, scans the full file tree instead of diffing) plus `git status --porcelain=v1` to inspect real files, cross-references them against dependency manifests (package.json/package-lock.json/yarn.lock/pnpm-lock.yaml, requirements.txt/pyproject.toml/Pipfile) and migration paths (prisma/schema.prisma, **/migrations/*.py), and detects unresolved merge conflicts (opening them in VS Code via `code <file>` right away). It then runs the needed install/migration commands directly — the tool's own permission prompt is the confirmation, so there is no second round-trip asking which to run.
---

# Sync Repo (post-pull assistant)

Personal workflow skill: after the user pulls/merges changes, scan what changed
and catch the local environment up — installs, migrations, and merge conflicts.

**Run what's needed directly.** Invoking a command already raises the normal
permission prompt, and that prompt *is* the user's chance to decline — so don't
stage a separate "here's what I'd run, which do you want?" round-trip on top of
it. Say in one line what you're running and why, then run it. The only things
worth stopping to ask about are genuine either/or decisions the file list can't
settle (see step 5) and the branch choice in step 0.

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
  proceed on the current one silently. This is the one question the skill always
  asks, because no file inspection can reveal intent.
- Once they've named it, act on that answer — `git checkout <branch>` (fetching
  first if it's remote-only) without asking a second time. Their choice was the
  confirmation, and the checkout's own permission prompt is the backstop.
- Then check whether the branch is behind its remote
  (`git fetch <remote> <branch>` + `git rev-list --left-right --count`). If it
  is, pull it — that's the sync the user asked for, so don't stop to ask whether
  they want it. Report how many commits came in.
- Before any checkout or pull, run `git status --porcelain=v1`: if the working
  tree is dirty, say which files are uncommitted and whether they collide with
  the incoming changes, since that's what decides if the pull is safe or needs a
  stash first.
- Only once the right branch is checked out and current do you move to step 1.

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
- If **step 0 did the pull**, you already know both ends: capture
  `git rev-parse HEAD` before pulling and diff that SHA against `HEAD` after.
  Prefer this over `HEAD@{1}` — it stays correct no matter how many reflog
  entries the checkout and pull added between them.
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

## 2. Detect dependency changes and install

Check the changed-files list for these manifests and map each to its command
(step 5 runs them):

**Node:**
- `package-lock.json` changed → `npm install`
- `yarn.lock` changed → `yarn install`
- `pnpm-lock.yaml` changed → `pnpm install`
- Only `package.json` changed (no lockfile in the list) → install with whichever
  lockfile exists in the repo root, and note the mismatch. If more than one
  lockfile exists, that's a step-5 question, not a guess.

**Python:**
- `requirements.txt` changed → `pip install -r requirements.txt`
  (also handle `-r requirements-dev.txt` etc. if such files changed too)
- `pyproject.toml` changed → `poetry install` if `poetry.lock` exists, else
  `pip install -e .`
- `Pipfile` / `Pipfile.lock` changed → `pipenv install`

If a monorepo has multiple manifests in different subdirectories, run per
directory rather than assuming repo root.

## 3. Detect likely-needed migrations

- Any path matching `prisma/schema.prisma` in the changed list → a migration is
  needed, but which one is a real decision: `npx prisma migrate dev` (local/dev
  DB, generates a migration) vs `npx prisma migrate deploy` (shared/staging,
  applies existing ones). Ask — don't guess.
- Any new/changed file under a path matching `**/migrations/*.py` (Django
  convention) → `python manage.py migrate`. List which app(s) got new migration
  files.

If neither pattern matches but a manifest changed in a way that smells
DB-related (e.g. an ORM model file changed with no matching migration file),
flag it as "possible missing migration" rather than silently skipping.

## 4. Detect merge conflicts

Run `git status --porcelain=v1` and look for `UU`, `AA`, `DD` (or similar
unmerged) markers, or grep tracked files for `<<<<<<<` conflict markers.

If any are found:
- List every conflicted file.
- Open them in VS Code immediately — run `code <file>` for each (the user
  confirmed VS Code as their editor). Don't ask first: opening a file changes
  nothing in the repo, and getting the conflicts in front of the user is the
  whole point of noticing them. If `code` isn't on PATH, say so and print the
  paths instead.
- Do not attempt to auto-resolve conflicts.
- Treat conflicts as blocking: hold off on installs and migrations until
  they're resolved. A half-merged tree makes both meaningless, and a conflicted
  lockfile or schema would have the install/migration acting on markers rather
  than real content.

## 5. Run what's needed, then summarize

No separate approval step — the permission prompt on each command is the gate.

- State in one line what you're running and why ("`package-lock.json` changed →
  running `npm install`"), then run it.
- One command at a time, showing output. Stop on the first failure and report it
  rather than pushing the remaining commands through a broken state.
- If a command is denied, that's an answer — note it and move on to the next
  category instead of re-asking or re-running it a different way.
- **Do** stop to ask when the file list genuinely can't settle a choice, because
  guessing wrong has a real cost:
  - `prisma migrate dev` (local DB, generates a new migration) vs
    `prisma migrate deploy` (shared/staging, applies existing ones) — the wrong
    one writes to the wrong database.
  - A `package.json` change with no lockfile change, in a repo with more than
    one lockfile — which package manager owns this tree.
  These are decisions, not permissions; the prompt can't disambiguate them.
- If nothing needs running in a category, say so in one line rather than
  omitting it silently.

Close with a short summary: what changed, what ran and its result, and what's
left for the user — conflicts to resolve, or a migration you deliberately
didn't guess at.

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
