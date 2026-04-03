---
name: Code Quality Standards
description: Compilation checks, static analysis (Checkstyle, SpotBugs), code formatting, dependency hygiene, and build verification gates
when_to_apply: After implementing changes and writing tests — verify that the code compiles, passes static analysis, and meets quality gates before committing
version: 1.0.0
languages: Java, Kotlin
globs: "**/*.{java,kt,gradle,xml}"
alwaysApply: false
---

# Code Quality Standards

## Overview

Quality gates catch problems that tests miss — style violations, potential bugs, unused imports, and formatting inconsistencies. This skill defines the checks that must pass before any code is committed. These checks are typically automated in CI, so running them locally first avoids wasted CI cycles and blocked PRs.

**Core principles:**
- Code must compile cleanly before committing
- Static analysis violations must be fixed, not suppressed
- Consistent formatting is enforced by automated tools
- The full build (compile, test, analyze) must pass before every commit
- New dependencies must be justified, stable, and scoped correctly

## When to Apply This Skill

- After completing implementation and tests, before committing
- When reviewing code changes for quality compliance
- When configuring or troubleshooting static analysis tools
- When resolving CI quality gate failures

## Quick Reference

| Rule | Requirement | Level | Key Point |
|------|------------|-------|-----------|
| CQ01 | Code must compile without warnings | MUST | compileJava and compileTestJava must succeed |
| CQ02 | Resolve deprecation warnings | SHOULD | Update to non-deprecated API or document follow-up |
| CQ03 | Checkstyle must pass | MUST | checkstyleMain checkstyleTest (if configured) |
| CQ04 | SpotBugs must pass | MUST | spotbugsMain (if configured) |
| CQ05 | Fix violations, don't suppress | MUST | No inline @SuppressFBWarnings or CHECKSTYLE:OFF |
| CQ06 | Follow consistent formatting | MUST | Use project formatter (Spotless, google-java-format) |
| CQ07 | Import order | SHOULD | java, javax/jakarta, third-party, project, static |
| CQ08 | Full build must pass | MUST | ./gradlew build or mvn clean verify |
| CQ09 | No new test failures | MUST | All tests must pass after changes |
| CQ10 | Dependency hygiene | SHOULD | Stable versions, correct scope, no duplicates |
| CQ11 | Pre-commit checklist | MUST | Compile, test, analyze, review diff before commit |

---

## Compilation Verification

### Rule CQ01: Code Must Compile Without Warnings

**Requirement Level**: MUST

Before committing, verify that all source and test code compiles:

```bash
# Gradle
./gradlew compileJava compileTestJava

# Maven
mvn compile test-compile
```

Address ALL compilation errors before proceeding. Common issues:
- Missing imports
- Type mismatches in mapper interfaces (MapStruct)
- Incorrect method signatures after refactoring
- Missing dependency declarations in build file

### Rule CQ02: Resolve All Deprecation Warnings

**Requirement Level**: SHOULD

If the build reports deprecation warnings, either:
1. Update to the non-deprecated API
2. If migration is non-trivial, document it as a separate follow-up task

Never suppress warnings with `@SuppressWarnings("deprecation")` unless there is a documented reason.

---

## Static Analysis

### Rule CQ03: Checkstyle Must Pass

**Requirement Level**: MUST (if configured in the project)

```bash
./gradlew checkstyleMain checkstyleTest
```

Common Checkstyle rules and their fixes:

| Violation | Fix |
|-----------|-----|
| `MissingJavadoc` on public API | Add Javadoc or reduce visibility |
| `LineLength > 120` | Break long lines at logical points |
| `UnusedImports` | Remove unused imports |
| `WhitespaceAround` | Fix spacing around operators and braces |
| `NeedBraces` | Always use braces for `if`, `for`, `while` |
| `MethodName` | Follow camelCase naming |
| `ConstantName` | Use UPPER_SNAKE_CASE for `static final` |

### Rule CQ04: SpotBugs Must Pass

**Requirement Level**: MUST (if configured in the project)

```bash
./gradlew spotbugsMain
```

Common SpotBugs findings and their fixes:

