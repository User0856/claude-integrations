---
name: Code Review Standards
description: Self-review checklist, diff analysis patterns, quality gate verification, and common review findings for ensuring production-ready code before committing
when_to_apply: After implementation and testing are complete, before committing — perform a systematic self-review of all changes
version: 1.0.0
languages: Java, Kotlin
globs: "**/*.{java,kt}"
alwaysApply: false
---

# Code Review Standards

## Overview

Every change must be self-reviewed before committing. A self-review catches issues that automated tools miss — logic errors, missing edge cases, inconsistent patterns, security concerns, and unnecessary changes. This skill defines a systematic review process that mirrors what a senior engineer would check during a pull request review.

**Core principles:**
- Review every changed line against the ticket requirements
- Ensure consistency with existing code patterns and naming conventions
- Check for security vulnerabilities (OWASP Top 10) in every change
- Verify that tests cover both happy paths and error cases
- Keep PR scope focused on the ticket with no unrelated changes

## When to Apply This Skill

- After implementation and all tests pass, before creating a commit
- When reviewing someone else's pull request
- When verifying that changes meet ticket acceptance criteria

## Quick Reference

| Rule | Requirement | Level | Key Point |
|------|------------|-------|-----------|
| CR01 | Review every changed line | MUST | Read every add/modify/remove via git diff |
| CR02 | Verify acceptance criteria coverage | MUST | Map each criterion to code and tests |
| CR03 | Check for debug and temporary code | MUST | No println, TODOs, commented-out code, @Disabled |
| CR04 | Check consistency with existing code | MUST | Match error handling, DTO style, test structure |
| CR05 | Check naming consistency | MUST | Classes, methods, variables, tests, REST paths |
| CR06 | Check for common security issues | MUST | Injection, broken auth, sensitive data, mass assignment |
| CR07 | Verify no sensitive data in logs | MUST | No PII, tokens, passwords, or financial data in logs |
| CR08 | Verify layer boundaries | MUST | No controller logic in services, no repo in controllers |
| CR09 | Check for missing error handling | MUST | 404, 400, 409/422 paths handled for all operations |
| CR10 | Review data model changes | MUST | Backward compatibility, indexes, nullable fields |
| CR11 | Verify test quality | MUST | One behavior per test, Given-When-Then, descriptive names |
| CR12 | Check for missing test cases | MUST | Nulls, boundaries, duplicates, invalid transitions |
| CR13 | Verify PR scope | SHOULD | All changes relate to ticket, under 500 lines |
| CR14 | Check for unintended file changes | MUST | No IDE config, no unrelated files in diff |

---

## Review Process

### Rule CR01: Review Every Changed Line

**Requirement Level**: MUST

Run `git diff` and read every line that was added, modified, or removed. For each change, ask:
1. Does this change directly serve the ticket requirements?
2. Could this change break something that isn't covered by tests?
3. Is this the simplest way to achieve the goal?

### Rule CR02: Verify Acceptance Criteria Coverage

**Requirement Level**: MUST

Map each acceptance criterion from the ticket to specific code changes:

| Acceptance Criterion | Implemented In | Tested In |
|---------------------|---------------|-----------|
| "Users can create X" | `XService.create()` | `XServiceTest.test_create_*` |
| "Returns 404 for unknown ID" | `GlobalExceptionHandler` | `XIntegrationTest.test_get_notFound` |
| "Validates email format" | `CreateXRequest.@Email` | `XIntegrationTest.test_create_invalidEmail` |

If any criterion lacks both implementation and test coverage, the work is not complete.

---

## Code Quality Checks

### Rule CR03: Check for Debug and Temporary Code

**Requirement Level**: MUST

Search for and remove:
- `System.out.println` / `System.err.println`
- `e.printStackTrace()`
- Hardcoded test values in production code
- `// TODO` comments (track in tickets instead)
- Commented-out code blocks
- `@Disabled` / `@Ignore` test annotations added for convenience

### Rule CR04: Check for Consistency with Existing Code

**Requirement Level**: MUST

Compare new code against existing code in the same layer:
- Do new service methods follow the same error handling pattern?
- Do new DTOs use the same validation annotation style?
- Do new tests follow the same structure (Given-When-Then)?
- Do new controllers use the same response wrapping pattern?
- Are logging statements consistent in level and format?

If you deviate from existing patterns, there must be a clear reason.

### Rule CR05: Check Naming Consistency

**Requirement Level**: MUST

Verify all new names follow the project conventions:

| Element | Convention | Verify |
|---------|-----------|--------|
| Class names | Match existing pattern | `ClientService` not `ServiceForClients` |
| Method names | `camelCase`, verb-first | `findByStatus` not `statusFind` |
| Variables | Descriptive, no abbreviations | `contractNumber` not `cNum` |
| Test methods | `test_{method}_{scenario}_{result}` | Consistent with existing tests |
| REST paths | Plural, kebab-case | `/api/v1/audit-events` not `/api/v1/auditEvent` |

