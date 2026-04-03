---
description: Perform a comprehensive, multi-dimensional review of a pull request against all coding standards, security rules, and architectural patterns
---

# Review Pull Request

Perform a thorough, structured review of a pull request — analyzing every changed file against all loaded coding standards, checking for security vulnerabilities, verifying test coverage, assessing architectural alignment, and producing a prioritized findings report with actionable fixes.

## Instructions

Follow these ten phases in strict order. Do not skip phases. If a phase fails, stop and report the failure with context before attempting recovery.

---

## Phase 1: Load Standards

Load all coding and quality standards by reading each skill's SKILL.md file. These skills define the rules you will review the PR against.

**Skills to load (all required):**

| Skill | Review Purpose |
|-------|---------------|
| `codebase-analysis` | Verify the PR follows existing project patterns |
| `architecture-design` | Check layer boundaries, API design, data model |
| `implementation` | Validate naming, Lombok usage, validation, logging |
| `unit-testing` | Assess unit test quality, structure, coverage |
| `integration-testing` | Assess integration test quality and completeness |
| `code-quality` | Check compilation, formatting, static analysis |
| `code-review` | Apply the full self-review checklist |
| `vcs` | Verify branch naming, commit messages, PR format |

**Loading procedure — try each location in order until found:**

For each skill listed above, read the SKILL.md file from the first location that exists:
1. `.claude/skills/<skill-name>/SKILL.md` (project-level)
2. `~/.claude/skills/<skill-name>/SKILL.md` (user-level)

For skills that also have an `examples.md` file in the same directory, read that too — examples establish the expected quality bar.

Load all 8 skills before proceeding to Phase 2. If any skill cannot be found, continue with the remaining skills.

---

## Phase 2: PR Discovery

Parse and identify the pull request from `$ARGUMENTS`.

### Input Handling

The argument can be one of:
1. **PR number** (e.g., `42`) — Fetch via `gh pr view 42`
2. **PR URL** (e.g., `https://github.com/org/repo/pull/42`) — Extract number, fetch via `gh pr view`
3. **Branch name** (e.g., `feature/cms-42`) — Find the PR via `gh pr list --head <branch>`
4. **No argument** — Review the PR for the current branch via `gh pr view`

### Extract PR Context

```bash
# Fetch PR metadata
gh pr view <number> --json title,body,author,baseRefName,headRefName,additions,deletions,changedFiles,commits,labels,reviewDecision

# Fetch the list of changed files
gh pr diff <number> --name-only

# Fetch the full diff
gh pr diff <number>
```

From the PR metadata, extract and document:
- **Title and description**: What the author says the PR does
- **Ticket reference**: Jira ID or GitHub Issue link from the title/body
- **Base branch**: The branch being merged into (main, develop, etc.)
- **Scope metrics**: Files changed, lines added, lines deleted
- **Commit count and messages**: All commits in the PR

Present a brief summary of the PR scope and proceed.

---

## Phase 3: Change Scope Analysis

Understand the shape and risk profile of the changes before diving into details.

### 3.1 Categorize Changed Files

Group every changed file into categories:

| Category | File Patterns | Risk Level |
|----------|--------------|------------|
| Domain Model | `**/model/**`, `**/entity/**`, `**/domain/**` | High — schema changes affect storage and compatibility |
| Repository | `**/repository/**` | High — query changes affect data integrity |
| Service | `**/service/**` | High — business logic changes affect core behavior |
| Controller | `**/controller/**` | Medium — API contract changes affect consumers |
| DTO | `**/dto/**`, `**/request/**`, `**/response/**` | Medium — affects API contract |
| Configuration | `**/config/**`, `application*.yml`, `build.gradle*` | Medium — affects runtime behavior |
| Exception | `**/exception/**` | Low — typically additive |
| Unit Test | `**/test/**/*Test.java` | Low — tests are safety nets |
| Integration Test | `**/integrationTest/**`, `**/it/**` | Low — tests are safety nets |
| Other | Everything else | Assess individually |

### 3.2 Compute Risk Metrics

Assess the PR against these thresholds:

| Metric | Green | Yellow | Red |
|--------|-------|--------|-----|
| Total lines changed | < 200 | 200–500 | > 500 |
| Files changed | < 10 | 10–20 | > 20 |
| Domain model files changed | 0–1 | 2–3 | > 3 |
| New dependencies added | 0 | 1 | > 1 |
| Config files changed | 0 | 1 | > 1 |

Flag any Red metrics as risk items in the final report.

### 3.3 Identify Missing Files

Based on the changes, check whether expected companion files are present:
- New service → expect a unit test
- New controller → expect an integration test
- New entity → expect repository, service, DTOs, controller
- New endpoint → expect request validation DTOs

Flag missing companions as findings.

