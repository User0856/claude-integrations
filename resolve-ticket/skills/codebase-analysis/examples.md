# Codebase Analysis Examples

Real examples demonstrating systematic codebase analysis patterns.

---

## Build System Analysis

When starting codebase analysis, the build file is the single most informative file. Here is an annotated example showing what to extract from a typical Gradle build.

```java
// build.gradle

plugins {
    id 'java'
    id 'org.springframework.boot' version '3.2.4'       // <-- Spring Boot version (determines compatible dependency versions)
    id 'io.spring.dependency-management' version '1.1.4' // <-- BOM management (centralizes versions)
    id 'checkstyle'                                       // <-- Static analysis: Checkstyle is active
    id 'com.github.spotbugs' version '6.0.8'             // <-- Static analysis: SpotBugs is active
    id 'jacoco'                                           // <-- Code coverage: JaCoCo is configured
}

group = 'com.enterprise'
version = '1.0.0-SNAPSHOT'

java {
    sourceCompatibility = JavaVersion.VERSION_17          // <-- Java 17 (use modern APIs: records, sealed, text blocks)
}

configurations {
    compileOnly {
        extendsFrom annotationProcessor
    }
}

repositories {
    mavenCentral()
}

dependencies {
    // --- Runtime dependencies ---
    implementation 'org.springframework.boot:spring-boot-starter-web'        // <-- REST API project
    implementation 'org.springframework.boot:spring-boot-starter-data-mongodb' // <-- MongoDB (not JPA) — use @Document, not @Entity
    implementation 'org.springframework.boot:spring-boot-starter-validation' // <-- Bean validation is available (jakarta.validation)
    implementation 'org.mapstruct:mapstruct:1.5.5.Final'                    // <-- MapStruct for DTO mapping

    // --- Compile-time only ---
    compileOnly 'org.projectlombok:lombok'                                  // <-- Lombok is active (@Data, @Builder, etc.)
    annotationProcessor 'org.projectlombok:lombok'
    annotationProcessor 'org.mapstruct:mapstruct-processor:1.5.5.Final'
    annotationProcessor 'org.projectlombok:lombok-mapstruct-binding:0.2.0' // <-- Lombok + MapStruct interop

    // --- Test dependencies ---
    testImplementation 'org.springframework.boot:spring-boot-starter-test'  // <-- JUnit 5 + Mockito + AssertJ
    testImplementation 'org.testcontainers:mongodb:1.19.7'                 // <-- Testcontainers for integration tests
    testImplementation 'org.testcontainers:junit-jupiter:1.19.7'
}

// --- Quality gate configuration ---
checkstyle {
    toolVersion = '10.14.0'
    configFile = file("config/checkstyle/checkstyle.xml")  // <-- Read this file to understand coding rules
    maxWarnings = 0                                         // <-- Zero tolerance: any warning breaks the build
}

spotbugs {
    effort = 'max'
    reportLevel = 'medium'
    excludeFilter = file("config/spotbugs/spotbugs-exclude.xml") // <-- Read this for suppressed patterns
}

jacocoTestReport {
    reports {
        xml.required = true
        html.required = true
    }
}

jacocoTestCoverageVerification {
    violationRules {
        rule {
            limit {
                minimum = 0.80                              // <-- 80% line coverage required
            }
        }
    }
}

test {
    useJUnitPlatform()                                      // <-- JUnit 5 platform
    finalizedBy jacocoTestReport
}

tasks.register('integrationTest', Test) {
    useJUnitPlatform {
        includeTags 'integration'                           // <-- Integration tests use @Tag("integration")
    }
    shouldRunAfter test
}

tasks.named('check') {
    dependsOn 'integrationTest'                             // <-- Full build runs integration tests too
}
```

**Key takeaways from this build file:**
- Spring Boot 3.2.4 with MongoDB (not JPA)
- Java 17 — can use records, sealed classes, text blocks
- Lombok + MapStruct both active (check annotation processor order)
- Checkstyle + SpotBugs + JaCoCo all enforced
- Integration tests use JUnit 5 `@Tag("integration")` and Testcontainers
- 80% minimum coverage required

