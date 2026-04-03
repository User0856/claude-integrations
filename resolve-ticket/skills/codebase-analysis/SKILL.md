---
name: Codebase Analysis
description: Systematic approach to understanding project structure, identifying patterns, and mapping dependencies before making changes
when_to_apply: Before implementing any changes — always analyze the codebase first to understand existing conventions and avoid breaking established patterns
version: 1.0.0
languages: Java, Kotlin
globs: "**/*.{java,kt,gradle,yml,yaml,xml,json,properties}"
alwaysApply: false
---

# Codebase Analysis

## Overview

Before writing a single line of code, you must understand the project you are working in. This skill defines the systematic approach to exploring a codebase — identifying the build system, framework, architectural patterns, test infrastructure, and coding conventions. Skipping this step leads to inconsistent implementations that clash with established patterns.

**Core principles:**
- Always analyze the codebase before writing code
- Identify build system, framework, and architecture as the first step
- Read before write — study examples of the same component type
- Document naming conventions, error handling, and test infrastructure
- Match existing patterns exactly in all new code

## When to Apply This Skill

- Starting work on any ticket in an unfamiliar or partially familiar codebase
- Before implementing a feature that touches multiple layers (controller, service, repository)
- When the ticket references components you have not previously read
- When evaluating the impact of a change across the project

## Quick Reference

| Rule | Requirement | Level | Key Point |
|------|------------|-------|-----------|
| CA01 | Identify build system | MUST | Gradle vs Maven, dependencies, plugins |
| CA02 | Identify framework and runtime | MUST | Spring Boot indicators, config files |
| CA03 | Map package structure | MUST | Layer-based vs feature-based layout |
| CA04 | Identify shared infrastructure | SHOULD | Base classes, exception handlers, configs |
| CA05 | Map external dependencies | SHOULD | Database, message brokers, HTTP clients |
| CA06 | Study existing implementations | MUST | Find and read examples of each component type |
| CA07 | Identify naming conventions | MUST | Entity, service, controller, DTO, test naming |
| CA08 | Understand error handling | MUST | Custom exceptions, handler mappings, error format |
| CA09 | Map test setup | MUST | Frameworks, base classes, test data, naming |
| CA10 | Check test coverage gaps | SHOULD | Current test count, disabled tests, untested paths |

---

## Phase 1: Project Identity

### Rule CA01: Identify the Build System

**Requirement Level**: MUST

Determine the build tool and project metadata before anything else.

| File | Build System | Key Commands |
|------|-------------|--------------|
| `build.gradle` / `build.gradle.kts` | Gradle | `./gradlew build`, `./gradlew test` |
| `pom.xml` | Maven | `mvn clean install`, `mvn test` |
| `settings.gradle` / `settings.gradle.kts` | Gradle multi-module | Check `include` directives |

Read the build file to identify:
- Java/Kotlin version
- Spring Boot version
- Key dependencies (MongoDB, JPA, MapStruct, Lombok, etc.)
- Test dependencies (JUnit 5, Mockito, Testcontainers, etc.)
- Plugins (Checkstyle, SpotBugs, JaCoCo, etc.)
- Custom tasks or configurations

### Rule CA02: Identify the Framework and Runtime

**Requirement Level**: MUST

Determine the application framework by checking:

| Indicator | Framework |
|-----------|-----------|
| `spring-boot-starter-*` dependencies | Spring Boot |
| `@SpringBootApplication` in main class | Spring Boot |
| `quarkus-*` dependencies | Quarkus |
| `io.micronaut` imports | Micronaut |

Check `application.yml` / `application.properties` for:
- Database connection configuration (MongoDB URI, JPA datasource)
- Server port and context path
- Active profiles
- Custom application properties

---

## Phase 2: Architecture Mapping

### Rule CA03: Map the Package Structure

**Requirement Level**: MUST

Scan the `src/main/java` (or `src/main/kotlin`) tree to understand the architectural layers.

**Common package patterns:**

```
com.company.project/
  config/          # Spring configuration classes
  domain/
    model/         # Entity classes, enums, value objects
    event/         # Domain events
    exception/     # Custom exceptions
  repository/      # Data access interfaces
  service/         # Business logic
  dto/             # Request/response DTOs
  mapper/          # MapStruct or manual mappers
  controller/      # REST controllers
```

**Alternative patterns:**

```
com.company.project/
  feature1/
    controller/
    service/
    repository/
    model/
  feature2/
    ...
```

Document which pattern the project follows — all new code must match.

### Rule CA04: Identify Shared Infrastructure

