---
name: Integration Testing Standards
description: Integration test patterns with Spring Boot Test, Testcontainers (MongoDB), REST API testing, and database verification for end-to-end workflow validation
when_to_apply: When writing integration tests that verify component interactions, REST API behavior, or database operations with real infrastructure
version: 1.0.0
languages: Java, Kotlin
globs: "src/test/**/*IntegrationTest.{java,kt}"
alwaysApply: false
---

# Integration Testing Standards

## Overview

Integration tests verify that components work together correctly with real infrastructure — a real MongoDB instance, Spring's dependency injection, HTTP request handling, and actual repository queries. Unlike unit tests that mock dependencies, integration tests boot the application context and exercise the full request-response cycle. This skill defines how to structure, configure, and write integration tests using Spring Boot Test and Testcontainers.

**Core principles:**
- Test with real infrastructure, not mocks
- Exercise the full HTTP request-response cycle
- Clean database state between tests
- Verify both API responses and database state
- Name tests to describe operation, scenario, and expected result

## When to Apply This Skill

- Writing tests for REST API endpoints (full request-response cycle)
- Testing repository queries against a real MongoDB instance
- Verifying that Spring wiring, validation, and exception handling work end-to-end
- Testing workflows that span multiple services
- Validating database state after operations

## Quick Reference

| Rule | Requirement | Level | Key Point |
|------|------------|-------|-----------|
| IT01 | Reusable base class | MUST | @SpringBootTest, @Testcontainers, MongoDBContainer |
| IT02 | Clean up between tests | MUST | @BeforeEach drops all collections |
| IT03 | Test full HTTP cycle | MUST | Real HTTP requests via TestRestTemplate, verify status + body |
| IT04 | Test error responses | MUST | Verify proper status codes for invalid requests |
| IT05 | Test CRUD end-to-end | SHOULD | Create, read, update, delete lifecycle |
| IT06 | Verify database state | SHOULD | Use MongoTemplate to check stored documents directly |
| IT07 | Test multi-step workflows | SHOULD | Verify state at each step of business workflows |
| IT08 | Follow naming convention | MUST | test_{operation}_{scenario}_{expectedResult} |

---

## Test Infrastructure

### Core Components

| Component | Purpose |
|-----------|---------|
| `@SpringBootTest` | Boots full application context |
| `@Testcontainers` | Manages Docker containers for test infrastructure |
| `MongoDBContainer` | Real MongoDB instance via Testcontainers |
| `TestRestTemplate` / `WebTestClient` | HTTP client for REST API testing |
| `@DynamicPropertySource` | Injects container connection details into Spring config |

### Standard Directory Structure

```
src/test/java/com/enterprise/cms/
    integration/
        BaseIntegrationTest.java        # Abstract base class
        ClientControllerIntegrationTest.java
        ContractLifecycleIntegrationTest.java
        InvoiceIntegrationTest.java
```

---

## Base Integration Test

### Rule IT01: Create a Reusable Base Class

**Requirement Level**: MUST

All integration tests must extend a base class that manages the MongoDB container lifecycle:

```java
package com.enterprise.cms.integration;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.web.client.TestRestTemplate;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.containers.MongoDBContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
public abstract class BaseIntegrationTest {

    @Container
    static MongoDBContainer mongoDBContainer = new MongoDBContainer("mongo:7.0")
            .withReuse(true);

    @Autowired
    protected TestRestTemplate restTemplate;

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.data.mongodb.uri", mongoDBContainer::getReplicaSetUrl);
    }
}
```

**Key decisions:**
- `RANDOM_PORT` avoids port conflicts when running tests in parallel
- `withReuse(true)` keeps the container alive across test classes for speed
- `@DynamicPropertySource` injects the real MongoDB URI into Spring config
- `static` container ensures one instance per test class

### Rule IT02: Clean Up Between Tests

**Requirement Level**: MUST

Each test must start with a clean database state:

```java
@Autowired
private MongoTemplate mongoTemplate;

@BeforeEach
void cleanDatabase() {
    mongoTemplate.getCollectionNames()
            .forEach(name -> mongoTemplate.dropCollection(name));
}
```

Or use `@DirtiesContext` per class (slower but simpler for small test suites).

---

## REST API Testing

### Rule IT03: Test Full HTTP Request-Response Cycle

**Requirement Level**: MUST

Integration tests must go through the HTTP layer — send real HTTP requests and verify responses including status codes, headers, and body:

```java
@Test
void test_createClient_validRequest_returns201WithBody() {
    // Given
    CreateClientRequest request = CreateClientRequest.builder()
            .companyName("Acme Corporation")
            .industry("Technology")
            .annualRevenue(BigDecimal.valueOf(5000000))
            .build();

    // When
    ResponseEntity<ClientResponse> response = restTemplate.postForEntity(
            "/api/v1/clients", request, ClientResponse.class);

    // Then
    assertThat(response.getStatusCode(), is(HttpStatus.CREATED));
    assertThat(response.getBody(), is(notNullValue()));
    assertThat(response.getBody().getCompanyName(), is("Acme Corporation"));
    assertThat(response.getBody().getIndustry(), is("Technology"));
    assertThat(response.getBody().getId(), is(notNullValue()));
}
```

### Rule IT04: Test Error Responses

**Requirement Level**: MUST

Verify that invalid requests return proper error status codes:

```java
@Test
void test_createClient_missingRequiredFields_returns400() {
    // Given
    CreateClientRequest request = CreateClientRequest.builder()
            .industry("Technology")  // companyName is missing
            .build();

    // When
    ResponseEntity<ErrorResponse> response = restTemplate.postForEntity(
            "/api/v1/clients", request, ErrorResponse.class);

    // Then
    assertThat(response.getStatusCode(), is(HttpStatus.BAD_REQUEST));
    assertThat(response.getBody().getMessage(), containsString("companyName"));
}

@Test
void test_getClient_nonExistentId_returns404() {
    // When
    ResponseEntity<ErrorResponse> response = restTemplate.getForEntity(
            "/api/v1/clients/nonexistent-id", ErrorResponse.class);

    // Then
    assertThat(response.getStatusCode(), is(HttpStatus.NOT_FOUND));
}
```

### Rule IT05: Test CRUD Operations End-to-End

**Requirement Level**: SHOULD

Test the complete lifecycle: create → read → update → delete:

```java
@Test
void test_clientCrudLifecycle() {
    // CREATE
    CreateClientRequest createRequest = CreateClientRequest.builder()
            .companyName("Test Corp")
            .industry("Finance")
            .annualRevenue(BigDecimal.valueOf(1000000))
            .build();
    ResponseEntity<ClientResponse> createResponse = restTemplate.postForEntity(
            "/api/v1/clients", createRequest, ClientResponse.class);
    assertThat(createResponse.getStatusCode(), is(HttpStatus.CREATED));
    String clientId = createResponse.getBody().getId();

    // READ
    ResponseEntity<ClientResponse> getResponse = restTemplate.getForEntity(
            "/api/v1/clients/" + clientId, ClientResponse.class);
    assertThat(getResponse.getStatusCode(), is(HttpStatus.OK));
    assertThat(getResponse.getBody().getCompanyName(), is("Test Corp"));

    // UPDATE
    UpdateClientRequest updateRequest = UpdateClientRequest.builder()
            .companyName("Updated Corp")
            .build();
    restTemplate.put("/api/v1/clients/" + clientId, updateRequest);
    ResponseEntity<ClientResponse> updatedResponse = restTemplate.getForEntity(
            "/api/v1/clients/" + clientId, ClientResponse.class);
    assertThat(updatedResponse.getBody().getCompanyName(), is("Updated Corp"));

    // DELETE
    restTemplate.delete("/api/v1/clients/" + clientId);
    ResponseEntity<ErrorResponse> deletedResponse = restTemplate.getForEntity(
            "/api/v1/clients/" + clientId, ErrorResponse.class);
    assertThat(deletedResponse.getStatusCode(), is(HttpStatus.NOT_FOUND));
}
```

---

## Database Verification

### Rule IT06: Verify Database State Directly

**Requirement Level**: SHOULD

For complex operations, verify the database state directly using MongoTemplate:

```java
@Autowired
private MongoTemplate mongoTemplate;

@Test
void test_transitionStatus_updatesDocumentInDatabase() {
    // Given — create a contract via API
    // ... (POST to create contract, extract ID)

    // When — transition status
    restTemplate.postForEntity(
            "/api/v1/contracts/" + contractId + "/transition",
            new ContractTransitionRequest(ContractStatus.PENDING_APPROVAL),
            ContractResponse.class);

    // Then — verify directly in MongoDB
    Contract stored = mongoTemplate.findById(contractId, Contract.class);
    assertThat(stored, is(notNullValue()));
    assertThat(stored.getStatus(), is(ContractStatus.PENDING_APPROVAL));
    assertThat(stored.getStatusHistory(), hasSize(1));
    assertThat(stored.getStatusHistory().get(0).getFromStatus(), is(ContractStatus.DRAFT));
}
```

