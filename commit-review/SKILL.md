---
name: commit-review
description: This skill should be used when the user wants to review and commit code changes. It groups changed files into focused logical commits and walks the user through each group interactively, allowing them to approve, skip, or request changes before committing.
---

# Commit Review

Interactive review-then-commit flow that groups changes logically and walks the user through each one.

## Grouping Rules

Group changed files into the smallest focused commits that each represent a single logical unit of work. Check memory for any past feedback on grouping preferences before proposing groups.

General principles:
- One commit per concern: don't mix a new feature with a refactor or a config change
- Files that are tightly coupled belong together (e.g., a component + its store change + its integration into a parent)
- Plans, docs, and config changes get their own commits
- New files that support the same feature can be grouped with related modifications
- Prefer more smaller commits over fewer large ones
- Order commits so earlier ones don't depend on later ones

For each group, determine:
- **Type**: `feat`, `fix`, `refactor`, `docs`, `test`, `chore`, `perf`, `style`, `ci`, `build`
- **Scope**: area of code (e.g., `ui`, `ws`, `store`, `api`, `canvas`)
- **Files**: exact list of files in this commit
- **Message**: why, not just what

Exclude files that should NOT be committed: `.env`, credentials, large binaries, `node_modules`, `__pycache__`

## Learning from Feedback

When the user corrects a grouping decision, save their preference as a feedback memory so it applies in future sessions.

## Flow

### 0. Check current branch

Run `git branch --show-current` to get the current branch name.

If the current branch is `main`:
- Ask the user if they'd like to check out a new branch before committing
- Propose a branch name based on the staged/unstaged changes (e.g. `feat/add-speed-slider` or `fix/auth-redirect`)
- If the user says yes, run `git checkout -b <proposed-name>` (or the name they provide)
- If the user says no, continue on `main`

### 1. Lint all files

Run the `/lint` skill to lint all changed files. If linting fails:
- Show the errors to the user
- Ask if they want to fix the errors before continuing
- If yes, fix them, re-run lint to confirm clean, then proceed
- If no, proceed anyway but note the outstanding lint errors

### 2. Gather context

Run these commands in parallel:

- `git status` (never use `-uall`)
- `git diff` and `git diff --staged` to see all changes
- `git log --oneline -10` to match recent commit style

### 3. Group changes

Apply the grouping rules above to produce an ordered list of commit groups.

### 4. Present the overall plan

Show a numbered overview of all proposed commits:

```
Proposed commits (in order):

1. feat(store): add playbackSpeed state to useUIStore
   Files: app/src/stores/useUIStore.ts

2. feat(ui): add SpeedSlider component and wire into header
   Files: app/src/components/controls/SpeedSlider.tsx, app/src/components/layout/Header.tsx

3. ...
```

Ask if the overall grouping and order looks right before starting the walkthrough. The user can reorder, merge, or split groups at this point.

### 5. Walk through each group

**CRITICAL: Process ONE group at a time. After each step, STOP and WAIT for the user's response. Never batch multiple commits in a single response.**

For each commit group, one at a time:

**a) Stage the files** — Run `git add <specific files>` for this group only. Then show `git diff --cached --stat` so the user can see what's staged.

**b) Show the proposed commit message**

**c) Tell the user the files are staged for review** — The user can inspect the staged diff in their IDE/terminal. Then STOP and WAIT for their response:
- **Yes** — Commit the staged files as proposed
- **Skip** — Unstage these files (`git reset HEAD <files>`), move to the next group
- **Change** — The user describes what they want changed. Make the edits, re-stage, show updated `git diff --cached --stat`. Re-prompt.
- **Question** — The user asks about the changes. Answer, then re-prompt.
- **Discuss** — Open-ended discussion. When done, re-prompt.

**IMPORTANT: After committing a group, STOP and WAIT for the user before moving to the next group. Do NOT stage the next group's files in the same response as the commit. Each group gets its own full cycle: stage → show message → wait → commit → wait → (next group).**

When the user says "yes":
1. Commit the already-staged files using a HEREDOC:

```bash
git commit -m "$(cat <<'EOF'
<type>(<scope>): <short description>

<optional body explaining why, not what>
EOF
)"
```

Commit message rules:
- Subject line: imperative mood, no period, max 72 characters
- Body is optional — include only when the "why" isn't obvious from the subject
- No Co-Authored-By or attribution footers — commits are from the user's account

3. Confirm success with `git status` before moving to the next group

### 6. Summary

After walking through all groups, show:
- Commits created (`git log --oneline` for the new commits)
- Any skipped or remaining uncommitted files
- Do NOT push unless the user explicitly asks
