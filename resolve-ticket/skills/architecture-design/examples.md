# Architecture Design Examples

This file contains complete code examples demonstrating the layered architecture patterns defined in the Architecture and Design Standards skill.

---

## Complete Entity with Value Objects

```java
package com.enterprise.cms.domain.model;

import lombok.*;
import org.springframework.data.annotation.*;
import org.springframework.data.mongodb.core.index.Indexed;
import org.springframework.data.mongodb.core.mapping.Document;

import java.math.BigDecimal;
import java.time.Instant;
import java.time.LocalDate;
import java.util.ArrayList;
import java.util.List;

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

    @Builder.Default
    private List<LineItem> lineItems = new ArrayList<>();

    @Builder.Default
    private List<StatusChange> statusHistory = new ArrayList<>();

    private LocalDate startDate;
    private LocalDate endDate;

    @CreatedDate
    private Instant createdAt;

    @LastModifiedDate
    private Instant updatedAt;
}

// Embedded value object — no @Document, no @Id
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class Money {
    private BigDecimal amount;
    private String currency;
}

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class LineItem {
    private String description;
    private int quantity;
    private Money unitPrice;
}

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class StatusChange {
    private ContractStatus fromStatus;
    private ContractStatus toStatus;
    private Instant changedAt;
    private String changedBy;
    private String reason;
}
```

---

## Enum with Transition Validation

```java
package com.enterprise.cms.domain.model;

import java.util.Map;
import java.util.Set;

public enum ContractStatus {
    DRAFT,
    PENDING_APPROVAL,
    ACTIVE,
    RENEWAL,
    EXPIRED,
    TERMINATED;

    private static final Map<ContractStatus, Set<ContractStatus>> VALID_TRANSITIONS = Map.of(
            DRAFT, Set.of(PENDING_APPROVAL),
            PENDING_APPROVAL, Set.of(ACTIVE, DRAFT),
            ACTIVE, Set.of(RENEWAL, TERMINATED, EXPIRED),
            RENEWAL, Set.of(ACTIVE, EXPIRED, TERMINATED),
            EXPIRED, Set.of(),
            TERMINATED, Set.of()
    );

    public boolean canTransitionTo(ContractStatus target) {
        return VALID_TRANSITIONS.getOrDefault(this, Set.of()).contains(target);
    }
}
```

---

## Repository Interface

```java
package com.enterprise.cms.repository;

import com.enterprise.cms.domain.model.Contract;
import com.enterprise.cms.domain.model.ContractStatus;
import org.springframework.data.mongodb.repository.MongoRepository;
import org.springframework.data.mongodb.repository.Query;

import java.time.LocalDate;
import java.util.List;
import java.util.Optional;

public interface ContractRepository extends MongoRepository<Contract, String> {

    List<Contract> findByClientId(String clientId);

    List<Contract> findByStatus(ContractStatus status);

    List<Contract> findByClientIdAndStatus(String clientId, ContractStatus status);

    Optional<Contract> findByContractNumber(String contractNumber);

    @Query("{ 'endDate': { $lte: ?0 }, 'status': 'ACTIVE' }")
    List<Contract> findRenewalsDueBefore(LocalDate date);

    long countByClientIdAndStatus(String clientId, ContractStatus status);

    boolean existsByContractNumber(String contractNumber);
}
```

---

## Service with Business Logic

