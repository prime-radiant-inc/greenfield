---
name: git-archaeology
description: Layer 1 skill for mining git history for behavioral intelligence. Repository overview, commit message mining, PR/MR description extraction, blame-based maintenance heat maps, issue tracker cross-referencing. Loaded by the analyzer agent during Layer 1.
---

# Git Archaeology Methodology

Extract behavioral intelligence from version control history. Commit messages, PR descriptions, blame annotations, and issue references encode design decisions, behavioral changes, and maintenance patterns that no other source captures.

## When to Use This Mode

Git archaeology activates when:
- The target is a git repository (or has a `.git` directory)
- The discovery inventory identifies a git repository with meaningful commit history
- Other modes discover a git repository during analysis

This mode runs independently of all other intelligence sources. It requires only access to the git repository and optionally to a GitHub/GitLab remote. All output is **RAW** (commit messages and PR descriptions may reference proprietary internals).

## Why Git History Matters

Source code tells you what the system does NOW. Git history tells you:
- **What changed and why** -- commit messages encode intent that is invisible in the current code
- **What broke and how it was fixed** -- fix commits reveal failure modes and edge cases
- **What was deliberately removed** -- reverted features and deleted code reveal abandoned behaviors
- **Where maintenance concentrates** -- frequently modified files signal complexity and instability
- **What decisions were debated** -- PR descriptions capture design trade-offs that never appear in documentation

## Phase 1: Repository Overview

**Goal:** Establish the scope, age, and shape of the project.

```bash
# Commit count and date range
echo "Total commits: $(git rev-list --count HEAD)"
echo "First commit: $(git log --reverse --format='%ai' | head -1)"
echo "Latest commit: $(git log -1 --format='%ai')"

# Contributors
git shortlog -sn --no-merges | head -20

# Active branches
git branch -r --sort=-committerdate | head -20

# Tags (releases)
git tag --sort=-version:refname | head -20

# Commit frequency (commits per month, last 12 months)
for i in $(seq 0 11); do
  month=$(date -d "$i months ago" +%Y-%m 2>/dev/null || date -v-${i}m +%Y-%m)
  count=$(git rev-list --count --after="${month}-01" --before="${month}-31" HEAD 2>/dev/null || echo "0")
  echo "$month: $count"
done

# Top-level directory structure at HEAD
git ls-tree --name-only HEAD
```

Write to `workspace/raw/project-history/overview.md`.

## Phase 2: Commit Message Mining

**Goal:** Extract behavioral claims from commit messages. Commit messages that describe features, fixes, and breaking changes encode behavioral contracts.

### 2.1 Feature Commits

```bash
# Conventional commits: feat
git log --oneline --grep="^feat" --no-merges | head -100

# Keywords indicating new behavior
git log --oneline --grep="add\|implement\|introduce\|support\|enable" -i --no-merges | head -100
```

For each feature commit, extract:
- What behavior was added (the claim)
- When it was added (the commit date, for version context)
- The commit SHA (for provenance)

### 2.2 Fix Commits

```bash
# Conventional commits: fix
git log --oneline --grep="^fix" --no-merges | head -100

# Keywords indicating bug fixes
git log --oneline --grep="fix\|bug\|repair\|correct\|resolve\|patch" -i --no-merges | head -100
```

Fix commits are especially valuable because they reveal:
- A behavior that was WRONG (the bug)
- A behavior that is now CORRECT (the fix)
- Edge cases that the original implementation missed

### 2.3 Breaking Change Commits

```bash
# Conventional commits: breaking
git log --oneline --grep="BREAKING" --no-merges | head -50

# Keywords indicating behavioral changes
git log --oneline --grep="breaking\|deprecat\|remov\|migration\|upgrade" -i --no-merges | head -50
```

Breaking changes reveal behavioral contracts that were considered important enough to announce their violation.

### 2.4 Detailed Message Extraction

For high-value commits (features, fixes, breaking changes), read the full commit message:

```bash
# Full message for a specific commit
git log -1 --format='%H%n%ai%n%an%n%n%B' <commit-sha>
```

Extract behavioral claims from the message body. Many commit messages contain:
- **Before/after descriptions** -- explicit behavioral contract changes
- **Issue references** -- links to detailed context
- **Test descriptions** -- what was verified
- **Migration instructions** -- how behavior shifted

Write to `workspace/raw/project-history/behavioral-claims.md`.

### Provenance for Commit-Derived Claims

```markdown
- The `--output csv` flag was added in v2.3
  <!-- cite: source=git-history, ref=abc1234, confidence=inferred, agent=git-archaeologist -->
```

## Phase 3: PR/MR Description Mining

**Goal:** Extract behavioral intelligence from pull request and merge request descriptions. PR descriptions often contain the richest behavioral context: motivation, design decisions, testing notes, and review discussions.

### 3.1 GitHub PRs

```bash
# List merged PRs (most recent first)
gh pr list --state merged --limit 100 --json number,title,body,mergedAt,labels

# View a specific PR with full body and comments
gh pr view <number> --json title,body,comments,reviews,labels,mergedAt

# Search PRs by keyword
gh pr list --state merged --search "authentication" --limit 20 --json number,title,body
```

### 3.2 GitLab MRs

If the remote is GitLab rather than GitHub:

```bash
# List merged MRs
glab mr list --state merged --per-page 100

# View a specific MR
glab mr view <number>
```

### 3.3 What to Extract from PRs/MRs

For each PR/MR, extract:
- **Motivation** -- why was this change made? What problem does it solve?
- **Design decisions** -- what alternatives were considered? Why was this approach chosen?
- **Testing notes** -- what was tested? How was it verified?
- **Review comments** -- what concerns were raised? What was changed in response?
- **Behavioral claims** -- explicit statements about what the code does or should do

PR descriptions that contain phrases like "this PR adds", "this fixes", "after this change", "the new behavior is" are direct behavioral claims.

### 3.4 Availability Check

Not all repositories have accessible PR history. Check first:

```bash
# GitHub: check if gh CLI is authenticated and repo is accessible
gh repo view --json name 2>/dev/null && echo "GitHub accessible" || echo "GitHub not accessible"

# Check if remote is GitHub or GitLab
git remote get-url origin
```

If PR history is not accessible, document the gap and move on. Do not treat inaccessibility as failure.

## Phase 4: Blame-Based Maintenance Heat Map

**Goal:** Identify which parts of the codebase are actively maintained, recently modified, or fossilized. Maintenance patterns reveal where behavioral complexity concentrates.

### 4.1 Recently Modified Files

```bash
# Files modified in the last 90 days, sorted by modification count
git log --since="90 days ago" --name-only --no-merges --format="" | sort | uniq -c | sort -rn | head -50
```

### 4.2 Most Frequently Modified Files (All Time)

```bash
# Files with the most commits
git log --name-only --no-merges --format="" | sort | uniq -c | sort -rn | head -50
```

### 4.3 Fossilized Code

```bash
# Files not modified in over a year
git log --diff-filter=M --since="1 year ago" --name-only --no-merges --format="" | sort -u > /tmp/recently-modified.txt
git ls-files | while read f; do
  grep -qx "$f" /tmp/recently-modified.txt || echo "$f"
done | head -100
```

### 4.4 Churn Analysis

```bash
# Files with the highest churn (lines added + removed) in last 6 months
git log --since="6 months ago" --numstat --no-merges --format="" | \
  awk '{adds[$3]+=$1; dels[$3]+=$2} END {for(f in adds) print adds[f]+dels[f], adds[f], dels[f], f}' | \
  sort -rn | head -30
```

### 4.5 Interpreting the Heat Map

