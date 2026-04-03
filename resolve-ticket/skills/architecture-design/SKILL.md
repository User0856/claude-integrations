---
name: Architecture and Design Standards
description: Layered architecture patterns following DDD principles for Spring Boot applications with MongoDB — defines layer responsibilities, dependency rules, and component design
when_to_apply: When designing new components, creating new services or endpoints, or reviewing architectural decisions in Spring Boot projects
version: 1.0.0
languages: Java, Kotlin
globs: "src/main/java/**/*.{java,kt}"
alwaysApply: false
---

# Architecture and Design Standards

## Overview

This skill defines the standard architectural patterns for backend services following Domain-Driven Design (DDD) principles with a layered architecture. Every component must belong to exactly one layer, and dependencies must flow in one direction: Controllers → Services → Repositories. Violations of layer boundaries create tight coupling, circular dependencies, and untestable code.

**Core principles:**
- Every component belongs to exactly one architectural layer
- Dependencies flow in one direction: Controllers → Services → Repositories
- Controllers handle HTTP only — zero business logic
- Use typed exceptions for domain errors, mapped via centralized exception handling
- Prefer domain events over direct service-to-service calls for side effects

## When to Apply This Skill

- Designing new features or components
- Creating new REST API endpoints
- Adding new domain entities or business logic
- Restructuring or refactoring existing code
- Reviewing pull requests for architectural compliance

## Quick Reference

| Rule | Requirement | Level | Key Point |
|------|------------|-------|-----------|
| AD01 | Controllers handle HTTP only | MUST | No business logic, delegate to services |
| AD02 | Use consistent URL patterns | MUST | `/api/v1/{resources}`, correct HTTP methods and status codes |
| AD03 | Design DTOs for each operation | MUST | Separate Create/Update request and Response DTOs |
| AD04 | Services own business logic | MUST | Business rules, orchestration, transactions, events |
| AD05 | Services prefer events over direct calls | SHOULD | Domain events for cross-service communication |
| AD06 | Use custom exceptions for domain errors | MUST | Typed exceptions mapped to HTTP status codes |
| AD07 | Repositories are interfaces | MUST | Spring Data interfaces with naming conventions |
| AD08 | Use MongoDB document patterns consistently | MUST | @Indexed on query fields, @CreatedDate/@LastModifiedDate |
| AD09 | Centralize exception handling | MUST | Single @RestControllerAdvice for all exception mapping |
| AD10 | Use domain events for side effects | SHOULD | Audit, notifications, cache invalidation via events |

---

## Architecture Layers

```
┌─────────────────────────────────┐
│         Controllers             │  ← HTTP interface, request/response mapping
│    (interfaces / REST API)      │
├─────────────────────────────────┤
│       Service Layer             │  ← Business logic, orchestration, transactions
│   (application / domain)        │
├─────────────────────────────────┤
│      Repository Layer           │  ← Data access, persistence
│     (infrastructure)            │
├─────────────────────────────────┤
│      Database / External        │  ← MongoDB, message brokers, external APIs
└─────────────────────────────────┘
```

---

## Controller Layer

### Rule AD01: Controllers Handle HTTP Only

**Requirement Level**: MUST

Controllers are responsible for:
- Receiving HTTP requests and parsing parameters
- Validating request DTOs (via Jakarta validation annotations)
- Delegating to the service layer
- Mapping service responses to HTTP responses (status codes, headers)

Controllers must NOT:
- Contain business logic (no `if/else` based on domain rules)
- Call repositories directly
- Perform data transformations beyond DTO mapping
- Catch and handle domain exceptions (use `@RestControllerAdvice` instead)

```java
@RestController
@RequestMapping("/api/v1/clients")
@RequiredArgsConstructor
public class ClientController {

    private final ClientService clientService;
    private final ClientMapper clientMapper;

    @PostMapping
    public ResponseEntity<ClientResponse> create(
            @Valid @RequestBody CreateClientRequest request) {
        Client client = clientService.create(clientMapper.toEntity(request));
        return ResponseEntity
                .status(HttpStatus.CREATED)
                .body(clientMapper.toResponse(client));
    }

    @GetMapping("/{id}")
    public ResponseEntity<ClientResponse> getById(@PathVariable String id) {
        return ResponseEntity.ok(
                clientMapper.toResponse(clientService.getById(id)));
    }
}
```

### Rule AD02: Use Consistent URL Patterns

**Requirement Level**: MUST

