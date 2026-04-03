# Claude Integrations

A Claude Code plugin with two commands: `/resolve-ticket` implements a ticket end-to-end, `/review-pr` performs a multi-dimensional code review. Both use the same 8 skills as their standard.

## Quick Start

### Prerequisites

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code/overview) installed
- A Java/Spring Boot project (or any project — the skills adapt)
- Git configured with a remote
- [GitHub CLI](https://cli.github.com/) (`gh`) for PR creation and review

### Install

```bash
cd /path/to/your-project

# Copy commands and skills into your project
mkdir -p .claude/commands .claude/skills
cp /path/to/claude-integrations/resolve-ticket/commands/*.md .claude/commands/
cp -r /path/to/claude-integrations/resolve-ticket/skills/* .claude/skills/
```

For global access (all projects), use `~/.claude/commands` and `~/.claude/skills` instead.

### Use

```bash
claude
```

```
/resolve-ticket CMS-42
/resolve-ticket https://github.com/my-org/my-repo/issues/42
/resolve-ticket "Add a health score calculation endpoint for clients"

/review-pr 42
/review-pr https://github.com/my-org/my-repo/pull/42
/review-pr feature/cms-42
/review-pr                    # reviews PR for current branch
```

---

## Commands

### `/resolve-ticket` — Implement a ticket end-to-end

Runs autonomously through 10 phases:

1. **Load Standards** — loads all 8 skills
2. **Analyze Ticket** — reads ticket from Jira, GitHub, or plain text
3. **Analyze Codebase** — scans project structure and existing patterns
4. **Design** — plans architecture, layers, data flow
5. **Branch Setup** — stash changes, update main, create `feature/<ticket-id>` branch
6. **Implement** — writes entities, services, controllers, DTOs
7. **Unit Tests** — JUnit 5 + Mockito, retries up to 3x
8. **Integration Tests** — Testcontainers, retries up to 3x
9. **Quality Check** — compile, test suite, static analysis, self-review
10. **Commit + PR** — commit, push, open pull request

Output: PR link, file change list, test results.

### `/review-pr` — Review a pull request

Runs autonomously through 10 phases:

1. **Load Standards** — loads all 8 skills
2. **PR Discovery** — fetches PR metadata, diff, and commit history via `gh`
3. **Change Scope Analysis** — categorizes files by risk, computes metrics, flags missing companions
4. **Codebase Context** — scans existing patterns to establish baselines for comparison
5. **File-Level Review** — reads every changed file in full context, not just the diff
6. **Parallel Skill Reviews** — launches 7 review agents (one per skill) against the full diff
7. **Security Review** — OWASP Top 10 scan, dependency review, configuration review
8. **Test Adequacy** — coverage mapping, quality assessment, missing scenario detection
9. **Findings Synthesis** — deduplicates, categorizes by severity, prioritizes action items
10. **Report** — posts structured review as PR comment, submits GitHub review verdict

Output: APPROVE / REQUEST CHANGES / COMMENT verdict, findings breakdown, PR comment link.

---

## Skills

| Skill | Covers |
|-------|--------|
| `codebase-analysis` | Project exploration and pattern recognition |
| `architecture-design` | Layered architecture, REST API design, MongoDB patterns |
| `implementation` | Naming, Lombok, validation, logging, Spring idioms |
| `unit-testing` | JUnit 5, Mockito, Given-When-Then, test data builders |
| `integration-testing` | Spring Boot Test, Testcontainers, DB verification |
| `code-quality` | Compilation, Checkstyle, SpotBugs, formatting |
| `code-review` | Self-review, security, architecture verification |
| `vcs` | Branch naming, commits, PR format, push checklist |

`/resolve-ticket` loads all skills automatically. You can also reference individual skills for guidance.

---

## Target Stack

Written for Java/Spring Boot but adapts to the target project's conventions:

- Java 17+ / Kotlin
- Spring Boot 3.x+
- MongoDB (Spring Data)
- Gradle or Maven
- JUnit 5, Mockito, Testcontainers

---

## Troubleshooting

**Command not found** — Check `.claude/commands/resolve-ticket.md` exists; restart Claude Code.

**Skills not found** — Verify all 8 directories under `.claude/skills/`, each with a `SKILL.md`. Run `ls .claude/skills/*/SKILL.md`.

**Jira fetch fails** — Configure Atlassian MCP (`claude mcp add atlassian`) or pass ticket text directly.

**Tests keep failing** — The command retries 3x then stops. Check the output and fix manually.

---

## File Structure

```
claude-integrations/
  README.md
  resolve-ticket/
    .claude-plugin/plugin.json
    README.md
    commands/
      resolve-ticket.md
      review-pr.md
    skills/
      codebase-analysis/
        SKILL.md
        examples.md
      architecture-design/
        SKILL.md
        examples.md
      implementation/
        SKILL.md
        examples.md
      unit-testing/
        SKILL.md
        examples.md
      integration-testing/
        SKILL.md
        examples.md
      code-quality/
        SKILL.md
        examples.md
      code-review/
        SKILL.md
        examples.md
      vcs/
        SKILL.md
        examples.md
        references/commit-standards.md
```