| Pattern | Signal |
|---------|--------|
| High recent churn, many contributors | Active development, likely unstable behavior |
| High recent churn, single contributor | Focused refactoring or feature work |
| No modifications in 1+ year | Fossilized: either stable or abandoned |
| Frequent modifications to test files | Behavioral contracts actively evolving |
| Config files frequently modified | Deployment or environment complexity |

Write to `workspace/raw/project-history/maintenance-heatmap.md`.

## Phase 5: Issue Tracker Cross-Reference

**Goal:** Extract issue references from commits and fetch the referenced issues for behavioral context.

### 5.1 Extract Issue References

```bash
# Find commits that reference issues (common patterns: #123, GH-123, JIRA-123)
git log --oneline --no-merges --format="%H %s" | grep -oE '#[0-9]+|[A-Z]+-[0-9]+' | sort | uniq -c | sort -rn | head -50

# Full commit messages that reference issues
git log --no-merges --grep='#[0-9]\+' --format="%H %ai %s" | head -50
```

### 5.2 Fetch Referenced Issues (GitHub)

```bash
# Fetch a specific issue
gh issue view <number> --json title,body,comments,labels,state

# List issues with behavioral labels
gh issue list --label "bug" --state all --limit 50 --json number,title,body,state
gh issue list --label "enhancement" --state all --limit 50 --json number,title,body,state
```

### 5.3 What to Extract from Issues

Issues provide behavioral context that commits alone cannot:
- **Bug reports** -- describe expected vs. actual behavior (behavioral contract violations)
- **Feature requests** -- describe desired behavior (behavioral requirements)
- **Discussion threads** -- capture design decisions and trade-offs
- **Reproduction steps** -- equivalent to test vectors

### 5.4 Availability Check

Issue tracker access depends on the hosting platform and repository visibility. If the issue tracker is not accessible, document the gap. Extract what you can from commit message references alone.

Write to `workspace/raw/project-history/issue-references.md`.

## Provenance Rules

### Source Type

All claims from git history analysis use `source=git-history`:

```markdown
- The retry logic was added after users reported timeout failures on slow connections
  <!-- cite: source=git-history, ref=abc1234, confidence=inferred, agent=git-archaeologist -->
```

### Confidence Levels

- **confirmed** -- the commit message or PR description explicitly states the behavioral claim, AND a second source (e.g., a linked issue or test change) corroborates it
- **inferred** -- the commit message or PR description directly states or strongly implies the behavioral claim from a single source
- **assumed** -- the behavioral claim is derived from commit patterns, file names, or structural reasoning rather than explicit statements

### Cite As You Go

Every behavioral claim gets an inline citation immediately after the claim. The `ref` field should be the commit SHA (short form is acceptable). For PR-derived claims, use `ref=PR-<number>`.

## Output Structure

```
workspace/raw/project-history/
    overview.md                # Repository overview: age, size, contributors, structure
    behavioral-claims.md       # Behavioral claims extracted from commit messages
    maintenance-heatmap.md     # File modification patterns and churn analysis
    issue-references.md        # Issue cross-references and extracted behavioral context
```

## Rules

1. **Git history is RAW** -- commit messages and PR descriptions may contain proprietary information. All output goes to `workspace/raw/`.
2. **Recency matters** -- recent commits are more likely to describe current behavior. Flag the date of every commit-derived claim.
3. **Commit messages are claims, not facts** -- a commit that says "fix authentication bug" does not prove the fix is correct. It is an `inferred` claim until corroborated by another source.
4. **Absence of history is not absence of behavior** -- a feature with no commit history may predate the repository or have been imported wholesale.
5. **Do not read source code diffs** -- this is git ARCHAEOLOGY, not source analysis. Extract behavioral claims from messages, descriptions, and patterns. Source code analysis is a separate mode.
6. **Document what is inaccessible** -- if PR history, issue trackers, or remote repositories are not reachable, document the gap explicitly. Partial results are valuable.
7. **Cite as you go** -- every behavioral claim gets an inline `<!-- cite: -->` comment immediately after the claim. Never defer citation to a later step.
