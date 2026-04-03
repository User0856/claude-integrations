# Code Quality Examples

Real examples demonstrating code quality checks and fixes.

---

## Compilation Fix Examples

Common compilation errors encountered in the CMS codebase and their fixes.

### Missing Import After Refactoring

**Error:**
```
src/main/java/com/enterprise/cms/service/InvoiceService.java:8: error: cannot find symbol
    import com.enterprise.cms.domain.model.ContractStatus;
                                                ^
  symbol:   class ContractStatus
  location: package com.enterprise.cms.domain.model
```

**Cause:** `ContractStatus` was moved from `domain.model` to `domain.model.contract` during a refactoring, but `InvoiceService` still uses the old import path.

**Before:**
```java
package com.enterprise.cms.service;

import com.enterprise.cms.domain.model.ContractStatus;
import com.enterprise.cms.domain.model.Invoice;
```

**After:**
```java
package com.enterprise.cms.service;

import com.enterprise.cms.domain.model.contract.ContractStatus;
import com.enterprise.cms.domain.model.Invoice;
```

### Type Mismatch in MapStruct Mapper

**Error:**
```
src/main/java/com/enterprise/cms/mapper/InvoiceMapperImpl.java:42: error: incompatible types:
  BigDecimal cannot be converted to Money
    invoice.setValue(request.getValue());
                           ^
```

**Cause:** The `CreateInvoiceRequest` has a `BigDecimal value` and `String currency` field, but `Invoice` has a `Money value` field. MapStruct cannot auto-map between them.

**Before (mapper missing custom mapping):**
```java
package com.enterprise.cms.mapper;

import com.enterprise.cms.domain.model.Invoice;
import com.enterprise.cms.dto.CreateInvoiceRequest;
import com.enterprise.cms.dto.InvoiceResponse;
import org.mapstruct.Mapper;
import org.mapstruct.Mapping;

@Mapper(componentModel = "spring")
public interface InvoiceMapper {

    @Mapping(target = "id", ignore = true)
    @Mapping(target = "status", ignore = true)
    @Mapping(target = "createdAt", ignore = true)
    @Mapping(target = "updatedAt", ignore = true)
    Invoice toEntity(CreateInvoiceRequest request);    // Fails: no mapping for Money value

    InvoiceResponse toResponse(Invoice invoice);
}
```

**After (with custom Money mapping):**
```java
package com.enterprise.cms.mapper;

import com.enterprise.cms.domain.model.Invoice;
import com.enterprise.cms.domain.model.Money;
import com.enterprise.cms.dto.CreateInvoiceRequest;
import com.enterprise.cms.dto.InvoiceResponse;
import org.mapstruct.Mapper;
import org.mapstruct.Mapping;
import org.mapstruct.Named;

@Mapper(componentModel = "spring")
public interface InvoiceMapper {

    @Mapping(target = "id", ignore = true)
    @Mapping(target = "status", ignore = true)
    @Mapping(target = "createdAt", ignore = true)
    @Mapping(target = "updatedAt", ignore = true)
    @Mapping(target = "value", source = ".", qualifiedByName = "toMoney")
    Invoice toEntity(CreateInvoiceRequest request);

    @Mapping(source = "value.amount", target = "value")
    @Mapping(source = "value.currency", target = "currency")
    InvoiceResponse toResponse(Invoice invoice);

    @Named("toMoney")
    default Money toMoney(CreateInvoiceRequest request) {
        return Money.builder()
                .amount(request.getValue())
                .currency(request.getCurrency())
                .build();
    }
}
```

### Missing Dependency in Build File

**Error:**
```
src/main/java/com/enterprise/cms/service/InvoiceService.java:5: error: package
  org.springframework.context does not exist
    import org.springframework.context.ApplicationEventPublisher;
                                       ^
```

**Cause:** The module only declared `spring-boot-starter-data-mongodb` but not `spring-boot-starter-web` (or a context-providing starter). `ApplicationEventPublisher` comes from `spring-context`.

**Fix in build.gradle:**
```java
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'         // Provides spring-context transitively
    implementation 'org.springframework.boot:spring-boot-starter-data-mongodb'
    // ...
}
```

---

## Checkstyle Violations and Fixes

### Unused Imports

**Violation:**
```
[WARN] InvoiceService.java:7: Unused import - java.util.stream.Collectors. [UnusedImports]
```

