---
name: Unit Testing Standards
description: Unit test patterns with JUnit 5, Mockito, and Hamcrest — covers test structure, mocking, assertions, test data builders, and naming conventions
when_to_apply: When writing unit tests for services, repositories, mappers, or any component tested in isolation
version: 1.0.0
languages: Java, Kotlin
globs: "src/test/**/*Test.{java,kt}"
alwaysApply: false
---

# Unit Testing Standards

## Overview

Unit tests verify individual components in isolation using mocks for external dependencies. Every service, mapper, and non-trivial utility must have corresponding unit tests. This skill defines the test structure, mocking patterns, assertion conventions, and test data strategies that produce reliable, readable, and maintainable tests.

**Core principles:**
- Test every public method with at least one test, covering both success and error paths
- Use MockitoExtension with @Mock and @InjectMocks for consistent test setup
- Follow Given-When-Then structure with explicit section comments
- Use test data builders with sensible defaults to keep tests focused
- Use parameterized tests for multiple input scenarios with the same assertion pattern

## When to Apply This Skill

- Writing unit tests for new or modified services
- Testing business logic in isolation from databases and external systems
- Mocking dependencies with Mockito
- Creating test data builders for complex domain objects
- Testing mappers for correct field mapping
- Testing exception scenarios and edge cases

## Quick Reference

| Rule | Requirement | Level | Key Point |
|------|------------|-------|-----------|
| UT01 | Use MockitoExtension | MUST | @ExtendWith(MockitoExtension.class) on every test class |
| UT02 | Use @InjectMocks for class under test | MUST | Dependencies use @Mock, never manual instantiation |
| UT03 | Follow Given-When-Then structure | MUST | Three sections with comments in every test |
| UT04 | Use descriptive test names | MUST | Format: test_{method}_{scenario}_{result} |
| UT05 | Use when().thenReturn() for stubbing | MUST | thenReturn, thenAnswer, or thenThrow for mocks |
| UT06 | Use verify() for interaction verification | MUST | verify, times, never, verifyNoMoreInteractions |
| UT07 | Use ArgumentCaptor for complex verification | SHOULD | Capture and inspect arguments passed to mocks |
| UT08 | Prefer Hamcrest for readable assertions | SHOULD | is, equalTo, hasSize, contains, allOf matchers |
| UT09 | Use JUnit 5 for exception assertions | MUST | assertThrows with message verification |
| UT10 | Use fluent builder pattern for test data | SHOULD | Builder classes with sensible defaults |
| UT11 | Provide sensible defaults | SHOULD | withDefaults() method, override only relevant fields |
| UT12 | Use @ParameterizedTest for multiple scenarios | SHOULD | @MethodSource for complex, @ValueSource for simple |
| UT13 | Use @ValueSource for simple cases | MAY | Strings, ints; combine with @NullAndEmptySource |

---

## Testing Framework

| Framework | Purpose |
|-----------|---------|
| JUnit 5 (Jupiter) | Core testing framework |
| Mockito | Mocking and stubbing dependencies |
| Hamcrest | Readable assertion matchers |
| AssertJ | Fluent assertions (alternative to Hamcrest) |

---

## Test Class Structure

### Rule UT01: Use MockitoExtension

**Requirement Level**: MUST

Every unit test class must use `@ExtendWith(MockitoExtension.class)`:

```java
@ExtendWith(MockitoExtension.class)
class ContractServiceTest {

    @Mock
    private ContractRepository contractRepository;

    @Mock
    private ApplicationEventPublisher eventPublisher;

    @InjectMocks
    private ContractService contractService;

    // tests...
}
```

### Rule UT02: Use @InjectMocks for Class Under Test

**Requirement Level**: MUST

- The class under test uses `@InjectMocks`
- All its dependencies use `@Mock`
- Never instantiate the class under test manually when using Mockito

### Rule UT03: Follow Given-When-Then Structure

**Requirement Level**: MUST

Every test must have three clearly separated sections with comments:

```java
@Test
void test_transitionStatus_validTransition_updatesStatusAndPublishesEvent() {
    // Given
    Contract contract = ContractBuilder.newBuilder()
            .withId("contract-1")
            .withStatus(ContractStatus.DRAFT)
            .withDefaults()
            .build();
    when(contractRepository.findById("contract-1")).thenReturn(Optional.of(contract));
    when(contractRepository.save(any(Contract.class))).thenAnswer(i -> i.getArgument(0));

    // When
    Contract result = contractService.transitionStatus("contract-1", ContractStatus.PENDING_APPROVAL);

    // Then
    assertThat(result.getStatus(), is(ContractStatus.PENDING_APPROVAL));
    assertThat(result.getStatusHistory(), hasSize(1));
    verify(contractRepository).save(contract);
    verify(eventPublisher).publishEvent(any(ContractStatusChangedEvent.class));
}
```

---

## Test Naming

### Rule UT04: Use Descriptive Test Names

**Requirement Level**: MUST

Format: `test_{methodName}_{scenario}_{expectedResult}`

| Pattern | Example |
|---------|---------|
| Success case | `test_create_validContract_returnsCreatedContract` |
| Error case | `test_create_duplicateNumber_throwsDuplicateException` |
| Null handling | `test_getById_nonExistentId_throwsNotFoundException` |
| Edge case | `test_calculateScore_noInvoices_returnsMaxScore` |
| Boundary | `test_calculateScore_allOverdue_returnsZero` |
| State transition | `test_transitionStatus_draftToPending_succeeds` |
| Invalid transition | `test_transitionStatus_expiredToActive_throwsException` |

---

## Mocking Patterns

### Rule UT05: Use when().thenReturn() for Stubbing

**Requirement Level**: MUST

```java
// Simple return value
when(contractRepository.findById("id-1")).thenReturn(Optional.of(contract));

// Return argument (for save operations)
when(contractRepository.save(any(Contract.class))).thenAnswer(i -> i.getArgument(0));

// Multiple calls return different values
when(contractRepository.findById("id-1"))
        .thenReturn(Optional.of(contractV1))
        .thenReturn(Optional.of(contractV2));

// Throw exception
when(contractRepository.findById("bad-id")).thenReturn(Optional.empty());
```

### Rule UT06: Use verify() for Interaction Verification

**Requirement Level**: MUST

Verify that the class under test interacts correctly with its dependencies:

```java
// Verify method was called
verify(contractRepository).save(contract);

// Verify called exactly N times
verify(contractRepository, times(1)).save(any());

// Verify never called
verify(eventPublisher, never()).publishEvent(any());

// Verify no more interactions
verifyNoMoreInteractions(contractRepository);

// Verify no interactions at all
verifyNoInteractions(auditService);
```

### Rule UT07: Use ArgumentCaptor for Complex Verification

**Requirement Level**: SHOULD

When you need to verify the content of arguments passed to mocked methods:

```java
@Captor
private ArgumentCaptor<ContractStatusChangedEvent> eventCaptor;

@Test
void test_transitionStatus_publishesEventWithCorrectData() {
    // Given
    Contract contract = ContractBuilder.newBuilder()
            .withId("c-1").withStatus(ContractStatus.DRAFT).withDefaults().build();
    when(contractRepository.findById("c-1")).thenReturn(Optional.of(contract));
    when(contractRepository.save(any())).thenAnswer(i -> i.getArgument(0));

    // When
    contractService.transitionStatus("c-1", ContractStatus.PENDING_APPROVAL);

    // Then
    verify(eventPublisher).publishEvent(eventCaptor.capture());
    ContractStatusChangedEvent event = eventCaptor.getValue();
    assertThat(event.getContract().getId(), is("c-1"));
    assertThat(event.getPreviousStatus(), is(ContractStatus.DRAFT));
}
```

---

## Assertions

### Rule UT08: Prefer Hamcrest for Readable Assertions

**Requirement Level**: SHOULD