---

## Phase 4: Codebase Context

Understand the project's existing patterns so you can judge whether the PR conforms to them.

### 4.1 Project Structure Scan

Using the `codebase-analysis` skill:
1. Identify the build system (Gradle/Maven), language version, and framework
2. Map the package structure and architectural layers
3. Identify the exception handling pattern (GlobalExceptionHandler, custom exceptions)
4. Identify the DTO pattern (separate Create/Update/Response DTOs, or shared)
5. Identify the test infrastructure (base classes, test utilities, builders)

### 4.2 Establish Baselines

For each type of file changed in the PR, find an existing example of the same type that was NOT changed:
- If a new service was added → read an existing service
- If a new test was added → read an existing test
- If a controller was modified → read another controller

These baselines are the "expected pattern." Deviations in the PR are potential findings.

### 4.3 Check Existing Conventions

Document the conventions you observe:
- Naming patterns (class, method, variable, test)
- Annotation usage (Lombok, validation, Spring)
- Import ordering
- Logging patterns (levels, format, what gets logged)
- Error handling patterns (exception types, how they map to HTTP status codes)

---

## Phase 5: File-Level Review

Read and review every changed file in full — not just the diff, but the entire file — to understand changes in their full context.

### 5.1 Read Complete Files

For every file that was modified (not just added):
1. Read the complete current file content
2. Read the diff for that file
3. Understand what existed before and what changed

For newly added files:
1. Read the complete file
2. Compare structure against the baseline file from Phase 4

### 5.2 Per-File Checks

For each file, check:

**All files:**
- Does it follow the project's naming conventions?
- Are imports ordered consistently with the rest of the codebase?
- Is formatting consistent (indentation, brace style, blank lines)?
- Are there any TODO comments, debug statements, or commented-out code?

**Source files:**
- Does the class/method structure match existing patterns?
- Are annotations used consistently with the baseline?
- Is error handling consistent with the project's exception hierarchy?
- Are there any hardcoded values that should be configurable?

**Test files:**
- Does each test follow Given-When-Then structure?
- Does the test name follow `test_{method}_{scenario}_{result}`?
- Are assertions meaningful (not just `assertNotNull`)?
- Would the test actually fail if the feature broke?

Document findings per file with exact line references.

---

## Phase 6: Parallel Skill Reviews

Launch parallel review agents — one per skill category — to systematically review the diff against every rule.

### 6.1 Prepare the Diff

```bash
# Get the full diff content to pass to each agent
gh pr diff <number>
```

### 6.2 Launch Review Agents

Launch 7 parallel agents using the Agent tool (model: sonnet). Each agent receives the full diff and reviews it against one skill's rules.

**Agent template for each skill:**

```
You are a code reviewer. Review this pull request diff against the {SKILL_NAME} skill rules.

## Skill Rules
{Content of the skill's SKILL.md}

## PR Diff
{Full diff content}

## Instructions
1. Check every changed line against each rule in the skill
2. For each violation found, report:
   - **Rule**: Rule ID and name (e.g., CR01: Review Every Changed Line)
   - **Level**: MUST / SHOULD / MAY
   - **File**: Exact file path
   - **Line**: Line number(s) in the diff
   - **Finding**: What is wrong
   - **Current**: What the code currently does
   - **Expected**: What it should do per the rule
   - **Fix**: Specific code change to resolve
3. For each rule, indicate: VIOLATED / SATISFIED / NOT APPLICABLE
4. Be strict on MUST rules, pragmatic on SHOULD rules
5. Do not report false positives — only flag genuine violations

Return findings as a structured list.
```

**Agents to launch (in parallel):**

| Agent | Skill | Focus |
|-------|-------|-------|
| 1 | `architecture-design` | Layer boundaries, API patterns, data model design |
| 2 | `implementation` | Naming, annotations, validation, logging, idioms |
| 3 | `unit-testing` | Test structure, coverage, mocking, assertions |
| 4 | `integration-testing` | Spring Boot Test setup, Testcontainers, API testing |
| 5 | `code-quality` | Compilation, formatting, static analysis concerns |
| 6 | `code-review` | The full self-review checklist (CR01–CR14) |
| 7 | `vcs` | Branch naming, commit messages, PR format, scope |

The `codebase-analysis` skill is not reviewed in parallel — it was already applied in Phase 4 to establish baselines.

### 6.3 Collect Results

Wait for all 7 agents to complete. Collect their findings into a single list.

---

## Phase 7: Security Review

Perform a dedicated security review beyond what the `code-review` skill's CR06/CR07 rules cover. This phase looks at the full changed codebase, not just individual rules.

### 7.1 OWASP Top 10 Scan

For each vulnerability category, actively search the changed code:

