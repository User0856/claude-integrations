---
name: Integration Testing Standards
description: Integration test patterns with Spring Boot Test, Testcontainers (MongoDB), REST API testing, and database verification for end-to-end workflow validation
when_to_apply: When writing integration tests that verify component interactions, REST API behavior, or database operations with real infrastructure
version: 2.0.0
languages: Java, Kotlin
globs: "src/integrationTest/**/*IntegrationTest.{java,kt}"
alwaysApply: false
---

# Integration Testing Standards

## Overview

Integration tests verify that components work together correctly with real infrastructure — a real MongoDB instance via Testcontainers, Spring's dependency injection, HTTP request handling via MockMvc, and actual repository queries. Unlike unit tests that mock dependencies, integration tests boot the application context and exercise the full request-response cycle.

**Core principles:**
- Test with real infrastructure via Testcontainers, not mocks
- Use MockMvc with @AutoConfigureMockMvc to test the HTTP layer without a real server
- Share a single MongoDB container across all test classes via a static singleton
- Build request JSON manually (not via ObjectMapper) to test the actual API contract
- Clean database state between tests by dropping all collections
- Verify both API responses (via jsonPath) and database state (via MongoTemplate)

## When to Apply This Skill

- Writing tests for REST API endpoints (full request-response cycle)
- Testing repository queries against a real MongoDB instance
- Verifying that Spring wiring, validation, and exception handling work end-to-end
- Testing workflows that span multiple services
- Validating database state after operations

## Quick Reference

| Rule | Requirement | Level | Key Point |
|------|-----------|-------|-----------|
| IT01 | Use static singleton container | MUST | Start MongoDB once in static block, NOT with @Container |
| IT02 | Use @ServiceConnection | MUST | Auto-configures spring.data.mongodb.uri, replaces @DynamicPropertySource |
| IT03 | Use MockMvc, not TestRestTemplate | MUST | @AutoConfigureMockMvc, no RANDOM_PORT needed |
| IT04 | Clean up between tests | MUST | @BeforeEach drops all collections via MongoTemplate |
| IT05 | Build JSON manually for requests | MUST | Raw JSON strings or text blocks, avoid ObjectMapper for request bodies |
| IT06 | Use jsonPath for response verification | MUST | Hamcrest matchers with $.field paths |
| IT07 | Test error responses with status and body | MUST | Verify status code, error message content |
| IT08 | Test CRUD end-to-end | SHOULD | Create, read, update, delete lifecycle |
| IT09 | Verify database state directly | SHOULD | MongoTemplate.findById after API calls |
| IT10 | Test multi-step workflows | SHOULD | State at each step of business workflows |
| IT11 | Follow naming convention | MUST | test_{operation}_{scenario}_{expectedResult} |

---

## Test Infrastructure

### Core Components

| Component | Purpose |
|-----------|---------|
| `@SpringBootTest` | Boots full application context |
| `@AutoConfigureMockMvc` | Configures MockMvc for HTTP testing without a real server |
| `@Testcontainers` | Enables Testcontainers JUnit 5 extension |
| `MongoDBContainer` (static singleton) | Real MongoDB instance shared across all test classes |
| `@ServiceConnection` | Auto-configures `spring.data.mongodb.uri` from the container |
| `MockMvc` | Performs HTTP requests against the application |
| `MongoTemplate` | Direct database access for verification and cleanup |
| `ObjectMapper` (tools.jackson) | Reads JSON responses, extracts IDs from response bodies |

### Standard Directory Structure

```
src/integrationTest/java/com/enterprise/cms/{service}/integration/
    BaseIntegrationTest.java                    # Abstract base class — ONE per service
    ClientControllerIntegrationTest.java        # Tests for client endpoints
    ContactControllerIntegrationTest.java       # Tests for contact endpoints
    {Feature}IntegrationTest.java               # Tests for new feature endpoints
```

Integration tests live in `src/integrationTest/java` (separate source set configured in `build.gradle`), NOT in `src/test/java`.

