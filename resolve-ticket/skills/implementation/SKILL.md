---
name: Implementation Standards
description: Coding conventions, naming rules, Lombok usage, validation patterns, logging practices, and Spring Boot idioms for consistent, production-grade Java code
when_to_apply: When writing or modifying any Java/Kotlin source code in Spring Boot projects
version: 1.0.0
languages: Java, Kotlin
globs: "src/**/*.{java,kt}"
alwaysApply: false
---

# Implementation Standards

## Overview

Consistent code is maintainable code. This skill defines the concrete coding conventions for Spring Boot applications — naming rules, annotation usage, dependency injection patterns, validation, logging, and common idioms. Every developer (human or AI) writing code in the project must follow these standards to maintain a uniform codebase.

**Core principles:**
- Follow Java naming conventions consistently across all code
- Use constructor injection via @RequiredArgsConstructor, never field injection
- Validate at system boundaries using Jakarta annotations on request DTOs
- Log at service boundaries with appropriate levels, never log sensitive data
- Keep methods short, focused, and use Stream API where it improves readability

## When to Apply This Skill

- Writing any new Java or Kotlin source file
- Modifying existing source code
- Adding new Spring beans, services, or components
- Implementing validation logic
- Adding logging statements
- Reviewing code for standards compliance

## Quick Reference

| Rule | Requirement | Level | Key Point |
|------|------------|-------|-----------|
| IM01 | Follow Java naming standards | MUST | PascalCase classes, camelCase methods, UPPER_SNAKE constants |
| IM02 | Use descriptive names | MUST | Names reveal intent, boolean prefixes (is/has/can) |
| IM03 | Use Lombok annotations consistently | MUST | @Data, @Builder, @RequiredArgsConstructor, @Slf4j |
| IM04 | Never use @Autowired for field injection | MUST | Constructor injection via @RequiredArgsConstructor only |
| IM05 | Validate at system boundaries | MUST | Jakarta validation annotations on request DTOs |
| IM06 | Use custom validation for business rules | SHOULD | Validate in service layer, throw typed exceptions |
| IM07 | Log at service boundaries | MUST | Entry and exit points of significant operations |
| IM08 | Use appropriate log levels | MUST | ERROR/WARN/INFO/DEBUG for correct scenarios |
| IM09 | Never log sensitive data | MUST NOT | No passwords, PII, financial data, health data |
| IM10 | Use structured logging parameters | SHOULD | SLF4J parameterized logging, no string concatenation |
| IM11 | Use Optional correctly | MUST | orElseThrow/map, not isPresent/get |
| IM12 | Use Stream API for collections | SHOULD | Stream over manual loops when cleaner |
| IM13 | Use Records for simple DTOs | MAY | Java 17+ records for immutable response DTOs |
| IM14 | Order class members consistently | SHOULD | Static fields, instance fields, constructors, public, private |
| IM15 | Keep methods short and focused | SHOULD | One thing per method, under 30 lines, max 4 parameters |

---

## Naming Conventions

### Rule IM01: Follow Java Naming Standards

**Requirement Level**: MUST

| Element | Convention | Example |
|---------|-----------|---------|
| Package | `lowercase.dotted` | `com.enterprise.cms.service` |
| Class | `PascalCase` | `ContractService`, `ClientMapper` |
| Interface | `PascalCase` | `ContractRepository` |
| Method | `camelCase` | `findByClientId`, `calculateHealthScore` |
| Variable | `camelCase` | `contractNumber`, `totalRevenue` |
| Constant | `UPPER_SNAKE_CASE` | `MAX_RETRY_COUNT`, `DEFAULT_PAGE_SIZE` |
| Enum value | `UPPER_SNAKE_CASE` | `PENDING_APPROVAL`, `AT_RISK` |
| Test method | `test_{method}_{scenario}_{result}` | `test_create_validInput_returnsCreated` |

### Rule IM02: Use Descriptive Names

**Requirement Level**: MUST

- Names must reveal intent without requiring a comment
- Avoid abbreviations unless universally understood (`id`, `dto`, `url` are acceptable)
- Boolean variables and methods use `is`, `has`, `can`, `should` prefixes
- Collection variables use plural nouns: `clients`, `lineItems`, `statusHistory`