| Vulnerability | What to Search For |
|--------------|-------------------|
| **Injection** | String concatenation in queries (`"SELECT " + input`), unsanitized input in shell commands, template injection |
| **Broken Authentication** | Missing `@PreAuthorize` or security annotations on new endpoints, hardcoded credentials, JWT mishandling |
| **Sensitive Data Exposure** | PII in logs, API responses returning internal fields (IDs, timestamps, internal status), error messages leaking stack traces |
| **Mass Assignment** | Request DTOs that bind to `id`, `role`, `status`, `createdBy`, or audit fields — any field the user should not control |
| **SSRF** | User-provided URLs passed to `RestTemplate`, `WebClient`, or HTTP clients without validation |
| **Insecure Deserialization** | `ObjectMapper.readValue()` on untrusted input without type restrictions |
| **Insufficient Logging** | Security-relevant operations (login, access control, data modification) without audit logging |

### 7.2 Dependency Review

If `build.gradle`, `build.gradle.kts`, or `pom.xml` was changed:
1. Identify any new dependencies added
2. Check if the dependency is well-maintained and widely used
3. Flag any dependency that handles security-sensitive operations (crypto, auth, serialization)
4. Verify the dependency scope is correct (test dependencies not leaking to compile)

### 7.3 Configuration Review

If `application.yml`, `application.properties`, or config classes changed:
1. Check for hardcoded secrets, API keys, or passwords
2. Verify CORS configuration is not overly permissive (`allowedOrigins: *`)
3. Check that debug/dev settings are not enabled in production profiles
4. Verify sensitive properties use environment variables or secret management

Document all security findings with severity: Critical, High, Medium, Low.

---

## Phase 8: Test Adequacy Review

Assess whether the tests in the PR are sufficient — not just present, but effective.

### 8.1 Coverage Mapping

Build a coverage map for every changed source file:

| Source File | Changed Methods | Unit Test | Integration Test | Coverage Gap |
|------------|----------------|-----------|-----------------|--------------|
| `ClientService.java` | `create()`, `findById()` | `ClientServiceTest` | `ClientControllerIntegrationTest` | None |
| `InvoiceService.java` | `calculateTotal()` | — | — | **Missing both** |

Flag any changed method that lacks a corresponding test.

### 8.2 Test Quality Assessment

For each test file in the PR, evaluate:

| Quality Dimension | Check | Pass/Fail |
|-------------------|-------|-----------|
| **Isolation** | Does each test verify one behavior? | |
| **Naming** | Does the name describe scenario and expected result? | |
| **Structure** | Does it follow Given-When-Then with comments? | |
| **Assertions** | Are assertions specific (not just `assertNotNull`)? | |
| **Failure signal** | Would it fail if the feature broke? | |
| **Edge cases** | Are nulls, empty, boundary values tested? | |
| **Error paths** | Are exceptions and error responses tested? | |
| **Mocking** | Are mocks minimal (only external dependencies)? | |

### 8.3 Missing Test Scenarios

For each new or modified feature, check whether these common scenarios are covered:

- **Happy path** — standard successful operation
- **Not found** — entity doesn't exist (expect 404)
- **Validation failure** — invalid input (expect 400)
- **Duplicate** — creating something that already exists (expect 409)
- **Invalid state transition** — if state machine logic exists
- **Empty collection** — list endpoint returns 0 items
- **Boundary values** — min, max, zero, negative where applicable
- **Null inputs** — required fields set to null

Flag missing scenarios as findings.

---

## Phase 9: Findings Synthesis

Aggregate all findings from Phases 5–8, deduplicate, and prioritize.

### 9.1 Deduplicate

Multiple agents may flag the same issue (e.g., Phase 5 file review and Phase 6 code-review agent both catch a TODO comment). Merge duplicates, keeping the most specific description and fix.

### 9.2 Categorize by Severity

Assign each finding a severity based on the rule level and impact:

| Severity | Criteria | Action Required |
|----------|----------|----------------|
| **Critical** | MUST-level rule violation, security vulnerability, data loss risk, broken functionality | Must fix before merge |
| **High** | MUST-level rule violation with lower impact, missing test coverage for critical paths | Should fix before merge |
| **Medium** | SHOULD-level rule violation, code quality concerns, missing edge case tests | Recommended to fix |
| **Low** | MAY-level suggestions, style preferences, minor improvements | Optional |

### 9.3 Group by Category

Organize findings into categories for the report:

1. **Security** — OWASP violations, sensitive data exposure, auth gaps
2. **Architecture** — Layer violations, API design issues, data model concerns
3. **Implementation** — Naming, conventions, coding patterns, error handling
4. **Testing** — Missing tests, weak assertions, coverage gaps
5. **Quality** — Formatting, imports, dead code, TODOs
6. **VCS** — Branch naming, commit messages, PR format, scope