| Bug Pattern | Meaning | Fix |
|------------|---------|-----|
| `NP_NULL_ON_SOME_PATH` | Possible null pointer | Add null check or use Optional |
| `EI_EXPOSE_REP` | Mutable field exposed | Return defensive copy |
| `EI_EXPOSE_REP2` | Mutable field stored directly | Store defensive copy |
| `RCN_REDUNDANT_NULLCHECK` | Redundant null check | Remove unnecessary check |
| `SE_BAD_FIELD` | Non-serializable field in serializable class | Mark as transient or make serializable |

### Rule CQ05: Fix Violations, Don't Suppress

**Requirement Level**: MUST

Never add `@SuppressFBWarnings` or `// CHECKSTYLE:OFF` to bypass violations without a documented justification. If a rule is genuinely inapplicable, configure the exclusion in the project's static analysis config files (`checkstyle.xml`, `spotbugs-exclude.xml`), not inline.

---

## Code Formatting

### Rule CQ06: Follow Consistent Formatting

**Requirement Level**: MUST

If the project uses a formatter (Spotless, google-java-format), run it:

```bash
# Spotless (Gradle)
./gradlew spotlessApply

# Google Java Format (standalone)
google-java-format --replace src/**/*.java
```

If no formatter is configured, follow these minimum rules:
- 4-space indentation (no tabs)
- Opening brace on same line as statement
- One blank line between methods
- No trailing whitespace
- File ends with a newline

### Rule CQ07: Import Order

**Requirement Level**: SHOULD

Follow a consistent import order:
1. `java.*`
2. `javax.*` / `jakarta.*`
3. Third-party libraries (alphabetical)
4. Project imports
5. Static imports last

Remove all unused imports.

---

## Build Verification

### Rule CQ08: Full Build Must Pass

**Requirement Level**: MUST

Run the complete build to catch issues that individual tasks might miss:

```bash
# Gradle
./gradlew build

# Maven
mvn clean verify
```

This runs compilation, unit tests, integration tests, and any configured quality plugins in the correct order.

### Rule CQ09: No New Test Failures

**Requirement Level**: MUST

After making changes, the full test suite must pass:

```bash
./gradlew test integrationTest
```

If pre-existing test failures exist (tests that were already failing before your changes), document them but do not fix them in the same PR unless they are directly related to your ticket.

### Rule CQ10: Dependency Hygiene

**Requirement Level**: SHOULD

When adding new dependencies:
- Verify the dependency is actively maintained (recent releases, not archived)
- Check for known vulnerabilities
- Use the latest stable version, not snapshots
- Add dependencies to the correct scope (`implementation` vs. `testImplementation`)
- Never add a dependency when the functionality already exists in a current dependency

---

## Pre-Commit Checklist

### Rule CQ11: Run This Checklist Before Every Commit

**Requirement Level**: MUST

```bash
# 1. Compile everything
./gradlew compileJava compileTestJava

# 2. Run all tests
./gradlew test integrationTest

# 3. Static analysis (if configured)
./gradlew checkstyleMain spotbugsMain 2>/dev/null

# 4. Review the diff
git diff --stat
git diff
```

If any step fails, fix the issue before committing. Do not commit with known failures.

---

## Enforcement Checklist

Before committing, verify:

- [ ] `compileJava` and `compileTestJava` succeed with no errors
- [ ] All unit tests pass (`./gradlew test`)
- [ ] All integration tests pass (`./gradlew integrationTest`)
- [ ] Checkstyle passes (if configured)
- [ ] SpotBugs passes (if configured)
- [ ] No unused imports
- [ ] No debugging code (`System.out.println`, hardcoded test values)
- [ ] No commented-out code blocks
- [ ] No `TODO` or `FIXME` comments added (track in tickets instead)
- [ ] Build file changes are minimal and justified
- [ ] All new dependencies use stable versions

## References

- For complete code examples, see [examples.md](examples.md)
- Checkstyle: https://checkstyle.sourceforge.io/
- SpotBugs: https://spotbugs.github.io/
- Google Java Format: https://github.com/google/google-java-format
- OWASP Dependency Check: https://owasp.org/www-project-dependency-check/