---

## Package Structure Mapping

After reading the build file, map the full package structure. Use the directory tree to understand the architectural layers.

```
src/main/java/com/enterprise/cms/
├── CmsApplication.java                          # @SpringBootApplication entry point
│
├── config/                                       # LAYER: Configuration
│   ├── MongoConfig.java                          #   MongoDB auditing, converters
│   └── WebConfig.java                            #   CORS, interceptors
│
├── domain/                                       # LAYER: Domain model (no framework dependencies)
│   ├── model/                                    #   Entities and value objects
│   │   ├── Client.java                           #     @Document — root aggregate
│   │   ├── Contract.java                         #     @Document — root aggregate
│   │   ├── Invoice.java                          #     @Document — root aggregate
│   │   ├── ContractStatus.java                   #     Enum with transition validation
│   │   ├── InvoiceStatus.java                    #     Enum with transition validation
│   │   ├── Money.java                            #     Embedded value object (no @Id)
│   │   ├── Address.java                          #     Embedded value object (no @Id)
│   │   ├── LineItem.java                         #     Embedded value object (no @Id)
│   │   └── StatusChange.java                     #     Embedded audit trail object
│   ├── event/                                    #   Spring application events
│   │   ├── ContractStatusChangedEvent.java       #     Published on status transitions
│   │   └── InvoiceCreatedEvent.java              #     Published on invoice creation
│   └── exception/                                #   Custom exception hierarchy
│       ├── EntityNotFoundException.java          #     → 404 NOT_FOUND
│       ├── DuplicateEntityException.java         #     → 409 CONFLICT
│       ├── InvalidContractTransitionException.java #   → 409 CONFLICT
│       └── BusinessRuleViolationException.java   #     → 422 UNPROCESSABLE_ENTITY
│
├── repository/                                   # LAYER: Data access
│   ├── ClientRepository.java                     #   MongoRepository<Client, String>
│   ├── ContractRepository.java                   #   MongoRepository<Contract, String>
│   └── InvoiceRepository.java                    #   MongoRepository<Invoice, String>
│
├── service/                                      # LAYER: Business logic
│   ├── ClientService.java                        #   CRUD + client-specific operations
│   ├── ContractService.java                      #   CRUD + status transitions + events
│   └── InvoiceService.java                       #   CRUD + generation from contracts
│
├── dto/                                          # LAYER: API contracts
│   ├── CreateClientRequest.java                  #   @Valid input DTO
│   ├── UpdateClientRequest.java                  #   Partial update DTO
│   ├── ClientResponse.java                       #   Output DTO
│   ├── CreateContractRequest.java                #   @Valid input DTO
│   ├── ContractResponse.java                     #   Output DTO
│   ├── ContractTransitionRequest.java            #   Status change DTO
│   ├── CreateInvoiceRequest.java                 #   @Valid input DTO
│   ├── InvoiceResponse.java                      #   Output DTO
│   └── ErrorResponse.java                        #   Standard error envelope
│
├── mapper/                                       # LAYER: DTO ↔ Entity mapping
│   ├── ClientMapper.java                         #   @Mapper(componentModel = "spring")
│   ├── ContractMapper.java                       #   @Mapper with custom Money mapping
│   └── InvoiceMapper.java                        #   @Mapper with line item mapping
│
└── controller/                                   # LAYER: REST endpoints
    ├── ClientController.java                     #   /api/v1/clients
    ├── ContractController.java                   #   /api/v1/contracts
    ├── InvoiceController.java                    #   /api/v1/invoices
    └── GlobalExceptionHandler.java               #   @RestControllerAdvice — maps exceptions to HTTP status
```

