# Resolve Ticket Plugin

End-to-end ticket resolution: analyze requirements, design architecture, implement code, write tests, verify quality, and prepare a production-ready commit — all from a single command.

## Commands

| Command | Description |
|---------|-------------|
| `/resolve-ticket` | Analyze a ticket, implement the solution end-to-end, write tests, verify quality, and prepare a commit with PR |

### `/resolve-ticket`

Full autonomous workflow that takes a ticket (Jira ID, GitHub Issue URL, or plain text description) and:

1. **Loads all coding standards** — architecture, implementation, testing, quality, VCS conventions
2. **Analyzes the ticket** — extracts requirements, acceptance criteria, scope
3. **Analyzes the codebase** — maps structure, patterns, test infrastructure, naming conventions
4. **Designs the solution** — determines layers, components, data flow, error handling
5. **Implements the code** — writes production code following existing patterns
6. **Writes unit tests** — JUnit 5 + Mockito with Given-When-Then structure
7. **Writes integration tests** — Spring Boot Test + Testcontainers for end-to-end verification
8. **Runs quality checks** — compilation, full test suite, static analysis, self-review
9. **Commits and creates PR** — proper branch naming, commit message, and PR body with ticket link

**Usage:**
```
/resolve-ticket CMS-42
/resolve-ticket https://github.com/org/repo/issues/42
/resolve-ticket "Add a health score calculation endpoint for clients"
```

The command runs fully autonomously — no confirmation prompts between phases.

## Skills

### Architecture & Implementation

| Skill | Description |
|-------|-------------|
| `resolve-ticket:codebase-analysis` | Systematic project structure analysis and pattern recognition |
| `resolve-ticket:architecture-design` | Layered architecture, DDD, Spring Boot patterns, REST API design |
| `resolve-ticket:implementation` | Coding conventions, naming rules, Lombok, validation, logging |

### Testing

| Skill | Description |
|-------|-------------|
| `resolve-ticket:unit-testing` | Unit test patterns with JUnit 5, Mockito, Hamcrest, and test data builders |
| `resolve-ticket:integration-testing` | Integration tests with Spring Boot Test, Testcontainers, and MongoDB |

### Quality & Process

| Skill | Description |
|-------|-------------|
| `resolve-ticket:code-quality` | Compilation, Checkstyle, SpotBugs, formatting, dependency hygiene |
| `resolve-ticket:code-review` | Self-review checklist, security review, architecture verification |
| `resolve-ticket:vcs` | Git workflow, branch naming, commit messages, PR conventions |

Skills activate automatically when referenced by the `/resolve-ticket` command. They can also be loaded individually for targeted guidance.

## Target Stack

This plugin is optimized for:
- **Language**: Java 17+ / Kotlin
- **Framework**: Spring Boot 3.x
- **Database**: MongoDB (Spring Data MongoDB)
- **Build**: Gradle or Maven
- **Testing**: JUnit 5, Mockito, Hamcrest, Testcontainers
- **Code Generation**: MapStruct, Lombok
- **Style**: Google Java Style Guide

The skills adapt to any Java/Spring Boot project by reading existing patterns in the codebase. Stack-specific examples serve as defaults when no existing patterns are found.

## Best Practices

- Use `/resolve-ticket` for the full workflow — it applies all standards automatically
- Skills are composable — you can invoke any skill independently for targeted guidance
- The plugin follows whatever conventions already exist in the target codebase
- If the codebase diverges from the loaded skills, codebase conventions take precedence

## Version

1.0.0 | AI Integration Team
