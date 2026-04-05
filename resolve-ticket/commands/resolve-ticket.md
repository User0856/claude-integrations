---
description: Analyze a ticket, implement the solution end-to-end, write tests, verify quality, and prepare a production-ready commit with PR
---

# Resolve Ticket End-to-End

Analyze a ticket (Jira, GitHub Issue, or plain text), implement the solution across all layers, write unit and integration tests, run quality checks, and prepare a production-ready commit — all following established coding standards and architectural patterns.

## Instructions

Follow these ten phases in strict order. Do not skip phases. If a phase fails, stop and report the failure with context before attempting recovery.

---

## Phase 1: Load Standards

Load all coding and quality standards by reading each skill's SKILL.md file. These skills define the rules you must follow throughout all subsequent phases.

**Skills to load (all required):**

| Skill | Purpose |
|-------|---------|
| `codebase-analysis` | Project structure analysis and pattern recognition |
| `architecture-design` | Layered architecture, DDD, and Spring Boot patterns |
| `implementation` | Coding standards, naming conventions, and best practices |
| `unit-testing` | Unit test patterns with JUnit 5 and Mockito |
| `integration-testing` | Integration test patterns with Testcontainers |
| `code-quality` | Compilation, linting, static analysis, and formatting |
| `code-review` | Self-review checklist and quality gates |
| `vcs` | Git workflow, branch naming, commit messages, and PR standards |

**Loading procedure — try each location in order until found:**

For each skill listed above, read the SKILL.md file from the first location that exists:
1. `.claude/skills/<skill-name>/SKILL.md` (project-level)
2. `~/.claude/skills/<skill-name>/SKILL.md` (user-level)

For skills that also have an `examples.md` file in the same directory, read that too.

Load all 8 skills before proceeding to Phase 2. If any skill cannot be found, continue with the remaining skills — the command can still function with partial standards.

---

## Phase 2: Ticket Analysis

Parse and understand the ticket from `$ARGUMENTS`.

### Input Handling

The argument can be one of:
1. **Jira ticket ID** (e.g., `CMS-42`) — Fetch via Atlassian MCP: `mcp__atlassian__jira_get_issue`
2. **GitHub Issue URL** (e.g., `https://github.com/org/repo/issues/42`) — Fetch via `gh issue view`
3. **Plain text description** — Use as-is

### Extract Requirements

From the ticket, extract and document:
- **Summary**: One-line description of what needs to be done
- **Type**: Bug fix, new feature, enhancement, refactoring, test gap, or chore
- **Acceptance criteria**: Explicit conditions that must be met
- **Scope boundaries**: What is explicitly in scope and out of scope
- **Linked context**: Related tickets, Confluence pages, design docs, or PRs
- **Priority and constraints**: Deadlines, performance requirements, backward compatibility

### Derive Identifiers

- **Ticket ID** (uppercase for commits/PRs): e.g., `CMS-42`
- **Branch slug** (lowercase): e.g., `feature/cms-42`

Present a brief summary of your understanding and proceed immediately.

---

## Phase 3: Codebase Analysis

Before writing any code, thoroughly understand the project.

### 3.1 Project Structure Scan

Using the `codebase-analysis` skill:
1. Identify the build system (Gradle/Maven), language version, and framework
2. Map the package structure and architectural layers
3. Find configuration files (`application.yml`, `docker-compose.yml`, etc.)
4. Identify existing test infrastructure (base classes, test utilities, test containers config)

### 3.2 Relevant Code Discovery

1. Identify files directly related to the ticket (entities, services, controllers, repositories)
2. Read each file completely — understand existing patterns before modifying
3. Find similar implementations in the codebase to follow as templates
4. Identify shared utilities, base classes, and common patterns
5. Check for existing tests that cover adjacent functionality

### 3.3 Impact Assessment

1. List all files that will need to be created or modified
2. Identify downstream dependencies (other services, controllers, or tests that reference affected code)
3. Check for any configuration changes needed
4. Estimate the total lines of change — if exceeding 500 lines, flag for potential PR splitting

---

## Phase 4: Architecture and Design

Apply the `architecture-design` skill.