**Observations:**
- Layer-based packaging (not feature-based) — all services in one package, all controllers in another
- Three root aggregates: Client, Contract, Invoice
- Consistent naming: `{Entity}Controller`, `{Entity}Service`, `{Entity}Repository`, `{Entity}Mapper`
- DTOs split into `Create{Entity}Request`, `Update{Entity}Request`, `{Entity}Response`
- Value objects (Money, Address, LineItem) are embedded, not separate documents

---

## Pattern Recognition

Compare two existing services to identify conventions. Read them side by side and document every pattern that repeats.

### Service A: ClientService

```java
package com.enterprise.cms.service;

import com.enterprise.cms.domain.exception.DuplicateEntityException;
import com.enterprise.cms.domain.exception.EntityNotFoundException;
import com.enterprise.cms.domain.model.Client;
import com.enterprise.cms.repository.ClientRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

import java.util.List;

@Service                                              // Pattern: always @Service
@RequiredArgsConstructor                              // Pattern: constructor injection via Lombok
@Slf4j                                                // Pattern: SLF4J logging on every service
public class ClientService {

    private final ClientRepository clientRepository;  // Pattern: single repository dependency

    public Client create(Client client) {             // Pattern: create() takes entity, returns saved entity
        if (clientRepository.existsByEmail(client.getEmail())) {
            throw new DuplicateEntityException("Client", "email", client.getEmail());
        }                                             // Pattern: uniqueness check before save
        Client saved = clientRepository.save(client);
        log.info("Created client {} with email {}", saved.getId(), saved.getEmail());
        return saved;                                 // Pattern: log after successful operation
    }

    public Client getById(String id) {                // Pattern: getById() throws EntityNotFoundException
        return clientRepository.findById(id)
                .orElseThrow(() -> new EntityNotFoundException("Client", id));
    }

    public List<Client> getAll() {                    // Pattern: getAll() returns List, no pagination
        return clientRepository.findAll();
    }

    public Client update(String id, Client updates) { // Pattern: update() fetches then patches
        Client existing = getById(id);
        existing.setName(updates.getName());
        existing.setEmail(updates.getEmail());
        existing.setPhone(updates.getPhone());
        existing.setAddress(updates.getAddress());
        return clientRepository.save(existing);
    }

    public void delete(String id) {                   // Pattern: delete() verifies existence first
        Client client = getById(id);
        clientRepository.delete(client);
        log.info("Deleted client {}", id);
    }
}
```

### Service B: InvoiceService

```java
package com.enterprise.cms.service;

import com.enterprise.cms.domain.event.InvoiceCreatedEvent;
import com.enterprise.cms.domain.exception.BusinessRuleViolationException;
import com.enterprise.cms.domain.exception.EntityNotFoundException;
import com.enterprise.cms.domain.model.*;
import com.enterprise.cms.repository.InvoiceRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.context.ApplicationEventPublisher;
import org.springframework.stereotype.Service;

import java.math.BigDecimal;
import java.time.LocalDate;
import java.util.List;

@Service                                              // Pattern: always @Service             ✔ matches
@RequiredArgsConstructor                              // Pattern: constructor injection        ✔ matches
@Slf4j                                                // Pattern: SLF4J logging               ✔ matches
public class InvoiceService {

    private final InvoiceRepository invoiceRepository; // Pattern: single repository           ✔ matches
    private final ContractService contractService;     // NEW: cross-service dependency
    private final ApplicationEventPublisher eventPublisher; // NEW: event publishing

    public Invoice create(Invoice invoice) {           // Pattern: create() takes entity       ✔ matches
        Contract contract = contractService.getById(invoice.getContractId());
        if (contract.getStatus() != ContractStatus.ACTIVE) {
            throw new BusinessRuleViolationException(
                    "Cannot create invoice for non-active contract");
        }                                              // Pattern: business rule validation    ✔ matches
        invoice.setStatus(InvoiceStatus.DRAFT);
        Invoice saved = invoiceRepository.save(invoice);
        eventPublisher.publishEvent(new InvoiceCreatedEvent(this, saved));
        log.info("Created invoice {} for contract {}", saved.getId(), saved.getContractId());
        return saved;                                  // Pattern: log after success           ✔ matches
    }

    public Invoice getById(String id) {                // Pattern: getById() + exception       ✔ matches
        return invoiceRepository.findById(id)
                .orElseThrow(() -> new EntityNotFoundException("Invoice", id));
    }

    public List<Invoice> getByContractId(String contractId) {
        return invoiceRepository.findByContractId(contractId);
    }

    public BigDecimal calculateTotal(String invoiceId) {
        Invoice invoice = getById(invoiceId);
        return invoice.getLineItems().stream()
                .map(li -> li.getUnitPrice().getAmount()
                        .multiply(BigDecimal.valueOf(li.getQuantity())))
                .reduce(BigDecimal.ZERO, BigDecimal::add);
    }
}
```

