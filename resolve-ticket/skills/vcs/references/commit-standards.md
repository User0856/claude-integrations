# Commit Message Standards — Quick Reference

## Format

```
<TICKET-ID>: <subject line — what and why, imperative mood, under 72 chars>

<optional body — bullet points for complex changes>
```

## Good Examples

```
CMS-42: Add client activity summary endpoint
CMS-1: Fix primary contact reassignment not updating old contact
CMS-3: Implement automated contract renewal workflow
CMS-4: Add missing unit tests for TaskService
CMS-5: Add bulk operations support for client tags
```

## With Body

```
CMS-42: Add client activity summary endpoint

- New ActivitySummaryService with weighted scoring algorithm
- GET /api/v1/clients/{id}/activity-summary returns aggregated scores
- Scores based on contracts (30%), payments (25%), interactions (20%),
  tasks (15%), and renewal proximity (10%)
- Unit tests cover all scoring branches
- Integration test verifies full request-response cycle
```

## Bad Examples

| Bad | Why | Fix |
|-----|-----|-----|
| `fix bug` | No ticket, no detail | `CMS-1: Fix primary contact reassignment` |
| `cms-42: add stuff` | Lowercase ticket, vague | `CMS-42: Add client activity summary endpoint` |
| `CMS-42` | No description | `CMS-42: Add unit tests for HealthScoreService` |
| `WIP` | Not descriptive | `CMS-42: Add initial domain model for activity summary` |
| `Updated ContractService.java` | Describes files, not behavior | `CMS-3: Add renewal status transition with validation` |
| `misc changes` | Completely meaningless | Describe the actual change |

## Rules

1. **Ticket ID is UPPERCASE** in the commit message: `CMS-42`, not `cms-42`
2. **Subject line uses imperative mood**: "Add feature" not "Added feature" or "Adding feature"
3. **Subject line under 72 characters** — this is a hard limit for git tooling
4. **No period at the end** of the subject line
5. **Blank line between subject and body** if a body is present
6. **Body wraps at 72 characters** per line
7. **Body uses bullet points** for multiple changes
8. **One logical change per commit** — don't mix unrelated changes

## Exception: Documentation-Only Changes

```
chore: Update README with MongoDB setup instructions
chore: Add API documentation for contract endpoints
```

Use `chore:` prefix only when no ticket exists and the change is purely documentation.