### 4.1 Design Decisions

For each component to be created or modified:
1. Determine the correct architectural layer (Controller → Service → Repository)
2. Verify the design follows established patterns in the codebase
3. Ensure proper separation of concerns — no business logic in controllers, no data access in services
4. Plan the data flow: request DTO → service method → domain model → repository → response DTO

### 4.2 API Design (if applicable)

For new or modified endpoints:
1. Follow RESTful conventions under `/api/v1/`
2. Use proper HTTP methods and status codes
3. Design request/response DTOs with Jakarta validation annotations
4. Plan error responses using the project's exception handling pattern

### 4.3 Domain Model Changes (if applicable)

1. Design entity changes following existing MongoDB `@Document` patterns
2. Plan embedded value objects vs. separate collections
3. Consider audit trail implications
4. Design any new domain events

Proceed to branch setup after design is clear. Do not ask for confirmation.

---

## Phase 5: Branch Setup

**Do NOT create or modify any source files until the feature branch exists and is checked out.**

Apply the `vcs` skill for branch naming conventions.

### 5.1 Safeguard Uncommitted Work

```bash
# Check for uncommitted changes — stash before switching branches
if ! git diff --quiet || ! git diff --cached --quiet; then
    echo "Uncommitted changes detected. Stashing before branch creation."
    git stash push -m "auto-stash before resolve-ticket"
fi
```

### 5.2 Update Default Branch

```bash
# Detect the default branch (main, master, or develop)
DEFAULT_BRANCH=$(git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@')
if [ -z "$DEFAULT_BRANCH" ]; then
    DEFAULT_BRANCH="main"
fi

# Fetch latest and update local default branch
git fetch origin
git checkout "$DEFAULT_BRANCH"
git pull origin "$DEFAULT_BRANCH"
```

### 5.3 Create Feature Branch

Branch name follows the `vcs` skill convention: `feature/<ticket-id-lowercase>`

```bash
# Create and checkout the feature branch from the up-to-date default branch
git checkout -b "feature/<ticket-id-lowercase>" "$DEFAULT_BRANCH"
```

If the branch already exists locally, check it out and rebase onto the latest default branch:

```bash
git checkout "feature/<ticket-id-lowercase>"
git rebase "$DEFAULT_BRANCH"
```

### 5.4 Verify

```bash
# Confirm you are on the correct branch before proceeding
git branch --show-current
# Expected: feature/<ticket-id-lowercase>
```

---

## Phase 6: Implementation

Apply the `implementation` skill throughout.

### 6.1 Implement Changes

**Detect the build tool and stack first** — check whether `build.gradle` / `build.gradle.kts` (Gradle) or `pom.xml` (Maven) exists. Adapt all build commands in subsequent phases accordingly.

Follow this order strictly:
1. **Domain model** — Entities, enums, value objects, domain events, exceptions
2. **Repository layer** — Repository interfaces and custom queries
3. **Service layer** — Business logic, validation, orchestration
4. **DTO layer** — Request/response DTOs, MapStruct mappers
5. **Controller layer** — REST endpoints, request validation, response mapping
6. **Configuration** — Any new beans, properties, or config changes

### Implementation Rules

- **Read at least one existing file of the same type** before creating a new one (e.g., read an existing `@Service` class before writing a new service). Match its style, imports, annotations, and patterns exactly.
- Follow existing code patterns — match style, naming, structure
- Use Lombok annotations consistent with the project (`@Data`, `@Builder`, `@RequiredArgsConstructor`)
- Apply Jakarta validation on all request DTOs (`@NotNull`, `@NotBlank`, `@Size`, etc.)
- Use constructor injection (never field injection)
- Handle errors using the project's established exception hierarchy
- Add `@Slf4j` logging at service boundaries with meaningful messages
- Services must return domain objects, not DTOs — mapping to DTOs happens in the controller layer
- Never introduce new dependencies without explicit justification

---

## Phase 7: Unit Tests

Apply the `unit-testing` skill.

### 7.1 Write Unit Tests