### Extracted Conventions

| Convention | Pattern | Confidence |
|------------|---------|------------|
| Class annotations | `@Service` + `@RequiredArgsConstructor` + `@Slf4j` | Definite (both match) |
| Injection style | `private final` fields, no `@Autowired` | Definite |
| Create method | Takes entity, returns saved entity | Definite |
| Uniqueness check | Check before save, throw `DuplicateEntityException` | Definite |
| Not-found handling | `findById().orElseThrow(() -> new EntityNotFoundException(...))` | Definite |
| Business rules | Throw `BusinessRuleViolationException` with descriptive message | Definite |
| Logging | `log.info()` after successful mutating operations, include IDs | Definite |
| Events | Use `ApplicationEventPublisher` for cross-cutting concerns | Likely (1 of 2 services) |
| Return type | Methods return entity or List, never Optional | Definite |

**Rule: Any new service must follow all "Definite" conventions exactly.**

---

## Test Infrastructure Discovery

Before writing tests, explore the test source tree to understand setup, base classes, data builders, and naming conventions.

### Step 1: Find the Base Integration Test

```java
package com.enterprise.cms;

import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.containers.MongoDBContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
public abstract class BaseIntegrationTest {

    @Container
    static MongoDBContainer mongoDBContainer = new MongoDBContainer("mongo:7.0");

    @DynamicPropertySource
    static void setProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.data.mongodb.uri", mongoDBContainer::getReplicaSetUrl);
    }
}
```

**What this tells you:**
- All integration tests extend `BaseIntegrationTest` — do not create your own container
- Testcontainers manages MongoDB lifecycle — no external database needed
- `RANDOM_PORT` means use `@LocalServerPort` or `TestRestTemplate` for HTTP tests
- The container is `static` — shared across all tests in the class (faster, one startup)

### Step 2: Find Test Data Builders

```java
package com.enterprise.cms.testutil;

import com.enterprise.cms.domain.model.*;

import java.math.BigDecimal;
import java.time.LocalDate;
import java.util.ArrayList;

public class TestDataBuilder {

    public static Client.ClientBuilder aClient() {
        return Client.builder()
                .name("Acme Corporation")
                .email("contact@acme.com")
                .phone("+1-555-0100")
                .address(Address.builder()
                        .street("123 Main St")
                        .city("Springfield")
                        .state("IL")
                        .zipCode("62704")
                        .country("US")
                        .build());
    }

    public static Contract.ContractBuilder aContract() {
        return Contract.builder()
                .contractNumber("CTR-2024-001")
                .status(ContractStatus.DRAFT)
                .value(Money.builder()
                        .amount(new BigDecimal("50000.00"))
                        .currency("USD")
                        .build())
                .lineItems(new ArrayList<>())
                .statusHistory(new ArrayList<>())
                .startDate(LocalDate.now())
                .endDate(LocalDate.now().plusYears(1));
    }

    public static Invoice.InvoiceBuilder anInvoice() {
        return Invoice.builder()
                .invoiceNumber("INV-2024-001")
                .status(InvoiceStatus.DRAFT)
                .issueDate(LocalDate.now())
                .dueDate(LocalDate.now().plusDays(30))
                .lineItems(new ArrayList<>());
    }

    public static LineItem.LineItemBuilder aLineItem() {
        return LineItem.builder()
                .description("Consulting Services")
                .quantity(10)
                .unitPrice(Money.builder()
                        .amount(new BigDecimal("150.00"))
                        .currency("USD")
                        .build());
    }
}
```