---

## Workflow Testing

### Rule IT07: Test Multi-Step Business Workflows

**Requirement Level**: SHOULD

For features involving multiple API calls and state transitions:

```java
@Test
void test_contractLifecycle_draftToActiveToRenewal() {
    // STEP 1: Create client
    ResponseEntity<ClientResponse> clientResponse = restTemplate.postForEntity(
            "/api/v1/clients",
            CreateClientRequest.builder()
                    .companyName("Lifecycle Corp").industry("Tech")
                    .annualRevenue(BigDecimal.valueOf(1000000)).build(),
            ClientResponse.class);
    String clientId = clientResponse.getBody().getId();

    // STEP 2: Create contract in DRAFT
    ResponseEntity<ContractResponse> contractResponse = restTemplate.postForEntity(
            "/api/v1/contracts",
            CreateContractRequest.builder()
                    .clientId(clientId).contractNumber("CNT-LC-001")
                    .value(BigDecimal.valueOf(50000)).currency("USD")
                    .startDate(LocalDate.now()).endDate(LocalDate.now().plusYears(1))
                    .build(),
            ContractResponse.class);
    String contractId = contractResponse.getBody().getId();
    assertThat(contractResponse.getBody().getStatus(), is("DRAFT"));

    // STEP 3: Transition DRAFT -> PENDING_APPROVAL
    transition(contractId, ContractStatus.PENDING_APPROVAL, HttpStatus.OK);

    // STEP 4: Transition PENDING_APPROVAL -> ACTIVE
    transition(contractId, ContractStatus.ACTIVE, HttpStatus.OK);

    // STEP 5: Transition ACTIVE -> RENEWAL
    ResponseEntity<ContractResponse> renewalResponse =
            transition(contractId, ContractStatus.RENEWAL, HttpStatus.OK);
    assertThat(renewalResponse.getBody().getStatus(), is("RENEWAL"));

    // STEP 6: Verify invalid transition is rejected
    ResponseEntity<ErrorResponse> errorResponse = restTemplate.postForEntity(
            "/api/v1/contracts/" + contractId + "/transition",
            new ContractTransitionRequest(ContractStatus.DRAFT),
            ErrorResponse.class);
    assertThat(errorResponse.getStatusCode(), is(HttpStatus.CONFLICT));
}

private ResponseEntity<ContractResponse> transition(
        String contractId, ContractStatus status, HttpStatus expectedStatus) {
    ResponseEntity<ContractResponse> response = restTemplate.postForEntity(
            "/api/v1/contracts/" + contractId + "/transition",
            new ContractTransitionRequest(status),
            ContractResponse.class);
    assertThat(response.getStatusCode(), is(expectedStatus));
    return response;
}
```

---

## Test Naming

### Rule IT08: Follow Integration Test Naming Convention

**Requirement Level**: MUST

Format: `test_{operation}_{scenario}_{expectedResult}`

| Pattern | Example |
|---------|---------|
| API success | `test_createClient_validRequest_returns201WithBody` |
| API validation | `test_createClient_missingName_returns400` |
| API not found | `test_getClient_nonExistentId_returns404` |
| Workflow | `test_contractLifecycle_draftToActiveToRenewal` |
| Database | `test_transitionStatus_updatesDocumentInDatabase` |

---

## Enforcement Checklist

Before submitting integration tests, verify:

- [ ] Test class extends `BaseIntegrationTest` (or equivalent)
- [ ] `@SpringBootTest` with `RANDOM_PORT` is used
- [ ] MongoDB Testcontainer is configured via `@DynamicPropertySource`
- [ ] Database is cleaned between tests (`@BeforeEach`)
- [ ] Tests go through the full HTTP layer (no direct service calls)
- [ ] Success responses verify status code AND body content
- [ ] Error responses verify status code AND error message
- [ ] CRUD lifecycle is tested end-to-end for new entities
- [ ] Multi-step workflows verify state at each step
- [ ] Database state is verified directly for complex operations
- [ ] Test names follow `test_{operation}_{scenario}_{result}` convention
- [ ] No hardcoded ports or database URIs

## References

- For complete code examples, see [examples.md](examples.md)
- Spring Boot Testing: https://docs.spring.io/spring-boot/reference/testing/
- Testcontainers Java: https://java.testcontainers.org/
- Testcontainers MongoDB Module: https://java.testcontainers.org/modules/databases/mongodb/
