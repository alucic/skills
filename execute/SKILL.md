---
name: execute
description: Execute a plan from a GitHub issue. Reads the issue, spins up subagents for each delivery step, tracks progress via issue comments, and handles plan evolution.
argument-hint: <github-issue-number-or-url>
allowed-tools: Bash(gh *) Read Write Edit Grep Glob Agent
---

You are executing a plan that lives in a GitHub issue. Your job is to coordinate subagents that each deliver one step of the plan.

## Phase 1: Load the Plan

1. Fetch the issue using `gh issue view $ARGUMENTS --comments` (extract owner/repo from URL if full URL given, or use current repo if just a number).
2. Parse the plan from the issue body. Identify all delivery steps from Section 5.
3. Read all comments on the issue to understand any evolution — decisions made, steps completed, scope changes from prior runs.
4. Determine which steps are already done (check for completion comments) and which are next.

If the issue doesn't contain a plan in the expected format, tell the user and stop.

## Phase 2: Execute Steps

For each pending step, in order:

### Before launching a subagent

1. Re-read the issue comments to catch any updates from previously completed steps.
2. Identify the step's dependencies — has every prior step it depends on been completed?
3. If parallel execution is possible (two steps with no shared files and no dependency), launch subagents concurrently.

### Subagent instructions

Spin up a subagent for each step using the Agent tool. Give the subagent:

- The full plan context (summary, architecture, relevant interfaces from Section 3)
- The specific step to execute (what, files, interface contracts, acceptance criteria)
- Clear boundaries: **only touch the files listed in your step**
- Instruction to verify acceptance criteria before reporting done
- The current state of the codebase (any relevant context from prior steps)

The subagent prompt should follow this structure:

```
You are implementing Step N of a plan.

## Context
[Summary from Section 1 — what the plan is about]

## Architecture
[Relevant portion of Section 2 — the components this step touches]

## Interface Contracts You Must Respect
[Relevant interfaces from Section 3 — both what this step implements and what it depends on]

## Your Step
[The full step description from Section 5]

## Rules
- Only modify these files: [list from the step]
- Implement the interface contracts exactly as specified
- Verify the acceptance criteria before reporting completion
- If you encounter something unexpected that requires changing the plan, STOP and report back what you found and what you think should change. Do NOT improvise outside your step's scope.
```

### After each subagent completes

1. Review what the subagent did — check that it stayed within scope and met acceptance criteria.
2. Post a comment on the GitHub issue:

```bash
gh issue comment $ISSUE_NUMBER --body "## Step N: [title] ✅

**Completed by**: subagent
**What was done**: [brief summary of changes]
**Files modified**: [list]
**Acceptance criteria met**: [yes/no with details]
**Notes**: [anything notable — deviations, surprises, decisions made]"
```

3. **Open a pull request for this step's changes, then STOP.** Create a branch off `main` for the step, commit the changes, push, and open a PR that references the plan issue (e.g. `Closes part of #$ISSUE_NUMBER — Step N`). After the PR is open, report the PR URL to the user and **do not proceed to the next step** until the user tells you to continue (typically after they've reviewed and merged the PR). The next step's subagent should start from an updated `main` that includes the merged PR.

4. If the subagent reports a problem that requires a plan change, handle it (see Plan Evolution below).

### If a subagent fails or reports a blocker

1. Post a comment explaining what happened.
2. Assess whether the blocker affects downstream steps.
3. Report to the user with options: retry, skip, or revise the plan.

## Plan Evolution

If during execution something changes — a step reveals the plan was wrong, a dependency doesn't work as expected, or the scope needs adjustment:

1. **Do NOT silently change the plan.** Always post a comment on the issue explaining:
   - What was discovered
   - Why the original plan doesn't work
   - What is being changed and why
   - Impact on remaining steps

```bash
gh issue comment $ISSUE_NUMBER --body "## ⚠️ Plan Update

**During**: Step N — [title]
**Discovery**: [what was found]
**Original assumption**: [what the plan expected]
**Decision**: [what we're doing instead]
**Impact on remaining steps**: [which steps are affected and how]"
```

2. Adjust remaining steps accordingly before continuing.

## Completion

When all steps are done:

1. Post a final summary comment on the issue:

```bash
gh issue comment $ISSUE_NUMBER --body "## ✅ Plan Complete

**Steps completed**: N/N
**Summary of all changes**: [high-level summary]
**Files modified across all steps**: [consolidated list]
**Any deviations from original plan**: [list or 'none']"
```

2. Report to the user with the full summary.

## Rules

- The GitHub issue is the single source of truth — every decision and completion gets posted there
- A completely context-free agent should be able to read the issue top-to-bottom and understand the full history
- Subagents must not modify files outside their step's scope
- If a subagent needs to touch a file owned by another step, that's a plan sequencing problem — escalate, don't hack around it
- Always verify acceptance criteria, don't take subagent's word for it
- Prefer sequential execution unless the plan explicitly marks steps as parallelizable