**What this tells you:**
- Builders return `{Entity}.{Entity}Builder` (not the entity itself) — so tests can override fields
- Method naming: `a{Entity}()` or `an{Entity}()` — follow this pattern for new entities
- Default values are realistic but deterministic (no random data)
- Always use `new ArrayList<>()` for list fields (not `List.of()`) — entities are mutable

### Step 3: Study Test Naming and Structure

```java
package com.enterprise.cms.service;

import com.enterprise.cms.domain.exception.DuplicateEntityException;
import com.enterprise.cms.domain.exception.EntityNotFoundException;
import com.enterprise.cms.domain.model.Client;
import com.enterprise.cms.repository.ClientRepository;
import com.enterprise.cms.testutil.TestDataBuilder;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.util.Optional;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)                    // Convention: Mockito extension, not SpringBootTest
class ClientServiceTest {

    @Mock
    private ClientRepository clientRepository;          // Convention: @Mock for dependencies

    @InjectMocks
    private ClientService clientService;                // Convention: @InjectMocks for the class under test

    @Test
    void create_shouldSaveAndReturnClient() {           // Naming: method_shouldExpected
        // given
        Client client = TestDataBuilder.aClient().build();
        when(clientRepository.existsByEmail(client.getEmail())).thenReturn(false);
        when(clientRepository.save(any(Client.class))).thenReturn(client);

        // when
        Client result = clientService.create(client);

        // then
        assertThat(result).isNotNull();
        assertThat(result.getEmail()).isEqualTo("contact@acme.com");
        verify(clientRepository).save(client);
    }

    @Test
    void create_shouldThrowWhenEmailAlreadyExists() {   // Naming: method_shouldThrowWhenCondition
        // given
        Client client = TestDataBuilder.aClient().build();
        when(clientRepository.existsByEmail(client.getEmail())).thenReturn(true);

        // when / then
        assertThatThrownBy(() -> clientService.create(client))
                .isInstanceOf(DuplicateEntityException.class)
                .hasMessageContaining("email");

        verify(clientRepository, never()).save(any());
    }

    @Test
    void getById_shouldReturnClientWhenExists() {       // Naming: method_shouldExpectedWhenCondition
        // given
        Client client = TestDataBuilder.aClient().build();
        when(clientRepository.findById("client-1")).thenReturn(Optional.of(client));

        // when
        Client result = clientService.getById("client-1");

        // then
        assertThat(result).isEqualTo(client);
    }

    @Test
    void getById_shouldThrowWhenNotFound() {
        // given
        when(clientRepository.findById("missing")).thenReturn(Optional.empty());

        // when / then
        assertThatThrownBy(() -> clientService.getById("missing"))
                .isInstanceOf(EntityNotFoundException.class)
                .hasMessageContaining("missing");
    }
}
```

### Extracted Test Conventions

| Convention | Pattern |
|------------|---------|
| Unit test runner | `@ExtendWith(MockitoExtension.class)` |
| Integration test base | Extend `BaseIntegrationTest` |
| Test naming | `methodName_shouldExpectedBehavior` or `methodName_shouldExpectedWhenCondition` |
| Test structure | `// given` / `// when` / `// then` comments |
| Assertion library | AssertJ (`assertThat`, `assertThatThrownBy`) — not JUnit assertions |
| Mocking | Mockito (`when`, `verify`, `never()`) |
| Test data | `TestDataBuilder.aClient().build()` — override only what matters for the test |
| Exception testing | `assertThatThrownBy` with `isInstanceOf` and `hasMessageContaining` |
| Verification | `verify(repo, never()).save(any())` to assert something did NOT happen |