---

## Security Review

### Rule CR06: Check for Common Security Issues

**Requirement Level**: MUST

Review changes for OWASP Top 10 vulnerabilities:

| Vulnerability | Check For |
|--------------|-----------|
| **Injection** | Raw string concatenation in queries, unsanitized user input in commands |
| **Broken Auth** | Missing authentication checks on new endpoints |
| **Sensitive Data Exposure** | PII in logs, credentials in code, secrets in config |
| **Mass Assignment** | DTOs accepting fields that shouldn't be user-settable (e.g., `id`, `role`, `status`) |
| **SSRF** | User-controlled URLs passed to HTTP clients |
| **Insecure Deserialization** | Untrusted data deserialized without validation |

### Rule CR07: Verify No Sensitive Data in Logs

**Requirement Level**: MUST

Check every `log.*()` call added or modified:
- No email addresses, phone numbers, or names
- No tokens, passwords, or API keys
- No financial data
- Use identifiers (IDs) instead of PII

---

## Architecture Review

### Rule CR08: Verify Layer Boundaries

**Requirement Level**: MUST

Check that no layer violations were introduced:
- Controllers must not contain business logic
- Services must not import controller DTOs directly
- Repositories must not appear in controller imports
- No circular dependencies between packages

### Rule CR09: Check for Missing Error Handling

**Requirement Level**: MUST

For each new code path, verify:
- What happens if the entity doesn't exist? (404)
- What happens if validation fails? (400)
- What happens if a business rule is violated? (409 or 422)
- What happens if a unique constraint is violated? (409)
- Are all exceptions caught by the global exception handler?

### Rule CR10: Review Data Model Changes

**Requirement Level**: MUST (if applicable)

If entities were modified:
- Are existing documents in MongoDB compatible with the new schema?
- Are new fields nullable (backward compatible) or required (migration needed)?
- Are indexes added for new query fields?
- Are embedded objects kept simple (no deeply nested structures)?

---

## Test Review

### Rule CR11: Verify Test Quality

**Requirement Level**: MUST

For each test, check:
- Does it test one specific behavior (not multiple unrelated assertions)?
- Does it have a descriptive name that explains the scenario?
- Does it follow Given-When-Then with clear sections?
- Does it verify behavior (not implementation details)?
- Would it fail if the feature broke? (avoid tests that always pass)

### Rule CR12: Check for Missing Test Cases

**Requirement Level**: MUST

Common cases that are often missed:
- Null or empty inputs
- Boundary values (0, max, min)
- Duplicate creation
- Concurrent modification
- Invalid state transitions
- Permission/authorization checks
- List endpoint with 0, 1, and many items

---

## PR Scope Review

### Rule CR13: Verify PR Scope

**Requirement Level**: SHOULD

Check that the changes are appropriately scoped:
- All changes relate to the ticket (no drive-by refactoring)
- No unrelated formatting changes
- Total significant lines of change is under 500
- If over 500 lines, consider splitting into sequential PRs

### Rule CR14: Check for Unintended File Changes

**Requirement Level**: MUST

Run `git diff --stat` and verify:
- No changes to files unrelated to the ticket
- No accidental IDE config changes (`.idea/`, `.vscode/`)
- No build file changes unless necessary
- No changes to test data or fixtures unrelated to the ticket
- Lock files are updated if dependencies changed

---

## Enforcement Checklist

Complete self-review before committing:

**Correctness:**
- [ ] All acceptance criteria have implementation and test coverage
- [ ] No debug code, TODOs, or commented-out code
- [ ] Error paths handled for all new operations
- [ ] Data model changes are backward compatible

**Consistency:**
- [ ] Code follows existing patterns in the same layer
- [ ] Naming conventions match the project
- [ ] Logging is at appropriate levels with no PII

**Security:**
- [ ] No injection vulnerabilities
- [ ] No sensitive data in logs or responses
- [ ] New endpoints have proper validation
- [ ] No mass assignment vulnerabilities (DTOs don't expose internal fields)

**Tests:**
- [ ] Each public method has at least one test
- [ ] Error paths and edge cases covered
- [ ] Tests would fail if the feature broke
- [ ] Integration tests cover the API contract

**Scope:**
- [ ] All changes relate to the ticket
- [ ] No unintended file changes
- [ ] Under 500 significant lines (or justified)

## References

- For complete code examples, see [examples.md](examples.md)
- OWASP Top 10: https://owasp.org/www-project-top-ten/
- Google Code Review Guide: https://google.github.io/eng-practices/review/
