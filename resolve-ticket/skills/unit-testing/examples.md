# Unit Testing Examples

Complete test class examples demonstrating all unit testing patterns.

---

## Complete Service Test Class

```java
package com.enterprise.cms.service;

import com.enterprise.cms.domain.event.ContractStatusChangedEvent;
import com.enterprise.cms.domain.exception.DuplicateEntityException;
import com.enterprise.cms.domain.exception.EntityNotFoundException;
import com.enterprise.cms.domain.exception.InvalidContractTransitionException;
import com.enterprise.cms.domain.model.*;
import com.enterprise.cms.repository.ContractRepository;
import com.enterprise.cms.testdata.ContractBuilder;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.Arguments;
import org.junit.jupiter.params.provider.MethodSource;
import org.mockito.ArgumentCaptor;
import org.mockito.Captor;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import org.springframework.context.ApplicationEventPublisher;

import java.util.List;
import java.util.Optional;
import java.util.stream.Stream;

import static org.hamcrest.MatcherAssert.assertThat;
import static org.hamcrest.Matchers.*;
import static org.junit.jupiter.api.Assertions.assertThrows;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class ContractServiceTest {

    @Mock
    private ContractRepository contractRepository;

    @Mock
    private ApplicationEventPublisher eventPublisher;

    @InjectMocks
    private ContractService contractService;

    @Captor
    private ArgumentCaptor<ContractStatusChangedEvent> eventCaptor;

    // --- create ---

    @Test
    void test_create_validContract_setsStatusToDraftAndSaves() {
        // Given
        Contract contract = ContractBuilder.newBuilder()
                .withDefaults()
                .withContractNumber("CNT-001")
                .build();
        when(contractRepository.existsByContractNumber("CNT-001")).thenReturn(false);
        when(contractRepository.save(any(Contract.class))).thenAnswer(i -> i.getArgument(0));

        // When
        Contract result = contractService.create(contract);

        // Then
        assertThat(result.getStatus(), is(ContractStatus.DRAFT));
        verify(contractRepository).save(contract);
    }

    @Test
    void test_create_duplicateContractNumber_throwsDuplicateException() {
        // Given
        Contract contract = ContractBuilder.newBuilder()
                .withDefaults()
                .withContractNumber("CNT-EXISTING")
                .build();
        when(contractRepository.existsByContractNumber("CNT-EXISTING")).thenReturn(true);

        // When / Then
        DuplicateEntityException exception = assertThrows(
                DuplicateEntityException.class,
                () -> contractService.create(contract));

        assertThat(exception.getMessage(), containsString("CNT-EXISTING"));
        verify(contractRepository, never()).save(any());
    }

    // --- getById ---

    @Test
    void test_getById_existingContract_returnsContract() {
        // Given
        Contract contract = ContractBuilder.newBuilder()
                .withDefaults()
                .withId("contract-1")
                .build();
        when(contractRepository.findById("contract-1")).thenReturn(Optional.of(contract));

        // When
        Contract result = contractService.getById("contract-1");

        // Then
        assertThat(result, is(notNullValue()));
        assertThat(result.getId(), is("contract-1"));
    }

    @Test
    void test_getById_nonExistentId_throwsNotFoundException() {
        // Given
        when(contractRepository.findById("bad-id")).thenReturn(Optional.empty());

        // When / Then
        EntityNotFoundException exception = assertThrows(
                EntityNotFoundException.class,
                () -> contractService.getById("bad-id"));

        assertThat(exception.getMessage(), containsString("Contract"));
        assertThat(exception.getMessage(), containsString("bad-id"));
    }

    // --- transitionStatus ---

    @Test
    void test_transitionStatus_draftToPending_updatesStatusAndHistory() {
        // Given
        Contract contract = ContractBuilder.newBuilder()
                .withDefaults()
                .withId("c-1")
                .withStatus(ContractStatus.DRAFT)
                .build();
        when(contractRepository.findById("c-1")).thenReturn(Optional.of(contract));
        when(contractRepository.save(any())).thenAnswer(i -> i.getArgument(0));

        // When
        Contract result = contractService.transitionStatus("c-1", ContractStatus.PENDING_APPROVAL);

        // Then
        assertThat(result.getStatus(), is(ContractStatus.PENDING_APPROVAL));
        assertThat(result.getStatusHistory(), hasSize(1));
        assertThat(result.getStatusHistory().get(0).getFromStatus(), is(ContractStatus.DRAFT));
        assertThat(result.getStatusHistory().get(0).getToStatus(), is(ContractStatus.PENDING_APPROVAL));
    }

    @Test
    void test_transitionStatus_validTransition_publishesEvent() {
        // Given
        Contract contract = ContractBuilder.newBuilder()
                .withDefaults()
                .withId("c-1")
                .withStatus(ContractStatus.DRAFT)
                .build();
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
                Arguments.of(ContractStatus.PENDING_APPROVAL, ContractStatus.DRAFT),
                Arguments.of(ContractStatus.ACTIVE, ContractStatus.RENEWAL),
                Arguments.of(ContractStatus.ACTIVE, ContractStatus.TERMINATED),
                Arguments.of(ContractStatus.ACTIVE, ContractStatus.EXPIRED),
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
        verify(contractRepository, never()).save(any());
        verify(eventPublisher, never()).publishEvent(any());
    }

    private static Stream<Arguments> invalidTransitions() {
        return Stream.of(
                Arguments.of(ContractStatus.DRAFT, ContractStatus.ACTIVE),
                Arguments.of(ContractStatus.DRAFT, ContractStatus.EXPIRED),
                Arguments.of(ContractStatus.EXPIRED, ContractStatus.ACTIVE),
                Arguments.of(ContractStatus.EXPIRED, ContractStatus.DRAFT),
                Arguments.of(ContractStatus.TERMINATED, ContractStatus.ACTIVE),
                Arguments.of(ContractStatus.TERMINATED, ContractStatus.DRAFT)
        );
    }

    // --- getByClientId ---

    @Test
    void test_getByClientId_existingClient_returnsContracts() {
        // Given
        List<Contract> contracts = List.of(
                ContractBuilder.newBuilder().withDefaults().withClientId("client-1").build(),
                ContractBuilder.newBuilder().withDefaults().withClientId("client-1").build());
        when(contractRepository.findByClientId("client-1")).thenReturn(contracts);

        // When
        List<Contract> result = contractService.getByClientId("client-1");

        // Then
        assertThat(result, hasSize(2));
    }

    @Test
    void test_getByClientId_noContracts_returnsEmptyList() {
        // Given
        when(contractRepository.findByClientId("client-none")).thenReturn(List.of());

        // When
        List<Contract> result = contractService.getByClientId("client-none");

        // Then
        assertThat(result, is(empty()));
    }
}
```