```java
import static org.hamcrest.MatcherAssert.assertThat;
import static org.hamcrest.Matchers.*;

// Equality
assertThat(result.getStatus(), is(ContractStatus.ACTIVE));
assertThat(result.getCompanyName(), equalTo("Acme Corp"));

// Null checks
assertThat(result, is(notNullValue()));
assertThat(result.getDeletedAt(), is(nullValue()));

// Collections
assertThat(result.getLineItems(), hasSize(3));
assertThat(result.getTags(), contains("enterprise", "premium"));
assertThat(result.getTags(), hasItem("enterprise"));
assertThat(result.getContracts(), is(empty()));

// Strings
assertThat(result.getContractNumber(), startsWith("CNT-"));
assertThat(result.getDescription(), containsString("renewal"));

// Numbers
assertThat(result.getHealthScore(), greaterThan(50));
assertThat(result.getTotal(), closeTo(BigDecimal.valueOf(100.00), BigDecimal.valueOf(0.01)));

// Combined
assertThat(result, allOf(
        hasProperty("status", is(ContractStatus.ACTIVE)),
        hasProperty("clientId", is("client-1"))
));
```

### Rule UT09: Use JUnit 5 for Exception Assertions

**Requirement Level**: MUST

```java
@Test
void test_getById_nonExistentId_throwsEntityNotFoundException() {
    // Given
    when(contractRepository.findById("bad-id")).thenReturn(Optional.empty());

    // When / Then
    EntityNotFoundException exception = assertThrows(
            EntityNotFoundException.class,
            () -> contractService.getById("bad-id"));

    assertThat(exception.getMessage(), containsString("bad-id"));
    assertThat(exception.getMessage(), containsString("Contract"));
}
```

---

## Test Data Builders

### Rule UT10: Use Fluent Builder Pattern for Test Data

**Requirement Level**: SHOULD

Create builder classes that produce domain objects with sensible defaults:

```java
public class ContractBuilder {

    private String id;
    private String clientId;
    private String contractNumber;
    private ContractStatus status = ContractStatus.DRAFT;
    private Money value;
    private LocalDate startDate;
    private LocalDate endDate;
    private List<LineItem> lineItems = new ArrayList<>();
    private List<StatusChange> statusHistory = new ArrayList<>();

    public static ContractBuilder newBuilder() {
        return new ContractBuilder();
    }

    public ContractBuilder withId(String id) {
        this.id = id;
        return this;
    }

    public ContractBuilder withClientId(String clientId) {
        this.clientId = clientId;
        return this;
    }

    public ContractBuilder withStatus(ContractStatus status) {
        this.status = status;
        return this;
    }

    public ContractBuilder withDefaults() {
        this.id = "contract-" + UUID.randomUUID().toString().substring(0, 8);
        this.clientId = "client-default";
        this.contractNumber = "CNT-" + System.currentTimeMillis();
        this.value = new Money(BigDecimal.valueOf(10000), "USD");
        this.startDate = LocalDate.now();
        this.endDate = LocalDate.now().plusYears(1);
        return this;
    }

    public Contract build() {
        return Contract.builder()
                .id(id)
                .clientId(clientId)
                .contractNumber(contractNumber)
                .status(status)
                .value(value)
                .startDate(startDate)
                .endDate(endDate)
                .lineItems(lineItems)
                .statusHistory(statusHistory)
                .build();
    }
}
```

### Rule UT11: Provide Sensible Defaults

**Requirement Level**: SHOULD

Every builder should have a `withDefaults()` method that sets all required fields to valid, non-null values. Tests should only override the specific fields relevant to the test scenario:

```java
// Only set what matters for this test
Contract contract = ContractBuilder.newBuilder()
        .withDefaults()
        .withStatus(ContractStatus.EXPIRED)  // this is what we're testing
        .build();
```

---

## Parameterized Tests

### Rule UT12: Use @ParameterizedTest for Multiple Scenarios

**Requirement Level**: SHOULD

When testing multiple inputs with the same assertion pattern:

```java
@ParameterizedTest
@MethodSource("validTransitions")
void test_transitionStatus_validTransitions_succeeds(
        ContractStatus from, ContractStatus to) {
    // Given
    Contract contract = ContractBuilder.newBuilder()
            .withDefaults().withStatus(from).build();
    when(contractRepository.findById(any())).thenReturn(Optional.of(contract));
    when(contractRepository.save(any())).thenAnswer(i -> i.getArgument(0));

    // When
    Contract result = contractService.transitionStatus(contract.getId(), to);

    // Then
    assertThat(result.getStatus(), is(to));
}

private static Stream<Arguments> validTransitions() {
    return Stream.of(
            Arguments.of(ContractStatus.DRAFT, ContractStatus.PENDING_APPROVAL),
            Arguments.of(ContractStatus.PENDING_APPROVAL, ContractStatus.ACTIVE),
            Arguments.of(ContractStatus.ACTIVE, ContractStatus.RENEWAL),
            Arguments.of(ContractStatus.ACTIVE, ContractStatus.TERMINATED),
            Arguments.of(ContractStatus.RENEWAL, ContractStatus.ACTIVE)
    );
}

@ParameterizedTest
@MethodSource("invalidTransitions")
void test_transitionStatus_invalidTransitions_throwsException(
        ContractStatus from, ContractStatus to) {
    // Given
    Contract contract = ContractBuilder.newBuilder()
            .withDefaults().withStatus(from).build();
    when(contractRepository.findById(any())).thenReturn(Optional.of(contract));

    // When / Then
    assertThrows(InvalidContractTransitionException.class,
            () -> contractService.transitionStatus(contract.getId(), to));
}

private static Stream<Arguments> invalidTransitions() {
    return Stream.of(
            Arguments.of(ContractStatus.DRAFT, ContractStatus.ACTIVE),
            Arguments.of(ContractStatus.EXPIRED, ContractStatus.ACTIVE),
            Arguments.of(ContractStatus.TERMINATED, ContractStatus.DRAFT)
    );
}
```

### Rule UT13: Use @ValueSource for Simple Cases

**Requirement Level**: MAY

```java
@ParameterizedTest
@ValueSource(strings = {"", "  ", "\t"})
void test_create_blankCompanyName_throwsException(String name) {
    CreateClientRequest request = CreateClientRequest.builder()
            .companyName(name).build();

    assertThrows(ConstraintViolationException.class,
            () -> clientService.create(request));
}

@ParameterizedTest
@NullAndEmptySource
@ValueSource(strings = {"  "})
void test_create_invalidCompanyName_throwsException(String name) {
    // covers null, "", and whitespace-only
}
```

---

## Test Directory Structure

```
src/test/java/com/enterprise/cms/
    service/
        ContractServiceTest.java
        ClientServiceTest.java
        HealthScoreServiceTest.java
    mapper/
        ContractMapperTest.java
        ClientMapperTest.java
    controller/
        ContractControllerTest.java    # @WebMvcTest (optional)
    testdata/
        ContractBuilder.java
        ClientBuilder.java
        InvoiceBuilder.java
```

---

## Enforcement Checklist

Before submitting unit tests, verify:

- [ ] Test class uses `@ExtendWith(MockitoExtension.class)`
- [ ] Class under test uses `@InjectMocks`
- [ ] All dependencies use `@Mock`
- [ ] Test methods follow `test_{method}_{scenario}_{result}` naming
- [ ] Tests follow Given-When-Then structure with comments
- [ ] Assertions use Hamcrest matchers or AssertJ fluent assertions
- [ ] Exception tests use `assertThrows()` and verify the message
- [ ] Mock interactions verified with `verify()`
- [ ] Test data uses builder pattern with sensible defaults
- [ ] Parameterized tests used for multiple input scenarios
- [ ] No actual external calls (database, network, file system)
- [ ] Each public method has at least one test
- [ ] Error paths and edge cases are covered

## References

- For complete code examples, see [examples.md](examples.md)
- JUnit 5 User Guide: https://junit.org/junit5/docs/current/user-guide/
- Mockito Documentation: https://javadoc.io/doc/org.mockito/mockito-core/latest/
- Hamcrest Matchers: https://hamcrest.org/JavaHamcrest/javadoc/
