---
name: plan-phase
description: Takes the next phase of a plan and produces a deep, step-by-step execution brief with explicit unknowns, deliverables, and done criteria. Use between ce:plan and ce:work to bridge high-level planning and implementation.
argument-hint: "[path to plan file] [optional: phase name or number]"
---

# Plan Phase

Drill into one phase of a plan and produce a focused execution brief: ordered steps, open unknowns, concrete outputs, and explicit done criteria. The result is a tightly-scoped document you can hand directly to `ce:work`.

## Input

<plan_input> #$ARGUMENTS </plan_input>

**If no plan path is provided:**
Use the native file-search/glob tool (e.g., Glob in Claude Code) to check `docs/plans/` for recent plan files. Present the most recent options and ask which plan to use.

**Do not proceed until a valid plan file path is confirmed.**

---

## Phase 1 — Read the Plan

Read the plan file in full. Extract:

- The list of phases, implementation units, or major work items
- Their ordering and stated dependencies
- Any `Deferred to Implementation`, `Unknowns`, or `Open Questions` sections
- The overall goal and acceptance criteria

---

## Phase 2 — Identify the Target Phase

**If a phase was specified in the arguments**, confirm it exists in the plan and use it.

**If no phase was specified**, identify the next phase that has not yet been started or completed:

- Look for status indicators in the plan (`[ ]` checkboxes, `status: pending`, etc.)
- If all phases look untouched, default to Phase 1
- If it is ambiguous which phase is next, use the platform's blocking question tool (e.g., `AskUserQuestion` in Claude Code, `request_user_input` in Codex, `ask_user` in Gemini) to ask — or, if no question tool is available, present a numbered list and wait for the user's reply

State clearly: "Deepening **[Phase Name]** from `[plan file path]`."

---

## Phase 3 — Analyse the Phase

Before writing the brief, think through the phase carefully:

- What does the plan say about this phase? Quote the relevant section.
- What work in this phase depends on work from a prior phase?
- What in this phase does a later phase depend on?
- What does the codebase already have that can be reused or extended?
- Where is the plan intentionally vague or deferred?

Use the native file-search/glob and content-search/grep tools to look at relevant existing code before writing the brief. Do not rely solely on the plan's description.

---

## Phase 4 — Write the Phase Brief

Produce the brief using the structure below. Be specific and concrete — this document replaces the need to re-read the parent plan during implementation.

````markdown
---
plan: [path to parent plan file]
phase: [Phase name / number]
status: pending
date: YYYY-MM-DD
---

# Phase Brief: [Phase Name]

## Goal

One paragraph. What does this phase accomplish, and why does it come at this point in the sequence?

## Context from the Plan

> [Direct quote or close paraphrase of the relevant section from the parent plan]

---

## Branch Strategy

Every phase works on at least one dedicated branch. Define the branch structure here before implementation begins.

**Primary branch:** `[type]/[plan-slug]-phase-[N]`
Examples: `feat/user-auth-phase-1`, `fix/checkout-race-phase-2`

**When to use additional branches:**
Consider splitting into multiple branches when this phase contains work that:
- Can be reviewed independently (e.g. a migration separate from the feature that uses it)
- Involves a risky or experimental change that should be isolated
- Crosses a major layer boundary (e.g. backend API work and frontend work)

List each branch and its scope:

| Branch | Scope | Merges into |
|--------|-------|-------------|
| `feat/[slug]-phase-[N]` | [Primary work — e.g. service layer + tests] | `main` / `develop` |
| `feat/[slug]-phase-[N]-migration` | [e.g. database migration only] | `feat/[slug]-phase-[N]` |

If the phase is small and cohesive, a single branch is fine — say so explicitly: _Single branch — phase scope is too small to justify splitting._

**Commit strategy:** Use `commit-review` at each logical stopping point during implementation (not just at the end) to group related changes into clean, reviewable commits before merging.

---

## Step-by-Step Approach

Ordered, concrete steps. Each step should be small enough to complete in one sitting and independently verifiable.

### Step 1 — [Action]

What to do. Which files to touch or create. Which existing patterns to follow (include file paths and line references where known).

**Verify:** How to confirm this step is done before moving on.

### Step 2 — [Action]

...

_(Continue for all steps. Err on the side of more steps over fewer — granularity is the point.)_

---

## Unknowns

