---
name: commit-review
description: Groups all changed files into logical commits with recommended messages and ordering, then walks through each group one at a time — automatically staging each group after plan approval, prompting for commit confirmation or feedback, and auto-staging the next group after each commit. Use when you want to review and commit changes interactively.
allowed-tools: Bash(git status), Bash(git diff *), Bash(git add *), Bash(git commit *), Bash(git restore --staged *), Bash(git log *)
---

# Commit Review

Walk through all changed files, group them into a recommended commit plan, then commit them one group at a time.

## Phase 0 — Pre-flight checks

### 0a. Check current branch

Run `git branch --show-current`. If the result is `main`, `develop`, or `staging`:
- Ask the user if they'd like to check out a new branch before committing
- Propose a branch name derived from the staged/unstaged changes (e.g. `feat/add-speed-slider` or `fix/auth-redirect`)
- If yes, run `git checkout -b <proposed-name>` (or the name they provide)
- If no, continue on `main`/`develop`/`staging`

After the branch is confirmed, extract the branch number: look for a leading or embedded integer in the branch name (e.g. `feat/123-add-auth` → `123`, `42-fix-bug` → `42`). Store this as `BRANCH_NUM`. If no number is found, `BRANCH_NUM` is unset.

### 0b. Lint all files

Run the `/lint` skill to lint all changed files. If linting fails:
- Show the errors to the user
- Ask if they want to fix the errors before continuing
- If yes, fix them, re-run lint to confirm clean, then proceed
- If no, proceed anyway but note the outstanding lint errors

## Phase 1 — Discover changes

Run the following in parallel to collect all changed files:
- `git diff --name-status HEAD` — unstaged changes
- `git diff --cached --name-status` — already-staged changes
- `git status --short` — untracked files

Merge the results into one flat list with no duplicates.

## Phase 2 — Build the commit plan

Analyse the full list and group files into logical commit groups. Each group must:
- Represent a single, clear purpose expressible as one subject line
- Be as small as possible while staying coherent
- Use a conventional commit type: `feat`, `fix`, `docs`, `style`, `refactor`, `test`, `chore`, `perf`, `ci`, `build`, `revert`

Message format: `type(optional-scope): description` — if `BRANCH_NUM` is set, append ` [#BRANCH_NUM]` to every commit message subject line (e.g. `feat(auth): add JWT middleware [#123]`).

Order the groups so that foundational or dependency changes come before the things that depend on them (e.g. migrations before features, config before code that uses it).

Present the full plan:

```
Proposed commit plan
────────────────────
1. chore(config): add eslint configuration [#123]
   · .eslintrc.json
   · .eslintignore

2. feat(auth): add JWT middleware [#123]
   · src/middleware/auth.ts
   · src/middleware/auth.test.ts

3. feat(user): add user profile endpoint [#123]
   · src/routes/user.ts
   · src/models/user.ts

────────────────────────────────────────
Order rationale: config first, then auth middleware that other routes depend on, then user routes that use that middleware.
```

Ask: **"Does this grouping and order look right? Suggest any changes, or say 'approve' to begin."**

Wait for the user's response. Incorporate any requested changes and re-present until approved. Do not proceed to Phase 3 until the user explicitly approves.

## Phase 3 — Commit loop

Once the plan is approved, immediately stage the first group and enter the loop.

### 3a. Stage the group and show it for review

1. Run `git restore --staged .` to clear any previously staged files
2. Run `git add <files in this group>`
3. Show the staged diff with `git diff --cached --name-status`

Then display:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Group 1 of 3 — staged and ready
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Files staged:
  · src/middleware/auth.ts
  · src/middleware/auth.test.ts

Commit message:
  feat(auth): add JWT middleware [#123]

Reply 'commit' to commit, 'edit' to change the message, 'skip' to discard and move on, or give feedback.
```

### 3b. Handle the response

**'commit'** (or clear equivalent):
1. Run the commit using a heredoc:
   ```
   git commit -m "$(cat <<'EOF'
   feat(auth): add JWT middleware [#123]
   EOF
   )"
   ```
2. Show `git log --oneline -1`
3. Immediately stage the next group (go to 3a for the next group)

**'edit'**: Ask for the new message. Update it, re-show the group prompt, and wait for another reply.

**'skip'**: Run `git restore --staged .`. Note the files as skipped. Immediately stage the next group.

**Any question or discussion**: Answer it, then re-show the current group prompt without advancing.

### 3c. End of groups

After the last group is committed or skipped, go to Phase 4.

## Phase 4 — Summary

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Session complete
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Committed:
  abc1234 chore(config): add eslint configuration
  def5678 feat(auth): add JWT middleware

Skipped / uncommitted:
  · src/routes/user.ts
  · src/models/user.ts
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

## Rules

- Never commit without the user saying 'commit' (or a clear equivalent) for that specific group.
- Never use `--no-verify`.
- Never `git push`.
- Never amend a previous commit unless the user explicitly requests it.
- Always clear the staging area with `git restore --staged .` before staging a new group.
- If you make any changes to a file during the review loop (fixes, improvements, lint corrections), immediately re-stage the affected files with `git add <files>` so the staged diff stays current before the user commits.
- Never add a `Co-Authored-By` trailer or any other attribution to yourself in commit messages. Commits are authored solely by the user.