---

## Base Integration Test

### Rule IT01: Use a Static Singleton Container (NOT @Container)

**Requirement Level**: MUST

The MongoDB container must be started ONCE in a `static {}` block and shared across ALL test classes. **Do NOT use `@Container`** — it creates one container per test class, causing port conflicts when Spring caches the application context.

### Rule IT02: Use @ServiceConnection (NOT @DynamicPropertySource)

**Requirement Level**: MUST

Spring Boot 3.1+ / 4.x provides `@ServiceConnection` which auto-configures `spring.data.mongodb.uri` from the Testcontainers container. **Do NOT use `@DynamicPropertySource`** — it's the old approach and doesn't integrate with context caching.

### Rule IT03: Use MockMvc (NOT TestRestTemplate)

**Requirement Level**: MUST

Use `@AutoConfigureMockMvc` with `MockMvc`. **Do NOT use `TestRestTemplate`** with `RANDOM_PORT` — it requires a real server, is slower, and error responses are harder to inspect.

```java
package com.enterprise.cms.client.integration;

import org.junit.jupiter.api.BeforeEach;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.testcontainers.service.connection.ServiceConnection;
import org.springframework.boot.webmvc.test.autoconfigure.AutoConfigureMockMvc;
import org.springframework.data.mongodb.core.MongoTemplate;
import org.springframework.test.web.servlet.MockMvc;
import org.testcontainers.containers.MongoDBContainer;
import org.testcontainers.junit.jupiter.Testcontainers;

import java.time.Duration;

@SpringBootTest
@AutoConfigureMockMvc
@Testcontainers
public abstract class BaseIntegrationTest {

    @ServiceConnection
    static MongoDBContainer mongoDBContainer = new MongoDBContainer("mongo:7.0")
            .withStartupTimeout(Duration.ofMinutes(3));

    static {
        mongoDBContainer.start();
    }

    @Autowired
    protected MockMvc mockMvc;

    @Autowired
    protected MongoTemplate mongoTemplate;

    @BeforeEach
    void cleanDatabase() {
        mongoTemplate.getCollectionNames()
                .forEach(name -> mongoTemplate.dropCollection(name));
    }
}
```

**Why this works:**
- `static {}` block starts the container once, before any test class loads
- `@ServiceConnection` on the static field auto-configures Spring to connect to this container
- `@SpringBootTest` (without `RANDOM_PORT`) uses a mock servlet environment — fast, no port conflicts
- `@AutoConfigureMockMvc` provides `MockMvc` for HTTP testing
- `MongoTemplate` is injected for direct database access (cleanup + verification)
- `@BeforeEach cleanDatabase()` drops all collections so each test starts clean

**Common mistakes that cause MongoTimeoutException:**
- Using `@Container` instead of manual `static {}` start — when multiple test classes run, each class tries to manage the container lifecycle independently, causing the cached Spring context to lose its MongoDB connection
- Using `@DynamicPropertySource` instead of `@ServiceConnection` — the property may not propagate correctly to cached contexts

### Rule IT04: Clean Up Between Tests

**Requirement Level**: MUST

Each test must start with a clean database. The base class handles this via `@BeforeEach`:

```java
@BeforeEach
void cleanDatabase() {
    mongoTemplate.getCollectionNames()
            .forEach(name -> mongoTemplate.dropCollection(name));
}
```

This drops ALL collections — simple and reliable. Do NOT use `@DirtiesContext` (too slow, restarts the entire Spring context).

---

## Writing Request JSON

### Rule IT05: Build JSON Manually (NOT via ObjectMapper)

**Requirement Level**: MUST

**Always use raw JSON strings (text blocks) for request bodies.** Do NOT serialize DTOs with `ObjectMapper.writeValueAsString()`.

