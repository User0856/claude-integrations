---
name: Version Control Standards
description: Git workflow, branch naming, commit message format, pull request conventions, and repository standards for traceable, reviewable changes
when_to_apply: When creating branches, writing commit messages, pushing code, or creating pull requests
version: 1.0.0
languages: Java, Kotlin
globs: "**"
alwaysApply: false
---

# Version Control Standards

## Overview

Every code change must be traceable from a ticket through a branch, commit, and pull request. This skill defines the Git workflow — branch naming, commit message format, PR conventions, and repository rules. Following these standards ensures that the Git history tells a clear story of what changed, why, and who approved it.

**Core principles:**
- Every change is traceable from ticket to branch to commit to PR
- Branch names are lowercase, commit ticket references are uppercase
- One branch per ticket, always branched from main
- Commit messages describe what changed and why
- Never force-push without explicit permission

## When to Apply This Skill

- Creating a new branch for a ticket
- Writing commit messages
- Pushing code to a remote repository
- Creating pull requests
- Reviewing PRs for VCS compliance
- Merging branches

---

## Quick Reference

| Task | Format | Example |
|------|--------|---------|
| Feature branch | `feature/<ticket-id>` | `feature/cms-42` |
| Hotfix branch | `hotfix/<ticket-id>` | `hotfix/cms-99` |
| Commit message | `<TICKET>: <message>` | `CMS-42: Add client activity summary endpoint` |
| Doc-only commit | `chore: <message>` | `chore: Update README with setup instructions` |
| PR title | `<TICKET>: <message>` | `CMS-42: Add client activity summary endpoint` |

**Branch names** are always lowercase. **Ticket references** in commits and PRs are always UPPERCASE.

| Rule | Requirement | Level | Key Point |
|------|------------|-------|-----------|
| VCS01 | Feature branch naming | MUST | feature/<ticket-id>, always lowercase |
| VCS02 | Always branch from main | MUST | git checkout -b feature/cms-42 origin/main |
| VCS03 | One branch per ticket | MUST | Each branch contains changes for exactly one ticket |
| VCS04 | Commit message format | MUST | <TICKET>: <descriptive message> |
| VCS05 | Describe what and why | MUST | Subject line explains the change and its reason |
| VCS06 | Doc-only commit exception | MAY | chore: prefix for non-ticket documentation changes |
| VCS07 | Multiple commits allowed | MAY | Logical steps sharing the same ticket prefix |
| VCS08 | PR title matches commit format | MUST | <TICKET>: <descriptive message> |
| VCS09 | PR body required sections | MUST | Summary, Ticket, Changes, Testing, Checklist |
| VCS10 | PR size under 500 lines | SHOULD | Split or justify if exceeding 500 significant lines |
| VCS11 | Link to Jira ticket | MUST | Clickable link in PR body |
| VCS12 | Push with upstream tracking | MUST | git push -u origin feature/cms-42 |
| VCS13 | Never force-push without confirmation | MUST | No --force or --force-with-lease without permission |
| VCS14 | Squash and merge | SHOULD | Single clean commit on main per PR |
| VCS15 | Verify before pushing | MUST | Branch name, commit messages, no conflicts, tests pass |

---

## Branch Naming

### Rule VCS01: Feature Branches Use `feature/<ticket-id>`

**Requirement Level**: MUST

```bash
# Correct
git checkout -b feature/cms-42 main
git checkout -b feature/cms-123 main

# Wrong
git checkout -b CMS-42            # missing prefix
git checkout -b feature/CMS-42    # uppercase ticket ID
git checkout -b feature-cms-42    # wrong separator
git checkout -b feat/cms-42       # wrong prefix
```

### Rule VCS02: Always Branch from `main`

**Requirement Level**: MUST

```bash
# Ensure main is up to date
git fetch origin

# Create branch from main
git checkout -b feature/cms-42 origin/main
```

Never branch from another feature branch unless explicitly coordinated.

### Rule VCS03: One Branch per Ticket

**Requirement Level**: MUST

Each feature branch contains changes for exactly one ticket. If a ticket requires changes across multiple repositories, create one branch per repository with the same ticket ID.

---

## Commit Messages

### Rule VCS04: Format: `<TICKET>: <descriptive message>`

**Requirement Level**: MUST

```bash
# Good
git commit -m "CMS-42: Add client activity summary endpoint"
git commit -m "CMS-42: Add unit tests for ClientActivityService"
git commit -m "CMS-1: Fix primary contact reassignment not updating old contact"

# Bad
git commit -m "fix stuff"                      # no ticket, not descriptive
git commit -m "cms-42: add endpoint"           # lowercase ticket ID
git commit -m "CMS-42"                         # no message
git commit -m "WIP"                            # not descriptive
git commit -m "CMS-42: Updated files"          # too vague
```

### Rule VCS05: Commit Messages Describe What and Why

**Requirement Level**: MUST

The subject line should describe WHAT changed and WHY:
- `CMS-42: Add client activity summary endpoint` (WHAT: new endpoint, WHY: implied by ticket)
- `CMS-1: Fix primary contact reassignment skipping old contact update` (WHAT: fix, WHY: was skipping)