| Operation | Method | URL Pattern | Status Code |
|-----------|--------|-------------|-------------|
| List/Search | GET | `/api/v1/{resources}` | 200 |
| Get by ID | GET | `/api/v1/{resources}/{id}` | 200 |
| Create | POST | `/api/v1/{resources}` | 201 |
| Full update | PUT | `/api/v1/{resources}/{id}` | 200 |
| Partial update | PATCH | `/api/v1/{resources}/{id}` | 200 |
| Delete | DELETE | `/api/v1/{resources}/{id}` | 204 |
| Custom action | POST | `/api/v1/{resources}/{id}/{action}` | 200 |
| Sub-resource | GET | `/api/v1/{resources}/{id}/{sub-resources}` | 200 |

- Resource names are plural and kebab-case: `/api/v1/audit-events`, not `/api/v1/auditEvent`
- Version prefix is always `v1` unless the project uses a different convention
- Custom actions use POST, not GET (actions have side effects)

### Rule AD03: Design DTOs for Each Operation

**Requirement Level**: MUST

Each entity should have separate DTOs for different operations:

| DTO Type | Purpose | Validation |
|----------|---------|------------|
| `Create{Entity}Request` | POST body for creation | All required fields validated |
| `Update{Entity}Request` | PUT/PATCH body for updates | Nullable fields for partial update |
| `{Entity}Response` | Response body | No validation needed |
| `{Entity}SummaryResponse` | List item (reduced fields) | No validation needed |
| `PageResponse<T>` | Paginated list wrapper | Generic, reusable |

Never expose domain entities directly in API responses. DTOs provide a stable API contract independent of internal model changes.

---

## Service Layer

### Rule AD04: Services Own Business Logic

**Requirement Level**: MUST

Services are responsible for:
- Implementing business rules and domain logic
- Orchestrating operations across multiple repositories
- Publishing domain events
- Transaction boundary management
- Input validation beyond DTO-level constraints

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class ContractService {

    private final ContractRepository contractRepository;
    private final AuditService auditService;
    private final ApplicationEventPublisher eventPublisher;

    public Contract transitionStatus(String contractId, ContractStatus newStatus) {
        Contract contract = contractRepository.findById(contractId)
                .orElseThrow(() -> new EntityNotFoundException("Contract", contractId));

        ContractStatus oldStatus = contract.getStatus();
        validateTransition(oldStatus, newStatus);

        contract.setStatus(newStatus);
        contract.getStatusHistory().add(StatusChange.builder()
                .fromStatus(oldStatus)
                .toStatus(newStatus)
                .changedAt(Instant.now())
                .build());

        Contract saved = contractRepository.save(contract);
        auditService.logChange("Contract", contractId, oldStatus, newStatus);
        eventPublisher.publishEvent(new ContractStatusChangedEvent(this, saved));

        log.info("Contract {} transitioned from {} to {}", contractId, oldStatus, newStatus);
        return saved;
    }

    private void validateTransition(ContractStatus from, ContractStatus to) {
        if (!from.canTransitionTo(to)) {
            throw new InvalidContractTransitionException(from, to);
        }
    }
}
```

### Rule AD05: Services Never Call Other Services Directly (Prefer Events)

**Requirement Level**: SHOULD

When one business action triggers another (e.g., contract activation generates an invoice), prefer domain events over direct service-to-service calls. This keeps services decoupled and independently testable.

**Preferred — event-driven:**
```java
// ContractService
eventPublisher.publishEvent(new ContractActivatedEvent(this, contract));

// InvoiceService
@EventListener
public void onContractActivated(ContractActivatedEvent event) {
    generateInvoice(event.getContract());
}
```

**Acceptable — direct call when events add unnecessary complexity:**
```java
// For simple CRUD orchestration within the same bounded context
public ClientDashboard getDashboard(String clientId) {
    Client client = clientRepository.findById(clientId).orElseThrow(...);
    List<Contract> contracts = contractRepository.findByClientId(clientId);
    List<Invoice> invoices = invoiceRepository.findByClientId(clientId);
    return buildDashboard(client, contracts, invoices);
}
```

### Rule AD06: Use Custom Exceptions for Domain Errors

**Requirement Level**: MUST

Define typed exceptions for business rule violations:

```java
public class EntityNotFoundException extends RuntimeException {
    public EntityNotFoundException(String entityType, String id) {
        super(String.format("%s not found with id: %s", entityType, id));
    }
}

public class InvalidContractTransitionException extends RuntimeException {
    public InvalidContractTransitionException(ContractStatus from, ContractStatus to) {
        super(String.format("Invalid transition from %s to %s", from, to));
    }
}