```java
package com.enterprise.cms.service;

import com.enterprise.cms.domain.event.ContractStatusChangedEvent;
import com.enterprise.cms.domain.exception.EntityNotFoundException;
import com.enterprise.cms.domain.exception.InvalidContractTransitionException;
import com.enterprise.cms.domain.exception.DuplicateEntityException;
import com.enterprise.cms.domain.model.*;
import com.enterprise.cms.repository.ContractRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.context.ApplicationEventPublisher;
import org.springframework.stereotype.Service;

import java.time.Instant;
import java.time.LocalDate;
import java.util.List;

@Service
@RequiredArgsConstructor
@Slf4j
public class ContractService {

    private final ContractRepository contractRepository;
    private final ApplicationEventPublisher eventPublisher;

    public Contract create(Contract contract) {
        if (contractRepository.existsByContractNumber(contract.getContractNumber())) {
            throw new DuplicateEntityException("Contract",
                    "contractNumber", contract.getContractNumber());
        }
        contract.setStatus(ContractStatus.DRAFT);
        Contract saved = contractRepository.save(contract);
        log.info("Created contract {} for client {}", saved.getId(), saved.getClientId());
        return saved;
    }

    public Contract getById(String id) {
        return contractRepository.findById(id)
                .orElseThrow(() -> new EntityNotFoundException("Contract", id));
    }

    public List<Contract> getByClientId(String clientId) {
        return contractRepository.findByClientId(clientId);
    }

    public Contract transitionStatus(String contractId, ContractStatus newStatus) {
        Contract contract = getById(contractId);
        ContractStatus oldStatus = contract.getStatus();

        if (!oldStatus.canTransitionTo(newStatus)) {
            throw new InvalidContractTransitionException(oldStatus, newStatus);
        }

        contract.setStatus(newStatus);
        contract.getStatusHistory().add(StatusChange.builder()
                .fromStatus(oldStatus)
                .toStatus(newStatus)
                .changedAt(Instant.now())
                .build());

        Contract saved = contractRepository.save(contract);
        eventPublisher.publishEvent(
                new ContractStatusChangedEvent(this, saved, oldStatus));

        log.info("Contract {} transitioned {} -> {}", contractId, oldStatus, newStatus);
        return saved;
    }

    public List<Contract> findRenewalsDue(int daysAhead) {
        LocalDate cutoff = LocalDate.now().plusDays(daysAhead);
        return contractRepository.findRenewalsDueBefore(cutoff);
    }
}
```

---

## Controller with Proper HTTP Semantics

```java
package com.enterprise.cms.controller;

import com.enterprise.cms.dto.*;
import com.enterprise.cms.mapper.ContractMapper;
import com.enterprise.cms.service.ContractService;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/v1/contracts")
@RequiredArgsConstructor
public class ContractController {

    private final ContractService contractService;
    private final ContractMapper contractMapper;

    @PostMapping
    public ResponseEntity<ContractResponse> create(
            @Valid @RequestBody CreateContractRequest request) {
        var contract = contractService.create(contractMapper.toEntity(request));
        return ResponseEntity.status(HttpStatus.CREATED)
                .body(contractMapper.toResponse(contract));
    }

    @GetMapping("/{id}")
    public ResponseEntity<ContractResponse> getById(@PathVariable String id) {
        return ResponseEntity.ok(
                contractMapper.toResponse(contractService.getById(id)));
    }

    @GetMapping
    public ResponseEntity<List<ContractResponse>> getByClientId(
            @RequestParam String clientId) {
        return ResponseEntity.ok(contractService.getByClientId(clientId).stream()
                .map(contractMapper::toResponse)
                .toList());
    }

    @PostMapping("/{id}/transition")
    public ResponseEntity<ContractResponse> transition(
            @PathVariable String id,
            @Valid @RequestBody ContractTransitionRequest request) {
        var contract = contractService.transitionStatus(id, request.getNewStatus());
        return ResponseEntity.ok(contractMapper.toResponse(contract));
    }

    @GetMapping("/renewals-due")
    public ResponseEntity<List<ContractResponse>> getRenewalsDue(
            @RequestParam(defaultValue = "30") int daysAhead) {
        return ResponseEntity.ok(contractService.findRenewalsDue(daysAhead).stream()
                .map(contractMapper::toResponse)
                .toList());
    }
}
```

---

## Request/Response DTOs

```java
package com.enterprise.cms.dto;

import jakarta.validation.constraints.*;
import lombok.*;
import java.math.BigDecimal;
import java.time.Instant;
import java.time.LocalDate;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class CreateContractRequest {

    @NotBlank(message = "Client ID is required")
    private String clientId;

    @NotBlank(message = "Contract number is required")
    private String contractNumber;

    @NotNull(message = "Value is required")
    @DecimalMin(value = "0.01", message = "Value must be positive")
    private BigDecimal value;

    @NotBlank(message = "Currency is required")
    @Size(min = 3, max = 3, message = "Currency must be a 3-letter ISO code")
    private String currency;

    @NotNull(message = "Start date is required")
    @FutureOrPresent(message = "Start date must be today or in the future")
    private LocalDate startDate;

    @NotNull(message = "End date is required")
    @Future(message = "End date must be in the future")
    private LocalDate endDate;
}

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class ContractResponse {
    private String id;
    private String clientId;
    private String contractNumber;
    private String status;
    private BigDecimal value;
    private String currency;
    private LocalDate startDate;
    private LocalDate endDate;
    private Instant createdAt;
    private Instant updatedAt;
}
```