### 9.4 Prioritize Action Items

Create a prioritized action list:
1. Critical findings first (must fix)
2. High findings second (should fix)
3. Medium findings grouped by effort (quick wins first)
4. Low findings at the end (nice to have)

---

## Phase 10: Report

Generate a structured review report and post it as a PR comment.

### 10.1 Report Format

```markdown
## PR Review: <PR Title>

### Summary

| Category | Critical | High | Medium | Low | Total |
|----------|----------|------|--------|-----|-------|
| Security | 0 | 0 | 0 | 0 | 0 |
| Architecture | 0 | 0 | 0 | 0 | 0 |
| Implementation | 0 | 0 | 0 | 0 | 0 |
| Testing | 0 | 0 | 0 | 0 | 0 |
| Quality | 0 | 0 | 0 | 0 | 0 |
| VCS | 0 | 0 | 0 | 0 | 0 |
| **Total** | **0** | **0** | **0** | **0** | **0** |

### Verdict

<One of: APPROVE / REQUEST CHANGES / COMMENT>

<1-3 sentence summary of the overall assessment>

### Risk Assessment

- **Scope**: <lines changed, files changed>
- **Risk areas**: <list of high-risk file categories touched>
- **Test coverage**: <adequate / gaps identified>
- **Security**: <no concerns / concerns found>

---

### Critical Findings (Must Fix)

#### 1. <Finding Title>

- **Rule**: <Rule ID — Rule Name>
- **File**: `<path/to/file.java>:<line>`
- **Finding**: <What is wrong>
- **Current**: <What the code currently does>
- **Expected**: <What it should do>
- **Fix**:
\```java
// suggested fix
\```

---

### High Findings (Should Fix)

...

### Medium Findings (Recommended)

...

### Low Findings (Optional)

...

---

### Test Coverage Map

| Source File | Changed Methods | Unit Test | Integration Test | Status |
|------------|----------------|-----------|-----------------|--------|
| ... | ... | ... | ... | ✅ / ❌ |

### Positive Observations

- <Things the PR does well — acknowledge good patterns, thorough tests, clean design>

---

Reviewed against: codebase-analysis, architecture-design, implementation, unit-testing, integration-testing, code-quality, code-review, vcs

Generated with Claude Code
```

### 10.2 Determine Verdict

Based on findings:
- **APPROVE** — No Critical or High findings. Medium/Low findings are minor.
- **REQUEST CHANGES** — Any Critical findings, or 3+ High findings.
- **COMMENT** — 1-2 High findings, or significant Medium findings that warrant discussion.

### 10.3 Post as PR Comment

```bash
# Verify gh CLI is available
if ! command -v gh &>/dev/null; then
    echo "GitHub CLI (gh) is not installed. Report printed above."
    echo "Copy and paste it as a PR comment manually."
    exit 0
fi

# Post the review report as a PR comment
gh pr comment <number> --body "$(cat <<'EOF'
<full report content>
EOF
)"
```

### 10.4 Submit Review Decision

If the verdict is APPROVE or REQUEST CHANGES, also submit a formal GitHub review:

```bash
# APPROVE
gh pr review <number> --approve --body "All checks passed. See detailed review in comments."

# REQUEST CHANGES
gh pr review <number> --request-changes --body "Found critical issues that must be addressed. See detailed review in comments."
```

---

## Output

When complete, present:

1. **Verdict** — APPROVE, REQUEST CHANGES, or COMMENT
2. **Summary** — One-line assessment of the PR
3. **Findings count** — Critical / High / Medium / Low breakdown
4. **Top issues** — The most important findings to address first
5. **PR comment link** — URL of the posted review comment

---

## Error Handling

- **PR not found**: Verify the PR number/URL and check that `gh` is authenticated with the correct repo
- **No changed files**: Report that the PR has no diff and cannot be reviewed
- **Agent failures**: If a parallel review agent fails, continue with the remaining agents and note partial coverage
- **`gh` CLI not installed**: Print the full report to the console instead of posting as a comment
- **No remote configured**: Print the report to console — the review is still valuable locally
- **PR already merged**: Warn that the PR is merged but proceed with the review anyway (useful for post-merge audits)
- **Large PR (>1000 lines)**: Warn about scope but proceed — flag PR size as a finding under VCS rules

---

## Notes

- This command runs fully autonomously — no confirmation prompts between phases
- All standards are loaded upfront (Phase 1) and applied throughout
- The review compares PR changes against the target project's existing conventions, not just abstract rules
- Positive observations are included — good code review acknowledges what was done well, not just what needs fixing
- Security review (Phase 7) is always performed regardless of the type of changes
- The report is posted as a PR comment so it becomes part of the PR history
- For post-merge audits, run with the PR number of an already-merged PR
