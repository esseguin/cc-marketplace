---
description: "Review local code changes for quality, functionality, and documentation. Use when the user says things like: 'review my changes', 'review this branch', 'code review', 'check my code', 'review before I commit', 'what do you think of these changes', 'look over my diff', or any request to evaluate uncommitted or branch-level changes. Proactively use this even if the user doesn't say 'review' explicitly but is clearly asking for feedback on their recent work."
---

Review the user's local code changes, provide structured findings with severity scores, and offer to fix issues.

## Step 1: Determine Scope

Figure out what to review based on the user's prompt and the repo state.

Run these commands to assess the situation:

```
git status
git branch --show-current
git log --oneline -1 @{upstream} 2>/dev/null || echo "no upstream"
```

**Scope rules:**

- If the user explicitly asks for a "branch review" or "PR review", review the full branch diff (even if there are also uncommitted changes — review both)
- If the user explicitly asks to review "my changes" or "uncommitted changes", review only the working tree
- If the user doesn't specify: check for uncommitted changes first. If there are uncommitted changes, review those. If the working tree is clean, review the full branch diff.
- If there are both uncommitted changes and the user's intent is ambiguous, review uncommitted changes but mention that a full branch review is available if they want it.

**For branch reviews**, find the merge base:

```
git merge-base HEAD main 2>/dev/null || git merge-base HEAD master 2>/dev/null
```

Use `git diff <merge-base>..HEAD` and `git log --oneline <merge-base>..HEAD` to understand the full scope.

**For uncommitted changes**, use `git diff` (unstaged) and `git diff --cached` (staged).

Tell the user what scope you're reviewing and why, e.g.: "Reviewing uncommitted changes (3 files modified)" or "Reviewing branch `feature/foo` — 12 commits since diverging from `main`".

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

Evaluate each change across these categories, weighted by relevance to what actually changed:

### Always review:

- **Functionality**: Does this do what it's supposed to? Are there bugs, logic errors, edge cases, or regressions? Is it performant — any obvious O(n^2) loops, unnecessary allocations, missing caches, or repeated work?
- **Code Quality**: Readability, conciseness, naming, adherence to project conventions (from CLAUDE.md), DRY, appropriate abstraction level. Are there simpler ways to achieve the same thing?
- **Documentation**: Do new public APIs have adequate docs? Do changed behaviors have updated docs? Are there now-stale comments, READMEs, or docstrings that reference old behavior?

### Review when relevant (use your judgment):

- **Architecture**: Does this change fit the existing patterns? Does it introduce unnecessary coupling or complexity?
- **Error handling**: Are failure modes handled? Are error messages helpful?
- **Testing**: If the project has tests, are new behaviors tested? Are existing tests now stale?
- **Anything else** you notice that a senior engineer would flag — use your expertise

## Step 4: Present Findings

Format your review as follows:

### Summary Header

Start with a one-line overall assessment, then the scope:

```
## Code Review: [brief description of what changed]

**Scope**: [Uncommitted changes | Branch `name` — N commits since `base-branch`]
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

## Step 5: Recommend and Fix

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