---

## Global Exception Handler

```java
package com.enterprise.cms.controller;

import com.enterprise.cms.domain.exception.*;
import com.enterprise.cms.dto.ErrorResponse;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.*;

import java.util.stream.Collectors;

@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {

    @ExceptionHandler(EntityNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(EntityNotFoundException ex) {
        log.warn("Entity not found: {}", ex.getMessage());
        return buildResponse(HttpStatus.NOT_FOUND, ex.getMessage());
    }

    @ExceptionHandler(DuplicateEntityException.class)
    public ResponseEntity<ErrorResponse> handleDuplicate(DuplicateEntityException ex) {
        log.warn("Duplicate entity: {}", ex.getMessage());
        return buildResponse(HttpStatus.CONFLICT, ex.getMessage());
    }

    @ExceptionHandler(InvalidContractTransitionException.class)
    public ResponseEntity<ErrorResponse> handleInvalidTransition(
            InvalidContractTransitionException ex) {
        log.warn("Invalid transition: {}", ex.getMessage());
        return buildResponse(HttpStatus.CONFLICT, ex.getMessage());
    }

    @ExceptionHandler(BusinessRuleViolationException.class)
    public ResponseEntity<ErrorResponse> handleBusinessRule(
            BusinessRuleViolationException ex) {
        log.warn("Business rule violated: {}", ex.getMessage());
        return buildResponse(HttpStatus.UNPROCESSABLE_ENTITY, ex.getMessage());
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(
            MethodArgumentNotValidException ex) {
        String details = ex.getBindingResult().getFieldErrors().stream()
                .map(e -> e.getField() + ": " + e.getDefaultMessage())
                .collect(Collectors.joining(", "));
        return buildResponse(HttpStatus.BAD_REQUEST, details);
    }

    private ResponseEntity<ErrorResponse> buildResponse(HttpStatus status, String message) {
        return ResponseEntity.status(status)
                .body(new ErrorResponse(status.value(), status.getReasonPhrase(), message));
    }
}
```

---

## MapStruct Mapper

```java
package com.enterprise.cms.mapper;

import com.enterprise.cms.domain.model.Contract;
import com.enterprise.cms.domain.model.Money;
import com.enterprise.cms.dto.*;
import org.mapstruct.*;

@Mapper(componentModel = "spring")
public interface ContractMapper {

    @Mapping(target = "id", ignore = true)
    @Mapping(target = "status", ignore = true)
    @Mapping(target = "statusHistory", ignore = true)
    @Mapping(target = "lineItems", ignore = true)
    @Mapping(target = "createdAt", ignore = true)
    @Mapping(target = "updatedAt", ignore = true)
    @Mapping(target = "value", source = ".", qualifiedByName = "toMoney")
    Contract toEntity(CreateContractRequest request);

    @Mapping(source = "status", target = "status")
    @Mapping(source = "value.amount", target = "value")
    @Mapping(source = "value.currency", target = "currency")
    ContractResponse toResponse(Contract contract);

    @Named("toMoney")
    default Money toMoney(CreateContractRequest request) {
        return Money.builder()
                .amount(request.getValue())
                .currency(request.getCurrency())
                .build();
    }
}
```

---

## Domain Event

```java
package com.enterprise.cms.domain.event;

import com.enterprise.cms.domain.model.Contract;
import com.enterprise.cms.domain.model.ContractStatus;
import lombok.Getter;
import org.springframework.context.ApplicationEvent;

@Getter
public class ContractStatusChangedEvent extends ApplicationEvent {

    private final Contract contract;
    private final ContractStatus previousStatus;

    public ContractStatusChangedEvent(Object source, Contract contract,
                                      ContractStatus previousStatus) {
        super(source);
        this.contract = contract;
        this.previousStatus = previousStatus;
    }
}
```