For complex changes, add a body:
```
CMS-42: Add client activity summary endpoint

- New ActivitySummaryService with weighted aggregation
- GET /api/v1/clients/{id}/activity-summary endpoint
- Unit tests for score calculation
- Integration test for full request cycle
```

### Rule VCS06: Exceptions for Doc-Only Changes

**Requirement Level**: MAY

Documentation-only changes that don't relate to a ticket may use:
```bash
git commit -m "chore: Update README with MongoDB setup instructions"
git commit -m "chore: Add CONTRIBUTING.md"
```

### Rule VCS07: Multiple Commits Are Acceptable

**Requirement Level**: MAY

Multiple commits on a feature branch are acceptable when they represent logical steps:
```
CMS-42: Add domain model for client activity summary
CMS-42: Add ActivitySummaryService with score calculation
CMS-42: Add REST endpoint and integration test
```

All commits on the branch share the same ticket prefix. Squash merging will combine them into one commit on `main`.

---

## Pull Requests

### Rule VCS08: PR Title Matches Commit Format

**Requirement Level**: MUST

```
CMS-42: Add client activity summary endpoint
```

Same format as commit messages: `<TICKET>: <descriptive message>`.

### Rule VCS09: PR Body Must Include Required Sections

**Requirement Level**: MUST

```markdown
## Summary

Brief description of what this PR does and why.

## Ticket

[CMS-42](https://jira.example.com/browse/CMS-42)

## Changes

- Added `ActivitySummaryService` with weighted score calculation
- Added `GET /api/v1/clients/{id}/activity-summary` endpoint
- Added unit tests for service and mapper
- Added integration test for full request cycle

## Testing

- [x] Unit tests added
- [x] Integration tests added
- [x] All existing tests pass
- [x] Manual verification performed

## Checklist

- [x] Code follows project conventions
- [x] Self-review completed
- [x] No debug code or TODOs
- [x] No merge conflicts
```

### Rule VCS10: PR Size Should Be Under 500 Lines

**Requirement Level**: SHOULD

If a PR exceeds 500 lines of significant changes (excluding generated code, test fixtures, and imports):
1. Consider whether it can be split into sequential PRs
2. If it cannot be split, explain why in the PR description
3. Highlight the key areas for reviewers to focus on

### Rule VCS11: Link to Jira Ticket

**Requirement Level**: MUST

The PR body must contain a clickable link to the Jira ticket. Format:
```markdown
## Ticket

[CMS-42](https://jira.example.com/browse/CMS-42)
```

If the ticket source is a GitHub Issue, link to it:
```markdown
## Ticket

Closes #42
```

---

## Push and Merge

### Rule VCS12: Push with Upstream Tracking

**Requirement Level**: MUST

```bash
git push -u origin feature/cms-42
```

### Rule VCS13: Never Force-Push Without Confirmation

**Requirement Level**: MUST

Force-pushing rewrites history and can destroy work. Never do it without explicit user instruction:

```bash
# FORBIDDEN without explicit permission
git push --force
git push --force-with-lease

# If needed after rebase, ask first
```

### Rule VCS14: Squash and Merge

**Requirement Level**: SHOULD

When merging PRs, prefer "Squash and Merge" to create a single clean commit on `main`. This keeps the main branch history readable.

---

## Pre-Push Checklist

### Rule VCS15: Verify Before Pushing

**Requirement Level**: MUST

```bash
# 1. Verify branch name
git rev-parse --abbrev-ref HEAD
# Should be: feature/cms-42 (lowercase)

# 2. Verify commit messages
git log --oneline main..HEAD
# Each should start with TICKET-ID:

# 3. Verify no merge conflicts with main
git fetch origin main
git merge --no-commit --no-ff origin/main; git merge --abort 2>/dev/null
# If merge reports conflicts, resolve before pushing

# 4. Verify all tests pass
./gradlew build

# 5. Check diff size
git diff --stat origin/main
```

---

## Deriving the Ticket ID

When invoking `/resolve-ticket`, the ticket ID is provided as an argument. Extract it as follows:

1. **From Jira ID** (e.g., `CMS-42`): Use directly
   - Branch: `feature/cms-42` (lowercase)
   - Commit: `CMS-42: ...` (uppercase)

2. **From GitHub Issue URL** (e.g., `https://github.com/org/repo/issues/42`): Use `#42`
   - Branch: `feature/issue-42` (lowercase)
   - Commit: `#42: ...`

3. **From plain text**: Generate a slug from the description
   - Branch: `feature/add-client-activity-summary`
   - Commit: `Add client activity summary endpoint`

---

## Enforcement Checklist

Before pushing, verify:

- [ ] Branch name is `feature/<ticket-id>` (lowercase)
- [ ] Branch was created from `main`
- [ ] All commit messages follow `<TICKET>: <message>` format
- [ ] Ticket ID is uppercase in commits and PR title
- [ ] PR body includes Summary, Ticket link, Changes, Testing, and Checklist sections
- [ ] PR is under 500 significant lines (or justified)
- [ ] No merge conflicts with `main`
- [ ] All tests pass
- [ ] No force-push performed

## References

- For complete code examples, see [examples.md](examples.md)
- Conventional Commits: https://www.conventionalcommits.org/
- Git Flow: https://nvie.com/posts/a-successful-git-branching-model/
- GitHub CLI (`gh`): https://cli.github.com/manual/