For every new or modified service/component:
1. Create test class following `{ClassName}Test` naming convention
2. Use `@ExtendWith(MockitoExtension.class)`
3. Mock all dependencies with `@Mock`, inject with `@InjectMocks`
4. Follow Given-When-Then structure with comments
5. Name tests: `test_{methodName}_{scenario}_{expectedResult}`

### 7.2 Coverage Requirements

- All public methods must have at least one test
- Test happy path, error paths, and edge cases
- Test validation logic and business rules
- Test state transitions and conditional branches
- Verify interactions with mocked dependencies using `verify()`

### 7.3 Run Unit Tests

```bash
# Gradle
./gradlew test --info 2>&1

# Maven (if pom.xml exists instead of build.gradle)
# mvn test 2>&1
```

If tests fail:
1. Read the failure output carefully
2. Fix the root cause (not the symptom)
3. Re-run tests
4. Maximum 3 fix-and-retry cycles — if still failing after 3 attempts, stop and report

---

## Phase 8: Integration Tests

Apply the `integration-testing` skill. Read both `SKILL.md` AND `examples.md` for this skill — the examples contain exact working code patterns that MUST be followed.

### 8.1 Evaluate Need

Integration tests are REQUIRED when:
- New REST endpoints are added or modified
- Database queries or repository methods change
- Multiple services interact in a workflow
- Configuration or wiring changes affect runtime behavior

Integration tests are OPTIONAL when:
- Changes are purely to internal service logic already covered by unit tests
- Only DTOs or mappers changed with no behavioral impact

**If the ticket explicitly states that integration tests are not needed, skip this phase entirely.**

### 8.2 Check for Existing Base Class

Before writing integration tests, check if a `BaseIntegrationTest` class already exists:
```bash
find src/integrationTest -name "BaseIntegrationTest.java" 2>/dev/null
```

If it exists, read it completely and extend it. **Do NOT create a new base class or modify the existing one.** If it does not exist, create one following the exact pattern in the `integration-testing` skill.

### 8.3 Write Integration Tests

If integration tests are needed:
1. Create test class in `src/integrationTest/java/...integration/` following `{Feature}IntegrationTest` naming
2. Extend `BaseIntegrationTest`
3. **CRITICAL: Use raw JSON strings (text blocks) for request bodies** — do NOT use `ObjectMapper.writeValueAsString()` for DTOs with boolean primitives. See the `integration-testing` skill for details.
4. Use `jsonPath()` with Hamcrest matchers for response verification
5. For Lombok `boolean isPrimary` fields, use `$.primary` in jsonPath (Jackson strips "is" prefix)
6. Write helper methods for creating prerequisite entities (clients, contacts, etc.)
7. Test success paths, validation errors (400), not-found (404), and business rule violations (422)

### 8.4 Run Integration Tests

```bash
# Gradle — run the integrationTest task
./gradlew integrationTest --info 2>&1

# If integrationTest task doesn't exist, check build.gradle or run:
# ./gradlew tasks --group=verification
```

Apply the same fix-and-retry logic as unit tests (max 3 cycles).

---

## Phase 9: Quality Verification

Apply the `code-quality` and `code-review` skills.

### 9.1 Compilation Check

```bash
# Gradle
./gradlew compileJava compileTestJava 2>&1

# Maven
# mvn compile test-compile 2>&1
```

Fix any compilation errors before proceeding.

### 9.2 Full Test Suite

Run the complete test suite to verify no regressions:

```bash
# Gradle
./gradlew test integrationTest 2>&1

# Maven
# mvn verify 2>&1
```

### 9.3 Static Analysis (if configured)

```bash
# Gradle — only run if these plugins are configured in build.gradle
./gradlew checkstyleMain spotbugsMain 2>&1

# Maven
# mvn checkstyle:check spotbugs:check 2>&1
```

Fix any violations. If the project does not have these plugins configured, skip this step.

### 9.4 Self-Review

Before committing, perform a self-review using the `code-review` skill:

1. Run `git diff` and review every changed line
2. Verify each change maps to a ticket requirement
3. Check for:
   - Accidental debug code, TODOs, or commented-out code
   - Missing validation or error handling at system boundaries
   - Inconsistent naming or style compared to surrounding code
   - Missing or incorrect test assertions
   - Any security concerns (SQL injection, XSS, mass assignment)
   - Proper logging (no sensitive data, meaningful messages)
4. Verify all acceptance criteria are met
5. If changes exceed 500 lines, evaluate whether the PR should be split

---

## Phase 10: Commit and Prepare PR

Apply the `vcs` skill.

### 10.1 Stage Changes

```bash
# Review what will be committed
git status
git diff --stat

# Stage changes — add specific files rather than using `git add -A`
# This avoids accidentally staging sensitive files (.env, credentials, IDE config)
git add <list of specific files that were created or modified>
```

**Important:** Review `git status` output before staging. Do NOT stage files like `.env`, `credentials.json`, `.idea/`, `.vscode/`, or any files containing secrets. Stage only the source files, test files, and configuration files you created or modified for this ticket.

### 10.2 Create Commit

Commit message format: `<TICKET-ID>: <descriptive message>`

The message must:
- Start with the ticket ID in uppercase
- Describe WHAT was done and WHY (not HOW)
- Be a single line under 72 characters for the subject
- Optionally include a body with bullet points for complex changes

```bash
git commit -m "<TICKET-ID>: <descriptive message>

- List key changes as bullet points
- Include any notable design decisions
- Mention test coverage additions"
```

### 10.3 Push and Create PR

```bash
# Push the feature branch
git push -u origin "feature/<ticket-id-lowercase>"

# Verify gh CLI is available before attempting PR creation
if ! command -v gh &>/dev/null; then
    echo "GitHub CLI (gh) is not installed. Push completed."
    echo "Create a PR manually at the URL shown above."
    exit 0
fi

# Create PR
gh pr create \
  --title "<TICKET-ID>: <descriptive message>" \
  --body "$(cat <<'EOF'
## Summary

<Brief description of what this PR does and why>

## Ticket

<Link to Jira ticket or issue>

## Changes

- <Key change 1>
- <Key change 2>
- <Key change 3>

## Testing

- [x] Unit tests added/updated
- [x] Integration tests added/updated (if applicable)
- [x] All existing tests pass
- [x] Manual verification performed

## Checklist

- [x] Code follows project conventions and patterns
- [x] No merge conflicts with main
- [x] Self-review completed
- [x] No debug code, TODOs, or commented-out code

Generated with Claude Code
EOF
)"
```

---

## Output

When complete, present:

1. **Summary** — What was implemented and why
2. **Files changed** — List of all created/modified files with brief descriptions
3. **Test results** — Unit test count (passed/failed), integration test count (passed/failed)
4. **Quality checks** — Compilation, static analysis, and self-review results
5. **PR link** — URL of the created pull request
6. **Commit** — The commit hash and message

---

## Error Handling

- **Ticket fetch fails**: If Jira MCP is unavailable, ask the user to paste the ticket description
- **Tests fail after 3 retries**: Stop, report the failing tests with full output, and ask for guidance
- **Build fails**: Report the exact compilation error with file and line number
- **Branch already exists**: Check out the existing branch and continue from where it left off
- **Push fails**: Report the error — never force-push without explicit user instruction
- **Static analysis violations**: Fix automatically if clear, otherwise report and ask
- **`gh` CLI not installed**: Complete push but skip PR creation; display the remote URL for manual PR creation
- **Uncommitted changes on current branch**: Stash changes automatically before creating the feature branch
- **`integrationTest` task not found**: Check `build.gradle` for the correct task name; fall back to `./gradlew test` if integration tests are included there
- **Non-standard default branch**: Detect `main`, `master`, or `develop` automatically via `git symbolic-ref refs/remotes/origin/HEAD`

---

## Notes

- This command runs fully autonomously — no confirmation prompts between phases
- All standards are loaded upfront (Phase 1) and applied throughout
- The implementation follows whatever patterns already exist in the target codebase
- If the codebase uses conventions different from the loaded skills, prefer the codebase conventions
- For changes exceeding 500 lines, suggest splitting into multiple PRs but proceed with one unless instructed otherwise
