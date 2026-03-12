---
description: "Review local code changes for quality, functionality, and documentation. Use when the user says things like: 'review my changes', 'review this branch', 'code review', 'check my code', 'review before I commit', 'what do you think of these changes', 'look over my diff', or any request to evaluate uncommitted or branch-level changes. Proactively use this even if the user doesn't say 'review' explicitly but is clearly asking for feedback on their recent work."
---

<!-- Inspired by https://github.com/anthropics/claude-plugins-official/blob/main/plugins/code-review/commands/code-review.md -->

Review the user's local code changes, provide structured findings with severity scores, and offer to fix issues.

## Step 1: Determine Scope

Figure out what to review based on the user's prompt and the repo state.

Run these commands in parallel to assess the situation:

```
git status --short
git branch --show-current
gh pr view --json number,title,url,baseRefName 2>/dev/null || echo "no PR"
```

**Scope rules (in priority order):**

1. If there are uncommitted changes (staged or unstaged), review those using `git diff` and `git diff --cached`. Mention that a PR or full branch review is also available if relevant.
2. If the working tree is clean and a PR exists, use `gh pr diff` to get the changes and `gh pr view` for context. This is the preferred path for branch reviews — simpler and more accurate than computing merge bases manually.
4. **Last resort** — no PR and clean working tree: fall back to a local branch diff. Find the merge base:
   ```
   git merge-base HEAD main 2>/dev/null || git merge-base HEAD master 2>/dev/null
   ```
   Use `git diff <merge-base>..HEAD` and `git log --oneline <merge-base>..HEAD`.

Tell the user what scope you're reviewing and why, e.g.: "Reviewing uncommitted changes (3 files modified)" or "Reviewing PR #42 — `feature/foo`" or "Reviewing branch `feature/foo` — 12 commits since diverging from `main`".

## Step 2: Gather Context

Before reviewing, build understanding:

1. **Read CLAUDE.md** (root and any in affected directories) to understand project conventions
2. **Read the actual diff** in full — don't summarize prematurely
3. **Read surrounding code** when the diff alone isn't enough to understand intent (e.g., a function signature changed — read the callers)
4. **Check commit messages** (for branch reviews) to understand the narrative of the changes

If the changes are large or the intent is unclear, **ask the user questions** before proceeding. Examples of good questions:

- "This branch touches both the auth system and the UI layer — is this one feature or two separate things?"
- "I see you removed the retry logic in `fetch_data()` — was that intentional or a side effect of the refactor?"
- "What's the expected behavior when `config.timeout` is not set?"

Don't ask questions about things you can figure out from the code. Only ask when the answer genuinely affects your review.

## Step 3: Review the Changes

Launch parallel Sonnet subagents to review the changes from different angles. Each agent gets the diff and the list of CLAUDE.md file paths, then focuses on its assigned category. Sonnet is the right model here — the review categories are well-scoped and don't need heavy reasoning. This parallelism makes reviews faster and ensures each category gets dedicated attention rather than being rushed through sequentially.

Spawn these agents in parallel (using Sonnet):

### Agent 1: Functionality & Bugs
Scan the diff for logic errors, edge cases, regressions, and performance issues (O(n²) loops, unnecessary allocations, missing caches, repeated work). Read surrounding code when needed to understand call sites and data flow. Focus on real bugs that would affect users — not things a linter or type checker would catch.

### Agent 2: Code Quality & Conventions
Check readability, naming, DRY, appropriate abstraction level, and adherence to project conventions from CLAUDE.md. Look for simpler ways to achieve the same thing. Not all CLAUDE.md instructions apply during review (some are about how to write code interactively), so use judgment about which ones are relevant.

### Agent 3: Dead Code & Completeness
Look for newly orphaned code: files, functions, imports, or variables that nothing references anymore — common leftovers after refactors, renames, or feature removals. Check whether old files were meant to be deleted. Also check for incomplete changes: did the author update the implementation but forget to update related tests, docs, config, or types?