**Why:** Spring Boot 4.x uses Jackson 3.x (`tools.jackson`), which has different boolean serialization than Jackson 2.x. Specifically, Lombok's `boolean isPrimary` field generates getter `isPrimary()`, which Jackson serializes as JSON key `"primary"` (strips the "is" prefix). If you use ObjectMapper to serialize the DTO, the JSON key will be `"primary"`, but the API contract expects `"isPrimary"`. This mismatch causes `HttpMessageNotReadableException: Cannot map null into type boolean`.

**Correct — raw JSON:**
```java
String json = """
        {
            "firstName": "John",
            "lastName": "Doe",
            "email": "john@example.com",
            "role": "TECHNICAL",
            "isPrimary": true
        }
        """;

mockMvc.perform(post("/api/v1/clients/" + clientId + "/contacts")
        .contentType(MediaType.APPLICATION_JSON)
        .content(json))
        .andExpect(status().isCreated());
```

**Correct — parameterized helper method:**
```java
private String contactJson(String firstName, String lastName, String email,
                           String role, boolean isPrimary) {
    return """
            {
                "firstName": "%s",
                "lastName": "%s",
                "email": "%s",
                "role": "%s",
                "isPrimary": %s
            }
            """.formatted(firstName, lastName, email, role, isPrimary);
}
```

**WRONG — ObjectMapper serialization (causes boolean field issues):**
```java
// DO NOT DO THIS — Jackson 3.x serializes boolean isPrimary as "primary"
CreateContactRequest request = CreateContactRequest.builder()
        .firstName("John").lastName("Doe").isPrimary(true).build();
mockMvc.perform(post(url)
        .content(objectMapper.writeValueAsString(request)))  // WRONG
```

**Exception for DTOs WITHOUT boolean primitives:** If the DTO has no `boolean` fields (only `Boolean` wrapper or no booleans at all), ObjectMapper serialization is safe. For example, `CreateClientRequest` (no boolean fields) works fine with ObjectMapper. But always prefer raw JSON for consistency and to test the real API contract.

**Reading responses IS safe with ObjectMapper:**
```java
// This is fine — reading response JSON to extract fields
@Autowired
private ObjectMapper objectMapper;

String responseBody = mockMvc.perform(post("/api/v1/clients")
        .contentType(MediaType.APPLICATION_JSON)
        .content(clientJson("Acme Corp", "Technology", 5000000)))
        .andReturn().getResponse().getContentAsString();

String clientId = objectMapper.readTree(responseBody).get("id").asText();
```

**Boolean fields in responses:** When reading boolean fields from JSON responses via jsonPath, use the Jackson-serialized name (`"primary"`, not `"isPrimary"`):
```java
// Correct — Jackson serializes boolean isPrimary as "primary" in response
.andExpect(jsonPath("$.primary", is(true)))

// WRONG — field name doesn't match Jackson's serialization
.andExpect(jsonPath("$.isPrimary", is(true)))
```

---

## Response Verification

### Rule IT06: Use jsonPath for Response Verification

**Requirement Level**: MUST

Use Spring's `jsonPath()` with Hamcrest matchers:

```java
mockMvc.perform(get("/api/v1/clients/" + clientId))
        .andExpect(status().isOk())
        .andExpect(jsonPath("$.companyName", is("Acme Corp")))
        .andExpect(jsonPath("$.status", is("PROSPECT")))
        .andExpect(jsonPath("$.id", is(notNullValue())))
        .andExpect(jsonPath("$.createdAt", is(notNullValue())));
```

**Common jsonPath patterns:**

```java
// Single field
.andExpect(jsonPath("$.companyName", is("Acme Corp")))

// Null check
.andExpect(jsonPath("$.id", is(notNullValue())))

// Array size
.andExpect(jsonPath("$", hasSize(3)))

// Array element
.andExpect(jsonPath("$[0].companyName", is("First Corp")))

// Nested field
.andExpect(jsonPath("$.address.city", is("New York")))

// Boolean (use Jackson-serialized name for Lombok boolean isPrimary)
.andExpect(jsonPath("$.primary", is(true)))

// String contains
.andExpect(jsonPath("$.message", containsString("not found")))
```