**Before:**
```java
package com.enterprise.cms.service;

import com.enterprise.cms.domain.model.Invoice;
import com.enterprise.cms.repository.InvoiceRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import java.util.stream.Collectors;                    // <-- Unused: switched to .toList()
import java.util.List;

@Service
@RequiredArgsConstructor
@Slf4j
public class InvoiceService {
    // ...
    public List<Invoice> getByContractId(String contractId) {
        return invoiceRepository.findByContractId(contractId)
                .stream()
                .toList();                             // Java 16+ .toList() replaces Collectors.toList()
    }
}
```

**After:**
```java
package com.enterprise.cms.service;

import com.enterprise.cms.domain.model.Invoice;
import com.enterprise.cms.repository.InvoiceRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import java.util.List;

@Service
@RequiredArgsConstructor
@Slf4j
public class InvoiceService {
    // ...
    public List<Invoice> getByContractId(String contractId) {
        return invoiceRepository.findByContractId(contractId)
                .stream()
                .toList();
    }
}
```

### Line Length Exceeds 120 Characters

**Violation:**
```
[WARN] ContractController.java:34: Line is longer than 120 characters (finds: 147). [LineLength]
```

**Before:**
```java
@PostMapping("/{id}/transition")
public ResponseEntity<ContractResponse> transition(@PathVariable String id, @Valid @RequestBody ContractTransitionRequest request) {
    var contract = contractService.transitionStatus(id, request.getNewStatus());
    return ResponseEntity.ok(contractMapper.toResponse(contract));
}
```

**After:**
```java
@PostMapping("/{id}/transition")
public ResponseEntity<ContractResponse> transition(
        @PathVariable String id,
        @Valid @RequestBody ContractTransitionRequest request) {
    var contract = contractService.transitionStatus(id, request.getNewStatus());
    return ResponseEntity.ok(contractMapper.toResponse(contract));
}
```

### Missing Braces on Control Statements

**Violation:**
```
[WARN] ContractService.java:28: 'if' construct must use '{}'s. [NeedBraces]
```

**Before:**
```java
public Contract transitionStatus(String contractId, ContractStatus newStatus) {
    Contract contract = getById(contractId);
    ContractStatus oldStatus = contract.getStatus();

    if (!oldStatus.canTransitionTo(newStatus))
        throw new InvalidContractTransitionException(oldStatus, newStatus);

    contract.setStatus(newStatus);
    return contractRepository.save(contract);
}
```

**After:**
```java
public Contract transitionStatus(String contractId, ContractStatus newStatus) {
    Contract contract = getById(contractId);
    ContractStatus oldStatus = contract.getStatus();

    if (!oldStatus.canTransitionTo(newStatus)) {
        throw new InvalidContractTransitionException(oldStatus, newStatus);
    }

    contract.setStatus(newStatus);
    return contractRepository.save(contract);
}
```

### Constant Naming Convention

**Violation:**
```
[WARN] InvoiceService.java:15: Name 'defaultDueDays' must match pattern '^[A-Z][A-Z0-9]*(_[A-Z0-9]+)*$'. [ConstantName]
```

**Before:**
```java
public class InvoiceService {

    private static final int defaultDueDays = 30;
    private static final String invoicePrefix = "INV";
```

**After:**
```java
public class InvoiceService {

    private static final int DEFAULT_DUE_DAYS = 30;
    private static final String INVOICE_PREFIX = "INV";
```

---

## SpotBugs Findings and Fixes

### NP_NULL_ON_SOME_PATH — Possible Null Pointer Dereference

**Finding:**
```
M B NP: Possible null pointer dereference of contract.getValue() in InvoiceService.generateFromContract(String)
  At InvoiceService.java:[line 45]
```

**Before:**
```java
public Invoice generateFromContract(String contractId) {
    Contract contract = contractService.getById(contractId);

    return Invoice.builder()
            .contractId(contractId)
            .clientId(contract.getClientId())
            .invoiceNumber(generateInvoiceNumber())
            .value(Money.builder()
                    .amount(contract.getValue().getAmount())     // SpotBugs: getValue() may return null
                    .currency(contract.getValue().getCurrency()) // SpotBugs: getValue() may return null
                    .build())
            .issueDate(LocalDate.now())
            .dueDate(LocalDate.now().plusDays(DEFAULT_DUE_DAYS))
            .status(InvoiceStatus.DRAFT)
            .build();
}
```