### Agent 4: Architecture & Error Handling
Does this change fit existing patterns? Does it introduce unnecessary coupling or complexity? Are failure modes handled? Are error messages helpful? If the project has tests, are new behaviors tested and are existing tests still valid?

Each agent should return a list of findings, where each finding includes:
- A short title
- The file and line(s) affected
- A clear explanation of the issue and why it matters
- A suggested category (Functionality, Quality, Dead Code, Architecture, Error Handling, Testing, Documentation)

## Step 4: Filter Findings

After collecting findings from all agents, use a Haiku subagent to do a quick sanity pass and filter out false positives before presenting to the user. Give it the diff and the combined findings list. Drop findings that match these patterns:

- **Pre-existing issues** — problems on lines the user didn't modify, unless the user's change makes them worse
- **Linter/compiler territory** — missing imports, type errors, formatting issues, style nitpicks that automated tools would catch
- **Intentional changes** — functionality changes that are clearly deliberate and directly related to the purpose of the diff
- **Generic quality complaints** — vague concerns about test coverage, security, or documentation unless specifically required by CLAUDE.md
- **Silenced warnings** — issues called out in CLAUDE.md but explicitly suppressed in code (e.g., lint ignore comments)

If a finding wouldn't survive light scrutiny from a senior engineer who understands the context, drop it. A shorter list of real issues is far more valuable than a long list padded with noise.

## Step 5: Present Findings

Format your review as follows:

### Summary Header

Start with a one-line overall assessment, then the scope:

```
## Code Review: [brief description of what changed]

**Scope**: [Uncommitted changes | PR #N — `branch-name` | Branch `name` — N commits since `base-branch`]
**Files reviewed**: N files changed (+X, -Y lines)
**Overall**: [Brief 1-sentence assessment — e.g. "Solid implementation with a few issues to address" or "Looks good, minor suggestions only"]
```

### Findings Table

Present a summary table first for quick scanning:

```
| # | Severity | Category | Finding |
|---|----------|----------|---------|
| 1 | 5 | Functionality | Brief description |
| 2 | 3 | Quality | Brief description |
| ... |
```

**Severity scale (1-5):**

- **5 — Must fix**: Bugs, data loss, security holes, crashes. The code is broken or dangerous.
- **4 — Should fix**: Significant issues that will cause problems — poor error handling for likely cases, performance issues at expected scale, violations of critical project conventions.
- **3 — Recommended**: Real issues worth fixing — confusing code, missing docs for public APIs, minor performance concerns, moderate convention violations.
- **2 — Consider**: Minor improvements — slightly better naming, small readability gains, non-critical doc updates.
- **1 — Nitpick**: Very minor style or preference suggestions. Mention only if there's not much else to flag.

### Detailed Findings

After the table, expand each finding:

```
### Finding 1 (Severity 5): [Title]
**Category**: Functionality
**File**: `path/to/file.gd:42`

[Explain the issue clearly. Show the problematic code. Explain *why* it's a problem and what could go wrong. Suggest a fix or direction.]
```

### Positive Callouts

If something is particularly well done, say so briefly at the end. Engineers appreciate knowing what's working well, not just what's wrong.

## Step 6: Recommend and Fix

After presenting findings, recommend which ones are worth fixing:

```
## Recommended Fixes

I'd recommend fixing findings #1, #2, and #4 (severity 3+). Findings #3 and #5 are minor and can be addressed later.

Want me to fix these? I'll put together a plan for your approval.
```

Wait for the user to tell you which findings they want fixed. They may agree with your recommendations, pick a subset, or add ones you marked as optional.

Once the user confirms which findings to fix, use `EnterPlanMode` to create a plan that addresses all selected findings. The plan should:

- Group related fixes together
- Note any fixes that might interact with each other
- Be specific about what changes in which files

After the user approves the plan, exit plan mode and implement all fixes. Then run any project-specific validation (like the checks mentioned in CLAUDE.md) to make sure the fixes don't introduce new issues.