public class BusinessRuleViolationException extends RuntimeException {
    public BusinessRuleViolationException(String rule, String detail) {
        super(String.format("Business rule violated [%s]: %s", rule, detail));
    }
}
```

Map exceptions to HTTP status codes in `@RestControllerAdvice`:

| Exception | HTTP Status |
|-----------|-------------|
| `EntityNotFoundException` | 404 Not Found |
| `InvalidContractTransitionException` | 409 Conflict |
| `BusinessRuleViolationException` | 422 Unprocessable Entity |
| `DuplicateEntityException` | 409 Conflict |
| `MethodArgumentNotValidException` | 400 Bad Request |
| `IllegalArgumentException` | 400 Bad Request |

---

## Repository Layer

### Rule AD07: Repositories Are Interfaces

**Requirement Level**: MUST

Use Spring Data repository interfaces. Add custom query methods following Spring Data naming conventions:

```java
public interface ContractRepository extends MongoRepository<Contract, String> {

    List<Contract> findByClientId(String clientId);

    List<Contract> findByStatus(ContractStatus status);

    @Query("{ 'endDate': { $lte: ?0 }, 'status': 'ACTIVE' }")
    List<Contract> findRenewalsDueBefore(LocalDate date);

    Optional<Contract> findByContractNumber(String contractNumber);

    long countByClientIdAndStatus(String clientId, ContractStatus status);
}
```

### Rule AD08: Use MongoDB Document Patterns Consistently

**Requirement Level**: MUST

```java
@Document(collection = "contracts")
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class Contract {

    @Id
    private String id;

    @Indexed
    private String clientId;

    @Indexed(unique = true)
    private String contractNumber;

    private ContractStatus status;

    private Money value;
    private LocalDate startDate;
    private LocalDate endDate;
    private List<LineItem> lineItems;
    private List<StatusChange> statusHistory;

    @CreatedDate
    private Instant createdAt;

    @LastModifiedDate
    private Instant updatedAt;
}
```

- Use `String` for `@Id` fields (MongoDB ObjectId as string)
- Add `@Indexed` on fields used in queries
- Use `@CreatedDate` / `@LastModifiedDate` with `@EnableMongoAuditing`
- Embed value objects directly (no separate collections for `Money`, `Address`, etc.)
- Use `List<>` for embedded collections, not `Set<>`

---

## Cross-Cutting Concerns

### Rule AD09: Centralize Exception Handling

**Requirement Level**: MUST

A single `@RestControllerAdvice` class handles all exception-to-response mapping:

```java
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {

    @ExceptionHandler(EntityNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(EntityNotFoundException ex) {
        log.warn("Entity not found: {}", ex.getMessage());
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
                .body(new ErrorResponse(HttpStatus.NOT_FOUND.value(), HttpStatus.NOT_FOUND.getReasonPhrase(), ex.getMessage()));
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException ex) {
        String details = ex.getBindingResult().getFieldErrors().stream()
                .map(e -> e.getField() + ": " + e.getDefaultMessage())
                .collect(Collectors.joining(", "));
        return ResponseEntity.badRequest()
                .body(new ErrorResponse(HttpStatus.BAD_REQUEST.value(), HttpStatus.BAD_REQUEST.getReasonPhrase(), details));
    }
}
```

### Rule AD10: Use Domain Events for Side Effects

**Requirement Level**: SHOULD

Side effects (audit logging, notifications, cache invalidation) should be triggered by domain events, not embedded in the service method:

```java
// Domain event
public class ContractStatusChangedEvent extends ApplicationEvent {
    private final Contract contract;
    private final ContractStatus previousStatus;

    // constructor...
}

// Listener in a separate service
@Component
public class ContractEventListener {

    @EventListener
    public void onStatusChanged(ContractStatusChangedEvent event) {
        // audit, notify, update dashboards, etc.
    }
}
```

---

## Enforcement Checklist

Before submitting code, verify:

- [ ] Every class belongs to exactly one architectural layer
- [ ] Controllers contain zero business logic
- [ ] Services do not call repositories from other bounded contexts directly
- [ ] Custom exceptions are used for all domain errors (no raw `RuntimeException`)
- [ ] Exception handler maps every custom exception to a proper HTTP status
- [ ] DTOs are separate from domain entities — no `@Document` class in API responses
- [ ] Repository interfaces use Spring Data conventions
- [ ] MongoDB documents use `@Indexed` on query fields and `@CreatedDate`/`@LastModifiedDate`
- [ ] REST endpoints follow `/api/v1/{resources}` pattern with correct HTTP methods and status codes
- [ ] Domain events are used for cross-service side effects

## References

- For complete code examples, see [examples.md](examples.md)
- Spring Boot Reference: https://docs.spring.io/spring-boot/reference/
- Spring Data MongoDB: https://docs.spring.io/spring-data/mongodb/reference/
- Domain-Driven Design (Eric Evans): Entities, Value Objects, Aggregates, Domain Events