---

## Test Data Builder

```java
package com.enterprise.cms.testdata;

import com.enterprise.cms.domain.model.*;

import java.math.BigDecimal;
import java.time.LocalDate;
import java.util.ArrayList;
import java.util.List;
import java.util.UUID;

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

    public ContractBuilder withContractNumber(String contractNumber) {
        this.contractNumber = contractNumber;
        return this;
    }

    public ContractBuilder withStatus(ContractStatus status) {
        this.status = status;
        return this;
    }

    public ContractBuilder withValue(BigDecimal amount, String currency) {
        this.value = new Money(amount, currency);
        return this;
    }

    public ContractBuilder withDateRange(LocalDate start, LocalDate end) {
        this.startDate = start;
        this.endDate = end;
        return this;
    }

    public ContractBuilder withLineItems(List<LineItem> lineItems) {
        this.lineItems = lineItems;
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

---

## Mapper Test

```java
package com.enterprise.cms.mapper;

import com.enterprise.cms.domain.model.*;
import com.enterprise.cms.dto.*;
import org.junit.jupiter.api.Test;
import org.mapstruct.factory.Mappers;

import java.math.BigDecimal;
import java.time.LocalDate;

import static org.hamcrest.MatcherAssert.assertThat;
import static org.hamcrest.Matchers.*;

class ContractMapperTest {

    private final ContractMapper mapper = Mappers.getMapper(ContractMapper.class);

    @Test
    void test_toEntity_validRequest_mapsAllFields() {
        // Given
        CreateContractRequest request = CreateContractRequest.builder()
                .clientId("client-1")
                .contractNumber("CNT-001")
                .value(BigDecimal.valueOf(5000))
                .currency("EUR")
                .startDate(LocalDate.of(2026, 1, 1))
                .endDate(LocalDate.of(2027, 1, 1))
                .build();

        // When
        Contract result = mapper.toEntity(request);

        // Then
        assertThat(result.getClientId(), is("client-1"));
        assertThat(result.getContractNumber(), is("CNT-001"));
        assertThat(result.getValue().getAmount(), comparesEqualTo(BigDecimal.valueOf(5000)));
        assertThat(result.getValue().getCurrency(), is("EUR"));
        assertThat(result.getStartDate(), is(LocalDate.of(2026, 1, 1)));
        assertThat(result.getEndDate(), is(LocalDate.of(2027, 1, 1)));
        assertThat(result.getId(), is(nullValue()));
        assertThat(result.getStatus(), is(nullValue()));
    }

    @Test
    void test_toResponse_fullContract_mapsAllFields() {
        // Given
        Contract contract = Contract.builder()
                .id("c-1")
                .clientId("client-1")
                .contractNumber("CNT-001")
                .status(ContractStatus.ACTIVE)
                .value(new Money(BigDecimal.valueOf(10000), "USD"))
                .startDate(LocalDate.of(2026, 1, 1))
                .endDate(LocalDate.of(2027, 1, 1))
                .build();

        // When
        ContractResponse result = mapper.toResponse(contract);

        // Then
        assertThat(result.getId(), is("c-1"));
        assertThat(result.getClientId(), is("client-1"));
        assertThat(result.getContractNumber(), is("CNT-001"));
        assertThat(result.getStatus(), is("ACTIVE"));
        assertThat(result.getValue(), comparesEqualTo(BigDecimal.valueOf(10000)));
        assertThat(result.getCurrency(), is("USD"));
    }
}
```
