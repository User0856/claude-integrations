# Version Control Examples

Real examples demonstrating Git workflow, commit messages, and PR conventions.

---

## Branch Naming Examples

### Good Branch Names

```bash
feature/cms-42          # standard feature branch for ticket CMS-42
feature/cms-108         # another ticket
hotfix/cms-99           # hotfix for production bug CMS-99
feature/issue-15        # GitHub Issue #15
```

All lowercase. Prefix matches the work type. Ticket ID follows the slash.

### Bad Branch Names

```bash
CMS-42                  # missing prefix (feature/ or hotfix/)
feature/CMS-42          # uppercase ticket ID — must be lowercase
feature-cms-42          # hyphen separator — must use slash
feat/cms-42             # abbreviated prefix — must be "feature"
feature/cms-42-add-client-endpoint  # extra description — ticket ID is sufficient
add-client-endpoint     # no prefix and no ticket ID
feature/cms-42-cms-99   # two tickets — one branch per ticket
```

---

## Commit Message Examples

### Good Commit Messages (Subject Only)

```
CMS-42: Add client activity summary endpoint
```

```
CMS-42: Add unit tests for ClientActivityService
```

```
CMS-1: Fix primary contact reassignment not updating old contact
```

```
chore: Update README with MongoDB setup instructions
```

Each starts with the ticket ID (uppercase) followed by a colon and a descriptive message. The `chore:` prefix is reserved for documentation-only changes with no ticket.

### Good Commit Message (With Body)

```
CMS-42: Add invoice generation for active contracts

- New InvoiceService.createFromContract() with line-item mapping
- Added validation: only ACTIVE contracts can be invoiced
- GET /api/v1/invoices/{id} and POST /api/v1/invoices endpoints
- Unit tests for service, mapper, and validation logic
- Integration test for full invoice creation cycle
```

The subject line describes what changed. The body lists the key changes as bullet points.

### Bad Commit Messages

```
fix stuff
```
No ticket ID. Not descriptive.

```
cms-42: add endpoint
```
Ticket ID must be uppercase (`CMS-42`).

```
CMS-42
```
No descriptive message after the ticket ID.

```
WIP
```
Not descriptive, no ticket ID. Every commit should be meaningful.

```
CMS-42: Updated files
```
Too vague. Say which files or what behavior changed.

```
CMS-42: Added ClientService.java and ClientRepository.java and ClientController.java and ClientMapper.java and CreateClientRequest.java and ClientResponse.java and ClientServiceTest.java
```
Lists file names instead of describing the behavior. Better: `CMS-42: Add client CRUD operations`.

---

## Pull Request Body Example

A complete PR body following the required format for ticket CMS-42:

```markdown
## Summary

Add the ability to generate invoices from active contracts. The endpoint validates
that the contract is in ACTIVE status, maps line items to invoice entries, and
assigns DRAFT status to the new invoice.

## Ticket

[CMS-42](https://jira.example.com/browse/CMS-42)

## Changes

- Added `InvoiceService.createFromContract()` with contract status validation
- Added `InvoiceMapper` for entity/DTO conversion with line-item mapping
- Added `POST /api/v1/invoices` endpoint in `InvoiceController`
- Added `GET /api/v1/invoices/{id}` endpoint for retrieval
- Added unit tests for `InvoiceService` (8 tests) and `InvoiceMapper` (3 tests)
- Added integration test for invoice creation and retrieval cycle

## Testing

- [x] Unit tests added for service and mapper
- [x] Integration tests added for API endpoints
- [x] All existing tests pass (`./gradlew build` clean)
- [x] Manual verification: created invoice from contract CNT-001 via Postman

## Checklist

- [x] Code follows project conventions
- [x] Self-review completed (CR01-CR14)
- [x] No debug code or TODOs
- [x] No sensitive data in logs
- [x] No merge conflicts with main
```

---

## Pre-Push Verification

Example terminal output showing the full pre-push checklist (Rule VCS15):

```bash
# 1. Verify branch name
$ git rev-parse --abbrev-ref HEAD
feature/cms-42

# 2. Verify commit messages
$ git log --oneline main..HEAD
a1b2c3d CMS-42: Add invoice generation for active contracts
e4f5a6b CMS-42: Add unit tests for InvoiceService and InvoiceMapper
c7d8e9f CMS-42: Add integration test for invoice creation endpoint

# 3. Verify no merge conflicts with main
$ git fetch origin main
From github.com:enterprise/cms
 * branch            main       -> FETCH_HEAD
$ git merge --no-commit --no-ff origin/main
Already up to date.

# 4. Verify all tests pass
$ ./gradlew build
BUILD SUCCESSFUL in 34s
8 actionable tasks: 8 executed

# 5. Check diff size
$ git diff --stat origin/main
 src/main/java/.../controller/InvoiceController.java   |  42 +++++
 src/main/java/.../domain/model/Invoice.java            |  38 +++++
 src/main/java/.../dto/CreateInvoiceRequest.java        |  22 +++
 src/main/java/.../dto/InvoiceResponse.java             |  26 +++
 src/main/java/.../mapper/InvoiceMapper.java            |  18 +++
 src/main/java/.../repository/InvoiceRepository.java    |  12 ++
 src/main/java/.../service/InvoiceService.java          |  56 +++++++
 src/test/java/.../service/InvoiceServiceTest.java      |  98 ++++++++++++
 src/test/java/.../mapper/InvoiceMapperTest.java        |  45 ++++++
 src/test/java/.../integration/InvoiceIntegrationTest.java | 72 +++++++++
 10 files changed, 429 insertions(+)

# All checks passed — safe to push
$ git push -u origin feature/cms-42
Enumerating objects: 24, done.
Counting objects: 100% (24/24), done.
Delta compression using up to 10 threads
Compressing objects: 100% (14/14), done.
Writing objects: 100% (18/18), 6.12 KiB | 6.12 MiB/s, done.
Total 18 (delta 6), reused 0 (delta 0)
remote: Resolving deltas: 100% (6/6), completed with 4 local objects.
To github.com:enterprise/cms.git
 * [new branch]      feature/cms-42 -> feature/cms-42
branch 'feature/cms-42' set up to track 'origin/feature/cms-42'.
```

All five checks passed: correct branch name, properly formatted commit messages, no merge conflicts, green build, and diff under 500 lines. The branch is pushed with upstream tracking (`-u`).