**Good:**
```java
private final ContractRepository contractRepository;
boolean isExpired = contract.getEndDate().isBefore(LocalDate.now());
List<Contract> activeContracts = findByStatus(ContractStatus.ACTIVE);
```

**Bad:**
```java
private final ContractRepository cr;       // abbreviated
boolean flag = contract.getEndDate().isBefore(LocalDate.now()); // meaningless
List<Contract> list = findByStatus(ContractStatus.ACTIVE);      // generic
```

---

## Lombok Usage

### Rule IM03: Use Lombok Annotations Consistently

**Requirement Level**: MUST

| Annotation | Use For |
|-----------|---------|
| `@Data` | DTOs, entities (generates getters, setters, equals, hashCode, toString) |
| `@Builder` | Any class that benefits from builder pattern (entities, DTOs, test data) |
| `@NoArgsConstructor` | Required by JPA/MongoDB deserialization alongside `@Builder` |
| `@AllArgsConstructor` | Required by `@Builder` alongside `@NoArgsConstructor` |
| `@RequiredArgsConstructor` | Service classes with constructor injection |
| `@Slf4j` | Any class that needs logging |
| `@Getter` / `@Setter` | When `@Data` is too broad (e.g., immutable response objects, use `@Getter` only) |
| `@Value` | Immutable value objects |

### Rule IM04: Never Use @Autowired for Field Injection

**Requirement Level**: MUST

Use constructor injection via `@RequiredArgsConstructor`:

**Good:**
```java
@Service
@RequiredArgsConstructor
public class ClientService {
    private final ClientRepository clientRepository;
    private final ApplicationEventPublisher eventPublisher;
}
```

**Bad:**
```java
@Service
public class ClientService {
    @Autowired
    private ClientRepository clientRepository;   // field injection
    @Autowired
    private ApplicationEventPublisher eventPublisher;
}
```

---

## Validation

### Rule IM05: Validate at System Boundaries

**Requirement Level**: MUST

Apply Jakarta validation annotations on request DTOs (the entry point of external data). Do not re-validate inside services unless there are business rules beyond basic format/presence checks.

**Standard annotations:**

| Annotation | Purpose | Example |
|-----------|---------|---------|
| `@NotNull` | Field must be present | Required references |
| `@NotBlank` | String must be non-null and non-empty | Names, descriptions |
| `@Size(min, max)` | String or collection length bounds | `@Size(max = 255)` |
| `@Email` | Valid email format | Email fields |
| `@Pattern(regexp)` | Regex match | Phone numbers, codes |
| `@DecimalMin` / `@DecimalMax` | Numeric bounds | `@DecimalMin("0.01")` |
| `@Future` / `@Past` | Date constraints | End dates, birth dates |
| `@Valid` | Cascade validation to nested objects | Embedded DTOs |

### Rule IM06: Use Custom Validation for Business Rules

**Requirement Level**: SHOULD

When Jakarta annotations are insufficient, validate in the service layer and throw typed exceptions:

```java
public Contract create(Contract contract) {
    if (contract.getEndDate().isBefore(contract.getStartDate())) {
        throw new BusinessRuleViolationException(
                "DATE_ORDER", "End date must be after start date");
    }
    if (contractRepository.existsByContractNumber(contract.getContractNumber())) {
        throw new DuplicateEntityException(
                "Contract", "contractNumber", contract.getContractNumber());
    }
    return contractRepository.save(contract);
}
```

---

## Logging

### Rule IM07: Log at Service Boundaries

**Requirement Level**: MUST

Add logging at the entry and exit points of significant operations:

```java
@Slf4j
@Service
public class InvoiceService {

    public Invoice generateFromContract(String contractId) {
        log.info("Generating invoice for contract {}", contractId);
        // ... logic ...
        log.info("Invoice {} generated for contract {}, total: {}",
                invoice.getId(), contractId, invoice.getTotal());
        return invoice;
    }
}
```

### Rule IM08: Use Appropriate Log Levels

**Requirement Level**: MUST