**Required imports:**
```java
import static org.hamcrest.Matchers.*;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;
```

### Rule IT07: Test Error Responses

**Requirement Level**: MUST

Verify that invalid requests return proper HTTP status codes:

```java
// 400 — validation failure
mockMvc.perform(post("/api/v1/clients")
        .contentType(MediaType.APPLICATION_JSON)
        .content("""
                { "industry": "Technology" }
                """))  // missing required companyName
        .andExpect(status().isBadRequest());

// 404 — entity not found
mockMvc.perform(get("/api/v1/clients/nonexistent-id"))
        .andExpect(status().isNotFound());

// 409 — duplicate entity
mockMvc.perform(post("/api/v1/clients")
        .contentType(MediaType.APPLICATION_JSON)
        .content(clientJson("Same Name Corp", "Tech", 100000)))
        .andExpect(status().isCreated());
mockMvc.perform(post("/api/v1/clients")
        .contentType(MediaType.APPLICATION_JSON)
        .content(clientJson("Same Name Corp", "Tech", 100000)))
        .andExpect(status().isConflict());

// 422 — business rule violation
mockMvc.perform(delete("/api/v1/clients/" + clientId + "/contacts/" + primaryContactId))
        .andExpect(status().isUnprocessableEntity());
```

---

## CRUD and Workflow Tests

### Rule IT08: Test CRUD Operations End-to-End

**Requirement Level**: SHOULD

```java
@Test
void test_clientCrudLifecycle() throws Exception {
    // CREATE
    String body = mockMvc.perform(post("/api/v1/clients")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content(clientJson("Lifecycle Corp", "Technology", 5000000)))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.companyName", is("Lifecycle Corp")))
            .andReturn().getResponse().getContentAsString();
    String clientId = objectMapper.readTree(body).get("id").asText();

    // READ
    mockMvc.perform(get("/api/v1/clients/" + clientId))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.companyName", is("Lifecycle Corp")));

    // UPDATE
    mockMvc.perform(put("/api/v1/clients/" + clientId)
                    .contentType(MediaType.APPLICATION_JSON)
                    .content("""
                            { "companyName": "Updated Corp", "industry": "Finance" }
                            """))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.companyName", is("Updated Corp")));

    // DELETE
    mockMvc.perform(delete("/api/v1/clients/" + clientId))
            .andExpect(status().isNoContent());

    // VERIFY DELETED
    mockMvc.perform(get("/api/v1/clients/" + clientId))
            .andExpect(status().isNotFound());
}
```

### Rule IT09: Verify Database State Directly

**Requirement Level**: SHOULD

For complex operations, verify the database state directly using MongoTemplate:

```java
@Test
void test_createClient_persistsFieldsToMongoDB() throws Exception {
    String body = mockMvc.perform(post("/api/v1/clients")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content(clientJson("Persistence Corp", "Technology", 5000000)))
            .andExpect(status().isCreated())
            .andReturn().getResponse().getContentAsString();
    String clientId = objectMapper.readTree(body).get("id").asText();

    // Verify directly in MongoDB
    Client stored = mongoTemplate.findById(clientId, Client.class);
    assertThat(stored, is(notNullValue()));
    assertThat(stored.getCompanyName(), is("Persistence Corp"));
    assertThat(stored.getStatus(), is(ClientStatus.PROSPECT));
    assertThat(stored.getCreatedAt(), is(notNullValue()));
    assertThat(stored.getUpdatedAt(), is(notNullValue()));
}
```

### Rule IT10: Test Multi-Step Business Workflows

**Requirement Level**: SHOULD

For features involving multiple API calls and state transitions:

```java
@Test
void test_contactPrimaryReassignment_workflow() throws Exception {
    String clientId = createClient("Workflow Corp");

    // Step 1: Create first contact as primary
    String body1 = createContact(clientId, "Alice", "Smith", "alice@test.com", true);
    String contactId1 = extractId(body1);

    // Step 2: Create second contact, not primary
    String body2 = createContact(clientId, "Bob", "Jones", "bob@test.com", false);
    String contactId2 = extractId(body2);

    // Step 3: Reassign primary to second contact
    mockMvc.perform(put("/api/v1/clients/" + clientId + "/contacts/" + contactId2 + "/primary"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.primary", is(true)));

    // Step 4: Verify first contact lost primary status
    mockMvc.perform(get("/api/v1/clients/" + clientId + "/contacts/" + contactId1))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.primary", is(false)));
}
```

---

## Helper Method Patterns

Every integration test class should define helper methods for creating prerequisite entities:

```java
// Helper: create client via API and return its ID
private String createClient(String companyName) throws Exception {
    String body = mockMvc.perform(post("/api/v1/clients")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content(clientJson(companyName, "Technology", 1000000)))
            .andExpect(status().isCreated())
            .andReturn().getResponse().getContentAsString();
    return objectMapper.readTree(body).get("id").asText();
}

// Helper: build client JSON (no boolean fields, ObjectMapper-safe but raw JSON preferred)
private String clientJson(String companyName, String industry, long revenue) {
    return """
            {
                "companyName": "%s",
                "industry": "%s",
                "annualRevenue": %d
            }
            """.formatted(companyName, industry, revenue);
}

// Helper: create contact via API and return the response body
private String createContact(String clientId, String firstName, String lastName,
                             String email, boolean isPrimary) throws Exception {
    return mockMvc.perform(post("/api/v1/clients/" + clientId + "/contacts")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content(contactJson(firstName, lastName, email, "TECHNICAL", isPrimary)))
            .andExpect(status().isCreated())
            .andReturn().getResponse().getContentAsString();
}

// Helper: extract ID from JSON response body
private String extractId(String responseBody) throws Exception {
    return objectMapper.readTree(responseBody).get("id").asText();
}
```

---

## Test Naming

### Rule IT11: Follow Integration Test Naming Convention

**Requirement Level**: MUST

Format: `test_{operation}_{scenario}_{expectedResult}`

| Pattern | Example |
|---------|---------|
| API success | `test_createClient_validRequest_returns201` |
| API validation | `test_createClient_missingName_returns400` |
| API not found | `test_getClient_nonExistentId_returns404` |
| Duplicate | `test_createClient_duplicateName_returns409` |
| Business rule | `test_deleteContact_primaryContact_returns422` |
| Workflow | `test_contactPrimaryReassignment_workflow` |
| Database | `test_createClient_persistsFieldsToMongoDB` |

---

## Build Configuration

Integration tests require a separate source set in `build.gradle`:

```groovy
// Integration test source set
sourceSets {
    integrationTest {
        java.srcDir 'src/integrationTest/java'
        resources.srcDir 'src/integrationTest/resources'
        compileClasspath += sourceSets.main.output + sourceSets.test.output
        runtimeClasspath += sourceSets.main.output + sourceSets.test.output
    }
}

configurations {
    integrationTestImplementation.extendsFrom testImplementation
    integrationTestRuntimeOnly.extendsFrom testRuntimeOnly
    integrationTestCompileOnly.extendsFrom testCompileOnly
    integrationTestAnnotationProcessor.extendsFrom testAnnotationProcessor
}

tasks.register('integrationTest', Test) {
    description = 'Runs integration tests.'
    group = 'verification'
    testClassesDirs = sourceSets.integrationTest.output.classesDirs
    classpath = sourceSets.integrationTest.runtimeClasspath
    useJUnitPlatform()
    shouldRunAfter tasks.named('test')
}
```

Key dependencies:
```groovy
testImplementation 'org.springframework.boot:spring-boot-starter-test'
testImplementation 'org.springframework.boot:spring-boot-starter-webmvc-test'  // Spring Boot 4.x name
testImplementation 'org.springframework.boot:spring-boot-testcontainers'
testImplementation 'org.testcontainers:mongodb'
testImplementation 'org.testcontainers:junit-jupiter'
```