**After:**
```java
public Invoice generateFromContract(String contractId) {
    Contract contract = contractService.getById(contractId);
    Money contractValue = contract.getValue();

    if (contractValue == null) {
        throw new BusinessRuleViolationException(
                "Cannot generate invoice: contract " + contractId + " has no value set");
    }

    return Invoice.builder()
            .contractId(contractId)
            .clientId(contract.getClientId())
            .invoiceNumber(generateInvoiceNumber())
            .value(Money.builder()
                    .amount(contractValue.getAmount())
                    .currency(contractValue.getCurrency())
                    .build())
            .issueDate(LocalDate.now())
            .dueDate(LocalDate.now().plusDays(DEFAULT_DUE_DAYS))
            .status(InvoiceStatus.DRAFT)
            .build();
}
```

### EI_EXPOSE_REP — Mutable Field Returned Directly

**Finding:**
```
M B EI2: ClientController.create() may expose internal representation by storing an externally mutable object
  into ContractStatusChangedEvent.contract
  At ContractStatusChangedEvent.java:[line 15]

M B EI: Contract.getLineItems() may expose internal representation by returning Contract.lineItems
  At Contract.java:[line 44]
```

**Before:**
```java
@Document(collection = "contracts")
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class Contract {

    @Id
    private String id;

    @Builder.Default
    private List<LineItem> lineItems = new ArrayList<>();

    @Builder.Default
    private List<StatusChange> statusHistory = new ArrayList<>();

    // Lombok @Data generates getLineItems() that returns the list directly — mutable!
}
```

**After (Option A — Defensive copies in domain event):**
```java
@Getter
public class ContractStatusChangedEvent extends ApplicationEvent {

    private final Contract contract;
    private final ContractStatus previousStatus;

    public ContractStatusChangedEvent(Object source, Contract contract,
                                      ContractStatus previousStatus) {
        super(source);
        this.contract = Contract.builder()
                .id(contract.getId())
                .clientId(contract.getClientId())
                .contractNumber(contract.getContractNumber())
                .status(contract.getStatus())
                .value(contract.getValue())
                .lineItems(new ArrayList<>(contract.getLineItems()))
                .statusHistory(new ArrayList<>(contract.getStatusHistory()))
                .startDate(contract.getStartDate())
                .endDate(contract.getEndDate())
                .createdAt(contract.getCreatedAt())
                .updatedAt(contract.getUpdatedAt())
                .build();
        this.previousStatus = previousStatus;
    }
}
```

**After (Option B — Unmodifiable getters via Lombok config):**
```java
@Document(collection = "contracts")
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class Contract {

    @Id
    private String id;

    @Builder.Default
    private List<LineItem> lineItems = new ArrayList<>();

    @Builder.Default
    private List<StatusChange> statusHistory = new ArrayList<>();

    public List<LineItem> getLineItems() {
        return Collections.unmodifiableList(lineItems);
    }

    public List<StatusChange> getStatusHistory() {
        return Collections.unmodifiableList(statusHistory);
    }

    public void addLineItem(LineItem item) {
        lineItems.add(item);
    }

    public void addStatusChange(StatusChange change) {
        statusHistory.add(change);
    }
}
```

### RCN_REDUNDANT_NULLCHECK — Redundant Null Check

**Finding:**
```
M B RCN: Redundant nullcheck of value which is known to be non-null
  At ClientService.java:[line 22]
```

**Before:**
```java
public Client getById(String id) {
    Client client = clientRepository.findById(id)
            .orElseThrow(() -> new EntityNotFoundException("Client", id));

    if (client != null) {                              // Redundant: orElseThrow guarantees non-null
        log.debug("Found client: {}", client.getId());
    }
    return client;
}
```

**After:**
```java
public Client getById(String id) {
    Client client = clientRepository.findById(id)
            .orElseThrow(() -> new EntityNotFoundException("Client", id));

    log.debug("Found client: {}", client.getId());     // No null check needed — orElseThrow guarantees non-null
    return client;
}
```

---

## Pre-Commit Checklist Walkthrough

A complete example of running the pre-commit checklist after implementing a new `InvoiceService.generateFromContract()` method.

### Step 1: Compile All Sources