Things not yet known that could affect how this phase is executed. For each unknown, include a resolution strategy.

| Unknown | Why it matters | How to resolve |
|---------|---------------|----------------|
| [e.g. Which caching layer is already in use?] | Determines whether to add a new adapter or extend existing | Read `config/` and `app/services/` before starting Step 3 |
| [e.g. Does the API client support retry on 429?] | May require wrapping or replacing the client | Check client source or test with a rate-limited request |

If there are no unknowns, write: _None identified — all decisions can be made from existing code._

---

## Outputs

Tangible deliverables this phase produces. Be explicit — list files, endpoints, database changes, etc.

| Output | Type | Description |
|--------|------|-------------|
| `app/services/foo_service.rb` | New file | Service object encapsulating [behaviour] |
| `db/migrate/YYYYMMDDHHMMSS_add_foo.rb` | New file | Migration adding [column] to [table] |
| `GET /api/v1/foo` | New endpoint | Returns [payload shape] |
| `spec/services/foo_service_spec.rb` | New file | Unit tests for the service |

---

## Done Criteria

The phase is complete when **all** of the following are true. These are the benchmarks — not guidelines.

- [ ] [Specific, testable criterion — e.g. "All unit tests in `spec/services/foo_service_spec.rb` pass"]
- [ ] [e.g. "`GET /api/v1/foo` returns a 200 with the correct payload shape when called with a valid token"]
- [ ] [e.g. "The migration runs cleanly on a fresh database with `rails db:migrate`"]
- [ ] [e.g. "No N+1 queries are introduced — verified by running the endpoint with `rack-mini-profiler` in development"]
- [ ] [e.g. "The linter passes with no new warnings"]

**Do not mark the phase complete until every item above is checked.**

---

## What This Phase Does NOT Include

Explicitly list anything adjacent to this phase that is intentionally deferred to a later phase. This prevents scope creep during implementation.

- [e.g. UI changes — those are in Phase 3]
- [e.g. Background job scheduling — deferred until the service is stable]

---

## Handoff to Next Phase

What the next phase depends on from this one. State the contract clearly so there is no ambiguity.

- [e.g. Phase 2 expects `FooService#call` to exist and accept a `user_id` argument]
- [e.g. The migration must be run before Phase 2's seed data scripts will work]
````

---

## Phase 5 — Save the Brief

Save the brief to disk alongside the parent plan:

- Directory: same folder as the parent plan (typically `docs/plans/`)
- Filename: derive from the parent plan filename, appending the phase identifier
  - Parent: `2026-04-12-001-feat-user-auth-plan.md`
  - Phase 1 brief: `2026-04-12-001-feat-user-auth-plan-phase-1.md`
  - Named phase: `2026-04-12-001-feat-user-auth-plan-phase-onboarding.md`

Confirm: "Phase brief written to `[path]`."

---

## Phase 6 — Create the Branch(es)

Before handing off to implementation, create the branch(es) defined in the brief's Branch Strategy.

For each branch in the table:

```bash
git checkout -b [branch-name]
```

If the phase uses multiple branches, create them in dependency order (the primary branch first, then any branches that fork from it).

Confirm each branch was created: "Branch `[branch-name]` created."

If the user is already on the correct branch from a prior phase, confirm that instead of creating a duplicate.

---

## Phase 7 — Present Next Steps

After saving the brief and creating the branch(es), ask the user what they'd like to do next:

1. **Start `ce:work`** — pass the phase brief to `ce:work` to begin implementation; use `commit-review` at each logical stopping point to group and commit changes
2. **Review the brief** — open the file for inspection before starting
3. **Adjust the brief** — make changes based on feedback, then re-save
4. **Deepen further** — run additional research on a specific section of the brief

Remind the user: when implementation produces a natural stopping point (a step complete, a layer done), run `commit-review` to stage and commit that work before moving on — do not save committing for the very end.

Act on the selection immediately.

---

## Rules

- Do not write or modify any application code. This skill only produces the phase brief document.
- Do not skip the codebase analysis in Phase 3 — the brief must reflect reality, not just the plan's assumptions.
- Every unknown must have a resolution strategy — "investigate later" is not acceptable.
- Done criteria must be testable and unambiguous. Avoid subjective criteria like "the feature works well."
- If the plan has no phases and is a flat list of tasks, treat the whole plan as a single phase.