**Spring Boot 4.x dependency names:** The web test starter is `spring-boot-starter-webmvc-test` (not `spring-boot-starter-test` alone). The web starter is `spring-boot-starter-webmvc` (not `spring-boot-starter-web`).

---

## Common Pitfalls and Fixes

### MongoTimeoutException when running multiple test classes

**Cause:** Using `@Container` annotation on the MongoDBContainer field. Each test class lifecycle-manages the container independently, but Spring caches the application context. When class B runs, Spring reuses the cached context that points to class A's container (which may have shut down).

**Fix:** Remove `@Container`. Start the container once in a `static {}` block:
```java
@ServiceConnection
static MongoDBContainer mongoDBContainer = new MongoDBContainer("mongo:7.0");
static { mongoDBContainer.start(); }
```

### HttpMessageNotReadableException: Cannot map null into type boolean

**Cause:** Using `ObjectMapper.writeValueAsString()` to serialize a DTO that has Lombok `boolean isPrimary`. Jackson 3.x sees the getter `isPrimary()` and serializes as `"primary": true`. On deserialization, the controller expects `"isPrimary"` to set the field, doesn't find it, and tries to map null to the primitive boolean.

**Fix:** Use raw JSON strings for request bodies:
```java
String json = """
        { "firstName": "John", "lastName": "Doe", "isPrimary": true }
        """;
```

### Tests pass individually but fail when run together

**Cause:** Database not cleaned between tests, or test order dependency.

**Fix:** Ensure `@BeforeEach cleanDatabase()` drops all collections. Never depend on data from another test.

### @AutoConfigureMockMvc import not found

**Cause:** In Spring Boot 4.x, the import changed from `org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc` to `org.springframework.boot.webmvc.test.autoconfigure.AutoConfigureMockMvc`.

**Fix:** Use the correct import:
```java
import org.springframework.boot.webmvc.test.autoconfigure.AutoConfigureMockMvc;
```

### ObjectMapper import (Jackson 3.x)

**Cause:** Spring Boot 4.x uses Jackson 3.x (`tools.jackson`), not Jackson 2.x (`com.fasterxml.jackson`).

**Fix:** Use the correct import:
```java
import tools.jackson.databind.ObjectMapper;
// NOT: import com.fasterxml.jackson.databind.ObjectMapper;
```

---

## Enforcement Checklist

Before submitting integration tests, verify:

- [ ] Test class extends `BaseIntegrationTest`
- [ ] `BaseIntegrationTest` uses static singleton container (`static {}` block, NOT `@Container`)
- [ ] `@ServiceConnection` on the container field (NOT `@DynamicPropertySource`)
- [ ] `@AutoConfigureMockMvc` (NOT `TestRestTemplate` with `RANDOM_PORT`)
- [ ] Database is cleaned between tests via `@BeforeEach`
- [ ] Request JSON is built manually (raw strings), NOT via `ObjectMapper.writeValueAsString()`
- [ ] Response boolean fields use Jackson-serialized names (`$.primary`, NOT `$.isPrimary`)
- [ ] Success responses verify status code AND body content via jsonPath
- [ ] Error responses verify status code (400, 404, 409, 422)
- [ ] CRUD lifecycle is tested end-to-end for new entities
- [ ] Multi-step workflows verify state at each step
- [ ] Test names follow `test_{operation}_{scenario}_{result}` convention
- [ ] All tests pass when run together (`./gradlew integrationTest`), not just individually
- [ ] Correct imports for Spring Boot 4.x (`tools.jackson`, `boot.webmvc.test.autoconfigure`)

## References

- For complete code examples, see [examples.md](examples.md)
- Spring Boot Testing: https://docs.spring.io/spring-boot/reference/testing/
- Testcontainers Java: https://java.testcontainers.org/
- Testcontainers MongoDB Module: https://java.testcontainers.org/modules/databases/mongodb/
