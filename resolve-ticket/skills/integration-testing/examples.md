# Integration Testing Examples

Complete integration test examples demonstrating Spring Boot Test with Testcontainers and MongoDB.

---

## Base Integration Test Class

```java
package com.enterprise.cms.integration;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.web.client.TestRestTemplate;
import org.springframework.data.mongodb.core.MongoTemplate;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.containers.MongoDBContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;
import org.junit.jupiter.api.BeforeEach;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
public abstract class BaseIntegrationTest {

    @Container
    static MongoDBContainer mongoDBContainer = new MongoDBContainer("mongo:7.0")
            .withReuse(true);

    @Autowired
    protected TestRestTemplate restTemplate;

    @Autowired
    protected MongoTemplate mongoTemplate;

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.data.mongodb.uri", mongoDBContainer::getReplicaSetUrl);
    }

    @BeforeEach
    void cleanDatabase() {
        mongoTemplate.getCollectionNames()
                .forEach(name -> mongoTemplate.dropCollection(name));
    }
}
```

---

## Complete Controller Integration Test

```java
package com.enterprise.cms.integration;

import com.enterprise.cms.domain.model.ClientStatus;
import com.enterprise.cms.dto.*;
import org.junit.jupiter.api.Test;
import org.springframework.http.*;

import java.math.BigDecimal;
import java.util.List;

import static org.hamcrest.MatcherAssert.assertThat;
import static org.hamcrest.Matchers.*;

class ClientControllerIntegrationTest extends BaseIntegrationTest {

    // --- CREATE ---

    @Test
    void test_createClient_validRequest_returns201WithGeneratedId() {
        // Given
        CreateClientRequest request = createValidClientRequest("Acme Corp");

        // When
        ResponseEntity<ClientResponse> response = restTemplate.postForEntity(
                "/api/v1/clients", request, ClientResponse.class);

        // Then
        assertThat(response.getStatusCode(), is(HttpStatus.CREATED));
        ClientResponse body = response.getBody();
        assertThat(body, is(notNullValue()));
        assertThat(body.getId(), is(notNullValue()));
        assertThat(body.getCompanyName(), is("Acme Corp"));
        assertThat(body.getIndustry(), is("Technology"));
        assertThat(body.getStatus(), is(ClientStatus.PROSPECT.name()));
    }

    @Test
    void test_createClient_missingCompanyName_returns400WithDetail() {
        // Given
        CreateClientRequest request = CreateClientRequest.builder()
                .industry("Technology")
                .annualRevenue(BigDecimal.valueOf(1000000))
                .build();

        // When
        ResponseEntity<ErrorResponse> response = restTemplate.postForEntity(
                "/api/v1/clients", request, ErrorResponse.class);

        // Then
        assertThat(response.getStatusCode(), is(HttpStatus.BAD_REQUEST));
        assertThat(response.getBody().getMessage(), containsString("companyName"));
    }

    @Test
    void test_createClient_negativeRevenue_returns400() {
        // Given
        CreateClientRequest request = CreateClientRequest.builder()
                .companyName("Bad Corp")
                .industry("Finance")
                .annualRevenue(BigDecimal.valueOf(-100))
                .build();

        // When
        ResponseEntity<ErrorResponse> response = restTemplate.postForEntity(
                "/api/v1/clients", request, ErrorResponse.class);

        // Then
        assertThat(response.getStatusCode(), is(HttpStatus.BAD_REQUEST));
    }

    // --- READ ---

    @Test
    void test_getClient_existingId_returns200WithFullData() {
        // Given
        String clientId = createClientAndGetId("Read Test Corp");

        // When
        ResponseEntity<ClientResponse> response = restTemplate.getForEntity(
                "/api/v1/clients/" + clientId, ClientResponse.class);

        // Then
        assertThat(response.getStatusCode(), is(HttpStatus.OK));
        assertThat(response.getBody().getCompanyName(), is("Read Test Corp"));
        assertThat(response.getBody().getCreatedAt(), is(notNullValue()));
    }

    @Test
    void test_getClient_nonExistentId_returns404() {
        // When
        ResponseEntity<ErrorResponse> response = restTemplate.getForEntity(
                "/api/v1/clients/nonexistent-id-12345", ErrorResponse.class);

        // Then
        assertThat(response.getStatusCode(), is(HttpStatus.NOT_FOUND));
        assertThat(response.getBody().getMessage(), containsString("Client"));
    }

    // --- LIST ---

    @Test
    void test_listClients_multipleExist_returnsAll() {
        // Given
        createClientAndGetId("Company A");
        createClientAndGetId("Company B");
        createClientAndGetId("Company C");

        // When
        ResponseEntity<ClientResponse[]> response = restTemplate.getForEntity(
                "/api/v1/clients", ClientResponse[].class);

        // Then
        assertThat(response.getStatusCode(), is(HttpStatus.OK));
        assertThat(response.getBody(), arrayWithSize(3));
    }

    @Test
    void test_listClients_noneExist_returnsEmptyArray() {
        // When
        ResponseEntity<ClientResponse[]> response = restTemplate.getForEntity(
                "/api/v1/clients", ClientResponse[].class);

        // Then
        assertThat(response.getStatusCode(), is(HttpStatus.OK));
        assertThat(response.getBody(), arrayWithSize(0));
    }

    // --- UPDATE ---

    @Test
    void test_updateClient_validChanges_returns200WithUpdatedData() {
        // Given
        String clientId = createClientAndGetId("Original Name");
        UpdateClientRequest updateRequest = UpdateClientRequest.builder()
                .companyName("Updated Name")
                .build();

        // When
        restTemplate.put("/api/v1/clients/" + clientId, updateRequest);

        // Then
        ResponseEntity<ClientResponse> getResponse = restTemplate.getForEntity(
                "/api/v1/clients/" + clientId, ClientResponse.class);
        assertThat(getResponse.getBody().getCompanyName(), is("Updated Name"));
        assertThat(getResponse.getBody().getIndustry(), is("Technology")); // unchanged
    }

    // --- DELETE ---

    @Test
    void test_deleteClient_existingId_returns204AndRemovesFromDatabase() {
        // Given
        String clientId = createClientAndGetId("Delete Me Corp");

        // When
        ResponseEntity<Void> deleteResponse = restTemplate.exchange(
                "/api/v1/clients/" + clientId,
                HttpMethod.DELETE, null, Void.class);

        // Then
        assertThat(deleteResponse.getStatusCode(), is(HttpStatus.NO_CONTENT));

        ResponseEntity<ErrorResponse> getResponse = restTemplate.getForEntity(
                "/api/v1/clients/" + clientId, ErrorResponse.class);
        assertThat(getResponse.getStatusCode(), is(HttpStatus.NOT_FOUND));
    }

    // --- DATABASE VERIFICATION ---

    @Test
    void test_createClient_persistsToMongoDB() {
        // Given
        String clientId = createClientAndGetId("Persistence Corp");

        // Then — verify directly in MongoDB
        var stored = mongoTemplate.findById(clientId,
                com.enterprise.cms.domain.model.Client.class);
        assertThat(stored, is(notNullValue()));
        assertThat(stored.getCompanyName(), is("Persistence Corp"));
        assertThat(stored.getCreatedAt(), is(notNullValue()));
        assertThat(stored.getUpdatedAt(), is(notNullValue()));
    }

    // --- HELPERS ---

    private CreateClientRequest createValidClientRequest(String companyName) {
        return CreateClientRequest.builder()
                .companyName(companyName)
                .industry("Technology")
                .annualRevenue(BigDecimal.valueOf(5000000))
                .build();
    }

    private String createClientAndGetId(String companyName) {
        ResponseEntity<ClientResponse> response = restTemplate.postForEntity(
                "/api/v1/clients",
                createValidClientRequest(companyName),
                ClientResponse.class);
        assertThat(response.getStatusCode(), is(HttpStatus.CREATED));
        return response.getBody().getId();
    }
}
```