| Level | When to Use | Example |
|-------|------------|---------|
| `ERROR` | Unexpected failure affecting a request | Database connection failure, unhandled exception |
| `WARN` | Recoverable issue, may need attention | Retry succeeded, deprecated API called |
| `INFO` | Significant business events and lifecycle | Entity created, status changed, job completed |
| `DEBUG` | Detailed flow for troubleshooting | Method parameters, intermediate calculations |

### Rule IM09: Never Log Sensitive Data

**Requirement Level**: MUST NOT

Never log:
- Passwords, tokens, or API keys
- Personal identifiable information (PII): email, phone, SSN
- Financial data: credit card numbers, bank accounts
- Health data

**Good:** `log.info("User {} authenticated successfully", userId);`
**Bad:** `log.info("User logged in with email={}, password={}", email, password);`

### Rule IM10: Use Structured Logging Parameters

**Requirement Level**: SHOULD

Use SLF4J parameterized logging (never string concatenation):

**Good:** `log.info("Processing order {} for client {}", orderId, clientId);`
**Bad:** `log.info("Processing order " + orderId + " for client " + clientId);`

---

## Spring Boot Idioms

### Rule IM11: Use Optional Correctly

**Requirement Level**: MUST

```java
// Good — orElseThrow for required entities
Client client = clientRepository.findById(id)
        .orElseThrow(() -> new EntityNotFoundException("Client", id));

// Good — map/flatMap for transformations
Optional<String> name = clientRepository.findById(id)
        .map(Client::getCompanyName);

// Bad — isPresent/get
Optional<Client> optional = clientRepository.findById(id);
if (optional.isPresent()) {    // avoid this pattern
    return optional.get();
}
```

### Rule IM12: Use Stream API for Collections

**Requirement Level**: SHOULD

```java
// Good
List<ClientResponse> responses = clients.stream()
        .map(clientMapper::toResponse)
        .toList();

// Good — filtering
List<Contract> active = contracts.stream()
        .filter(c -> c.getStatus() == ContractStatus.ACTIVE)
        .toList();

// Avoid — manual loop when stream is cleaner
List<ClientResponse> responses = new ArrayList<>();
for (Client client : clients) {
    responses.add(clientMapper.toResponse(client));
}
```

### Rule IM13: Use Records for Simple DTOs (Java 17+)

**Requirement Level**: MAY

For immutable response DTOs with no validation:

```java
public record ErrorResponse(int status, String error, String message) {}
public record PageResponse<T>(List<T> content, int page, int size, long totalElements) {}
```

For request DTOs that need `@Valid` and Lombok builders, stick with classes.

---

## Code Organization

### Rule IM14: Order Class Members Consistently

**Requirement Level**: SHOULD

1. Static fields (constants)
2. Instance fields
3. Constructors
4. Public methods
5. Package-private methods
6. Private methods

### Rule IM15: Keep Methods Short and Focused

**Requirement Level**: SHOULD

- Methods should do one thing
- If a method exceeds 30 lines, consider extracting helper methods
- If a method has more than 4 parameters, consider a parameter object
- Private helper methods should follow the method they support

---

## Enforcement Checklist

Before submitting code, verify:

- [ ] All names follow Java naming conventions (Rule IM01)
- [ ] No meaningless names (`temp`, `data`, `result`, `flag`) without context
- [ ] Lombok annotations match project patterns
- [ ] No `@Autowired` field injection — all constructor injection via `@RequiredArgsConstructor`
- [ ] Request DTOs have Jakarta validation annotations
- [ ] Business rule violations throw typed exceptions
- [ ] `@Slf4j` logging at service boundaries with meaningful messages
- [ ] No sensitive data in log statements
- [ ] Parameterized logging used (no string concatenation)
- [ ] `Optional` used with `orElseThrow` / `map`, not `isPresent`/`get`
- [ ] Stream API used where it improves readability
- [ ] Methods under 30 lines, classes have clear single responsibility

## References

- For complete code examples, see [examples.md](examples.md)
- Google Java Style Guide: https://google.github.io/styleguide/javaguide.html
- Spring Boot Best Practices: https://docs.spring.io/spring-boot/reference/
- Lombok Documentation: https://projectlombok.org/features/