**Requirement Level**: SHOULD

Look for cross-cutting components:
- **Base classes**: `BaseEntity`, `BaseIntegrationTest`, `AbstractService`
- **Global exception handlers**: `@RestControllerAdvice` classes
- **Audit infrastructure**: `@EnableMongoAuditing`, `AuditAware` implementations
- **Configuration classes**: `WebConfig`, `MongoConfig`, `SecurityConfig`
- **Utility classes**: Common validators, date utilities, string helpers

### Rule CA05: Map External Dependencies

**Requirement Level**: SHOULD

Identify external system integrations:
- Database type and access pattern (Spring Data MongoDB, JPA, JDBC)
- Message brokers (RabbitMQ, Kafka, ActiveMQ)
- External HTTP clients (RestTemplate, WebClient, Feign)
- File storage, email, caching

---

## Phase 3: Pattern Recognition

### Rule CA06: Study Existing Implementations

**Requirement Level**: MUST

Before creating a new component, find at least one existing example of the same type in the codebase:

| Creating | Find Example Of |
|----------|----------------|
| New entity | Existing `@Document` or `@Entity` class |
| New service | Existing `@Service` class with similar complexity |
| New controller | Existing `@RestController` with CRUD operations |
| New repository | Existing `MongoRepository` or `JpaRepository` interface |
| New DTO | Existing request/response DTO pair |
| New mapper | Existing MapStruct `@Mapper` interface |
| New test | Existing test class for the same layer |

Read the example completely. Match its:
- Import style (wildcard vs. specific)
- Annotation usage
- Method ordering
- Naming conventions
- Error handling approach
- Logging patterns

### Rule CA07: Identify Naming Conventions

**Requirement Level**: MUST

Document the conventions used in the project:

| Element | Common Pattern | Example |
|---------|---------------|---------|
| Entity class | `PascalCase`, singular | `Client`, `Contract`, `Invoice` |
| Repository | `{Entity}Repository` | `ClientRepository` |
| Service | `{Entity}Service` or `{Feature}Service` | `ClientService`, `HealthScoreService` |
| Controller | `{Entity}Controller` | `ClientController` |
| Create DTO | `Create{Entity}Request` | `CreateClientRequest` |
| Update DTO | `Update{Entity}Request` | `UpdateClientRequest` |
| Response DTO | `{Entity}Response` | `ClientResponse` |
| Mapper | `{Entity}Mapper` | `ClientMapper` |
| Test class | `{ClassName}Test` | `ClientServiceTest` |
| Integration test | `{Feature}IntegrationTest` | `ClientControllerIntegrationTest` |

### Rule CA08: Understand Error Handling

**Requirement Level**: MUST

Identify how the project handles errors:
- Custom exception classes and their hierarchy
- Global exception handler mappings (exception → HTTP status)
- Error response format (body structure)
- Validation error handling approach

---

## Phase 4: Test Infrastructure

### Rule CA09: Map Test Setup

**Requirement Level**: MUST

Understand the testing infrastructure before writing tests:

1. **Unit test setup**: Which mocking framework, assertion library, base classes
2. **Integration test setup**: Testcontainers configuration, `@DynamicPropertySource` usage, test profiles
3. **Test data**: Builders, factories, fixtures, or inline construction
4. **Test naming**: `test_method_scenario`, `shouldDoXWhenY`, or other convention
5. **Test structure**: Given-When-Then comments, Arrange-Act-Assert, or no explicit structure

### Rule CA10: Check Test Coverage Gaps

**Requirement Level**: SHOULD

Before writing new tests, check what is already tested:
- Run `./gradlew test` or `mvn test` to see current test count
- Check for `@Disabled` or `@Ignore` tests
- Look for TODO comments indicating planned tests
- Identify untested code paths that may be affected by the change

---

## Enforcement Checklist

Before proceeding to implementation, verify:

- [ ] Build system identified (Gradle/Maven) with all key dependencies listed
- [ ] Framework and runtime configuration understood
- [ ] Package structure mapped and architectural pattern documented
- [ ] At least one example of each component type found and read
- [ ] Naming conventions documented
- [ ] Error handling pattern understood
- [ ] Test infrastructure and conventions mapped
- [ ] All files that will be modified have been read completely
- [ ] Impact on existing tests assessed

## References

- For complete code examples, see [examples.md](examples.md)
- Project build file (`build.gradle` or `pom.xml`)
- `application.yml` / `application.properties`
- Existing `@RestControllerAdvice` exception handler
- Base test classes in `src/test/`