---

## Workflow Integration Test

```java
package com.enterprise.cms.integration;

import com.enterprise.cms.domain.model.ContractStatus;
import com.enterprise.cms.dto.*;
import org.junit.jupiter.api.Test;
import org.springframework.http.*;

import java.math.BigDecimal;
import java.time.LocalDate;

import static org.hamcrest.MatcherAssert.assertThat;
import static org.hamcrest.Matchers.*;

class ContractLifecycleIntegrationTest extends BaseIntegrationTest {

    @Test
    void test_fullContractLifecycle_fromDraftToExpired() {
        // STEP 1: Create a client
        String clientId = createClient("Lifecycle Corp");

        // STEP 2: Create a contract (starts as DRAFT)
        ResponseEntity<ContractResponse> createResponse = restTemplate.postForEntity(
                "/api/v1/contracts",
                CreateContractRequest.builder()
                        .clientId(clientId)
                        .contractNumber("CNT-LIFE-001")
                        .value(BigDecimal.valueOf(100000))
                        .currency("USD")
                        .startDate(LocalDate.now())
                        .endDate(LocalDate.now().plusYears(1))
                        .build(),
                ContractResponse.class);
        assertThat(createResponse.getStatusCode(), is(HttpStatus.CREATED));
        String contractId = createResponse.getBody().getId();
        assertThat(createResponse.getBody().getStatus(), is("DRAFT"));

        // STEP 3: DRAFT -> PENDING_APPROVAL
        ContractResponse afterPending = transitionAndVerify(
                contractId, ContractStatus.PENDING_APPROVAL, "PENDING_APPROVAL");

        // STEP 4: PENDING_APPROVAL -> ACTIVE
        ContractResponse afterActive = transitionAndVerify(
                contractId, ContractStatus.ACTIVE, "ACTIVE");

        // STEP 5: ACTIVE -> EXPIRED
        ContractResponse afterExpired = transitionAndVerify(
                contractId, ContractStatus.EXPIRED, "EXPIRED");

        // STEP 6: Verify status history has 3 transitions
        var stored = mongoTemplate.findById(contractId,
                com.enterprise.cms.domain.model.Contract.class);
        assertThat(stored.getStatusHistory(), hasSize(3));
    }

    @Test
    void test_invalidTransition_returns409Conflict() {
        // Given
        String clientId = createClient("Conflict Corp");
        String contractId = createContract(clientId, "CNT-CONFLICT-001");

        // When — try invalid DRAFT -> ACTIVE (must go through PENDING_APPROVAL)
        ResponseEntity<ErrorResponse> response = restTemplate.postForEntity(
                "/api/v1/contracts/" + contractId + "/transition",
                new ContractTransitionRequest(ContractStatus.ACTIVE),
                ErrorResponse.class);

        // Then
        assertThat(response.getStatusCode(), is(HttpStatus.CONFLICT));
        assertThat(response.getBody().getMessage(), containsString("DRAFT"));
        assertThat(response.getBody().getMessage(), containsString("ACTIVE"));
    }

    // --- HELPERS ---

    private String createClient(String name) {
        ResponseEntity<ClientResponse> response = restTemplate.postForEntity(
                "/api/v1/clients",
                CreateClientRequest.builder()
                        .companyName(name).industry("Tech")
                        .annualRevenue(BigDecimal.valueOf(1000000)).build(),
                ClientResponse.class);
        return response.getBody().getId();
    }

    private String createContract(String clientId, String contractNumber) {
        ResponseEntity<ContractResponse> response = restTemplate.postForEntity(
                "/api/v1/contracts",
                CreateContractRequest.builder()
                        .clientId(clientId).contractNumber(contractNumber)
                        .value(BigDecimal.valueOf(50000)).currency("USD")
                        .startDate(LocalDate.now())
                        .endDate(LocalDate.now().plusYears(1)).build(),
                ContractResponse.class);
        return response.getBody().getId();
    }

    private ContractResponse transitionAndVerify(
            String contractId, ContractStatus status, String expectedStatusName) {
        ResponseEntity<ContractResponse> response = restTemplate.postForEntity(
                "/api/v1/contracts/" + contractId + "/transition",
                new ContractTransitionRequest(status),
                ContractResponse.class);
        assertThat(response.getStatusCode(), is(HttpStatus.OK));
        assertThat(response.getBody().getStatus(), is(expectedStatusName));
        return response.getBody();
    }
}
```