```bash
$ ./gradlew compileJava compileTestJava

> Task :compileJava
> Task :compileTestJava

BUILD SUCCESSFUL in 4s
2 actionable tasks: 2 executed
```

Result: PASS. No compilation errors.

### Step 2: Run All Tests

```bash
$ ./gradlew test

> Task :test

InvoiceServiceTest > create_shouldSaveAndReturnInvoice() PASSED
InvoiceServiceTest > create_shouldThrowWhenContractNotActive() PASSED
InvoiceServiceTest > generateFromContract_shouldCreateDraftInvoice() PASSED
InvoiceServiceTest > generateFromContract_shouldThrowWhenContractValueIsNull() PASSED
ClientServiceTest > create_shouldSaveAndReturnClient() PASSED
ClientServiceTest > create_shouldThrowWhenEmailAlreadyExists() PASSED
ClientServiceTest > getById_shouldReturnClientWhenExists() PASSED
ClientServiceTest > getById_shouldThrowWhenNotFound() PASSED
ContractServiceTest > transitionStatus_shouldTransitionAndPublishEvent() PASSED
ContractServiceTest > transitionStatus_shouldThrowWhenInvalidTransition() PASSED

BUILD SUCCESSFUL in 8s
3 actionable tasks: 1 executed, 2 up-to-date
```

Result: PASS. All 10 tests passed, including the 2 new tests.

```bash
$ ./gradlew integrationTest

> Task :integrationTest

ClientControllerIntegrationTest > shouldCreateAndRetrieveClient() PASSED
ContractControllerIntegrationTest > shouldCreateContractAndTransitionStatus() PASSED
InvoiceControllerIntegrationTest > shouldGenerateInvoiceFromContract() PASSED

BUILD SUCCESSFUL in 22s
4 actionable tasks: 1 executed, 3 up-to-date
```

Result: PASS. All 3 integration tests passed.

### Step 3: Static Analysis

```bash
$ ./gradlew checkstyleMain checkstyleTest

> Task :checkstyleMain
> Task :checkstyleTest

BUILD SUCCESSFUL in 3s
2 actionable tasks: 2 executed
```

Result: PASS. No Checkstyle violations.

```bash
$ ./gradlew spotbugsMain

> Task :spotbugsMain

BUILD SUCCESSFUL in 5s
1 actionable task: 1 executed
```

Result: PASS. No SpotBugs findings.

### Step 4: Review the Diff

```bash
$ git diff --stat

 src/main/java/com/enterprise/cms/service/InvoiceService.java   | 28 +++++++++++++
 src/test/java/com/enterprise/cms/service/InvoiceServiceTest.java | 34 ++++++++++++++++
 2 files changed, 62 insertions(+)
```

```bash
$ git diff

diff --git a/src/main/java/com/enterprise/cms/service/InvoiceService.java b/...
@@ -4,6 +4,7 @@
+import com.enterprise.cms.domain.exception.BusinessRuleViolationException;
 import com.enterprise.cms.domain.model.*;
 import com.enterprise.cms.repository.InvoiceRepository;
@@ -28,4 +29,31 @@
+    public Invoice generateFromContract(String contractId) {
+        Contract contract = contractService.getById(contractId);
+        Money contractValue = contract.getValue();
+
+        if (contractValue == null) {
+            throw new BusinessRuleViolationException(
+                    "Cannot generate invoice: contract " + contractId + " has no value set");
+        }
+
+        return Invoice.builder()
+                .contractId(contractId)
+                .clientId(contract.getClientId())
+                .invoiceNumber(generateInvoiceNumber())
+                .value(Money.builder()
+                        .amount(contractValue.getAmount())
+                        .currency(contractValue.getCurrency())
+                        .build())
+                .issueDate(LocalDate.now())
+                .dueDate(LocalDate.now().plusDays(DEFAULT_DUE_DAYS))
+                .status(InvoiceStatus.DRAFT)
+                .build();
+    }
```

**Verify before committing:**
- [x] Only expected files are modified (2 files: service + test)
- [x] No debugging code (`System.out.println`, hardcoded URLs)
- [x] No commented-out code blocks
- [x] No `TODO` or `FIXME` comments
- [x] No new dependencies added
- [x] Import added for `BusinessRuleViolationException` is correct

**All checks passed. Safe to commit.**
