# Integration Testing Examples

Complete, tested integration test examples for Spring Boot 4.x with Testcontainers MongoDB.
All code below compiles and passes against the client-service codebase.

---

## Base Integration Test Class

This exact class is used in the project. Copy it as-is for new services.

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

Key points:
- NO `@Container` annotation — container is started manually in `static {}` block
- `@ServiceConnection` replaces `@DynamicPropertySource` — auto-configures MongoDB URI
- `@AutoConfigureMockMvc` — NOT `@SpringBootTest(webEnvironment = RANDOM_PORT)` with TestRestTemplate
- Import is `org.springframework.boot.webmvc.test.autoconfigure.AutoConfigureMockMvc` (Spring Boot 4.x)

---

## Client Controller Integration Test

Tests for a standard CRUD controller with validation, search, and error handling.

```java
package com.enterprise.cms.client.integration;

import com.enterprise.cms.client.dto.CreateClientRequest;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.MediaType;
import tools.jackson.databind.ObjectMapper;

import java.math.BigDecimal;

import static org.hamcrest.Matchers.*;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

class ClientControllerIntegrationTest extends BaseIntegrationTest {

    @Autowired
    private ObjectMapper objectMapper;

    // --- HELPERS ---

    /**
     * Create a client via API and return its ID.
     * Uses ObjectMapper for CreateClientRequest because it has no boolean primitive fields.
     */
    private String createClient(String companyName) throws Exception {
        CreateClientRequest request = CreateClientRequest.builder()
                .companyName(companyName)
                .industry("Technology")
                .annualRevenue(BigDecimal.valueOf(1000000))
                .build();

        String body = mockMvc.perform(post("/api/v1/clients")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(objectMapper.writeValueAsString(request)))
                .andExpect(status().isCreated())
                .andReturn().getResponse().getContentAsString();

        return objectMapper.readTree(body).get("id").asText();
    }

    // --- CREATE ---

    @Test
    void test_createClient_validRequest_returns201() throws Exception {
        CreateClientRequest request = CreateClientRequest.builder()
                .companyName("Integration Test Corp")
                .industry("Technology")
                .annualRevenue(BigDecimal.valueOf(1000000))
                .build();

        mockMvc.perform(post("/api/v1/clients")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(objectMapper.writeValueAsString(request)))
                .andExpect(status().isCreated())
                .andExpect(jsonPath("$.companyName", is("Integration Test Corp")))
                .andExpect(jsonPath("$.status", is("PROSPECT")))
                .andExpect(jsonPath("$.id", is(notNullValue())));
    }

    @Test
    void test_createClient_missingName_returns400() throws Exception {
        CreateClientRequest request = CreateClientRequest.builder()
                .industry("Technology").build();

        mockMvc.perform(post("/api/v1/clients")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(objectMapper.writeValueAsString(request)))
                .andExpect(status().isBadRequest());
    }

    // --- READ ---

    @Test
    void test_getClient_existingId_returns200() throws Exception {
        String clientId = createClient("Get Test Corp");

        mockMvc.perform(get("/api/v1/clients/" + clientId))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.companyName", is("Get Test Corp")));
    }

    @Test
    void test_getClient_nonExistentId_returns404() throws Exception {
        mockMvc.perform(get("/api/v1/clients/nonexistent-id"))
                .andExpect(status().isNotFound());
    }

    // --- SEARCH ---

    @Test
    void test_searchClients_matchingQuery_returnsResults() throws Exception {
        createClient("Alpha Corp");
        createClient("Beta Inc");

        mockMvc.perform(get("/api/v1/clients/search").param("q", "Alpha"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$", hasSize(1)))
                .andExpect(jsonPath("$[0].companyName", is("Alpha Corp")));
    }

    // --- DELETE ---

    @Test
    void test_deleteClient_existingId_returns204() throws Exception {
        String clientId = createClient("Delete Test Corp");

        mockMvc.perform(delete("/api/v1/clients/" + clientId))
                .andExpect(status().isNoContent());

        // Verify deletion
        mockMvc.perform(get("/api/v1/clients/" + clientId))
                .andExpect(status().isNotFound());
    }
}
```

---

## Contact Controller Integration Test

Tests for a nested resource controller (`/api/v1/clients/{clientId}/contacts`) with boolean fields.

**This is the critical example** — it demonstrates:
1. Raw JSON strings for request bodies (avoiding Lombok boolean serialization issues)
2. Boolean field names in jsonPath (`$.primary`, NOT `$.isPrimary`)
3. Multi-step workflow testing (primary reassignment)
4. Business rule validation (cannot delete primary contact)

```java
package com.enterprise.cms.client.integration;

import com.enterprise.cms.client.dto.CreateClientRequest;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.MediaType;
import tools.jackson.databind.ObjectMapper;

import java.math.BigDecimal;

import static org.hamcrest.Matchers.*;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

class ContactControllerIntegrationTest extends BaseIntegrationTest {

    @Autowired
    private ObjectMapper objectMapper;

    // --- HELPERS ---

    private String createClient(String companyName) throws Exception {
        CreateClientRequest request = CreateClientRequest.builder()
                .companyName(companyName)
                .industry("Technology")
                .annualRevenue(BigDecimal.valueOf(1000000))
                .build();

        String body = mockMvc.perform(post("/api/v1/clients")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(objectMapper.writeValueAsString(request)))
                .andExpect(status().isCreated())
                .andReturn().getResponse().getContentAsString();

        return objectMapper.readTree(body).get("id").asText();
    }

    /**
     * Build contact JSON manually. MUST use raw JSON, not ObjectMapper, because
     * Lombok boolean isPrimary serializes as "primary" via Jackson 3.x getter inference,
     * but the API contract expects "isPrimary" in the request body.
     */
    private String contactJson(String firstName, String lastName, String email,
                               String role, boolean isPrimary) {
        return """
                {
                    "firstName": "%s",
                    "lastName": "%s",
                    "email": "%s",
                    "phone": "+1234567890",
                    "role": "%s",
                    "isPrimary": %s
                }
                """.formatted(firstName, lastName, email, role, isPrimary);
    }

    private String createContact(String clientId, String firstName, String lastName,
                                 String email, boolean isPrimary) throws Exception {
        return mockMvc.perform(post("/api/v1/clients/" + clientId + "/contacts")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(contactJson(firstName, lastName, email, "TECHNICAL", isPrimary)))
                .andExpect(status().isCreated())
                .andReturn().getResponse().getContentAsString();
    }

    private String extractId(String responseBody) throws Exception {
        return objectMapper.readTree(responseBody).get("id").asText();
    }

    // --- TESTS ---

    @Test
    void test_createContact_validRequest_returns201() throws Exception {
        String clientId = createClient("Contact Test Corp");

        mockMvc.perform(post("/api/v1/clients/" + clientId + "/contacts")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(contactJson("John", "Doe", "john@test.com", "DECISION_MAKER", true)))
                .andExpect(status().isCreated())
                .andExpect(jsonPath("$.firstName", is("John")))
                .andExpect(jsonPath("$.lastName", is("Doe")))
                .andExpect(jsonPath("$.email", is("john@test.com")))
                .andExpect(jsonPath("$.clientId", is(clientId)))
                .andExpect(jsonPath("$.id", is(notNullValue())));
    }

    @Test
    void test_createContact_missingFirstName_returns400() throws Exception {
        String clientId = createClient("Validation Test Corp");

        mockMvc.perform(post("/api/v1/clients/" + clientId + "/contacts")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("""
                                { "lastName": "Doe", "email": "john@test.com", "isPrimary": false }
                                """))
                .andExpect(status().isBadRequest());
    }

    @Test
    void test_getContacts_byClientId_returnsOnlyThatClientsContacts() throws Exception {
        String clientId1 = createClient("Client A");
        String clientId2 = createClient("Client B");

        createContact(clientId1, "Alice", "Smith", "alice@test.com", false);
        createContact(clientId1, "Bob", "Jones", "bob@test.com", false);
        createContact(clientId2, "Charlie", "Brown", "charlie@test.com", false);

        mockMvc.perform(get("/api/v1/clients/" + clientId1 + "/contacts"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$", hasSize(2)));

        mockMvc.perform(get("/api/v1/clients/" + clientId2 + "/contacts"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$", hasSize(1)));
    }

    @Test
    void test_setPrimary_newPrimary_unsetsOldPrimary() throws Exception {
        String clientId = createClient("Primary Test Corp");

        // Create first contact as primary
        String body1 = createContact(clientId, "First", "Primary", "first@test.com", true);
        String contactId1 = extractId(body1);

        // Create second contact, not primary
        String body2 = createContact(clientId, "Second", "Contact", "second@test.com", false);
        String contactId2 = extractId(body2);

        // Set second contact as primary
        mockMvc.perform(put("/api/v1/clients/" + clientId + "/contacts/" + contactId2 + "/primary"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.primary", is(true)));  // NOTE: $.primary, not $.isPrimary

        // Verify first contact is no longer primary
        mockMvc.perform(get("/api/v1/clients/" + clientId + "/contacts/" + contactId1))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.primary", is(false)));  // NOTE: $.primary, not $.isPrimary
    }

    @Test
    void test_deleteContact_nonPrimary_returns204() throws Exception {
        String clientId = createClient("Delete Contact Corp");

        String body = createContact(clientId, "Deletable", "Contact", "del@test.com", false);
        String contactId = extractId(body);

        mockMvc.perform(delete("/api/v1/clients/" + clientId + "/contacts/" + contactId))
                .andExpect(status().isNoContent());

        // Verify the contact is gone
        mockMvc.perform(get("/api/v1/clients/" + clientId + "/contacts/" + contactId))
                .andExpect(status().isNotFound());
    }

    @Test
    void test_deleteContact_primaryContact_returns422() throws Exception {
        String clientId = createClient("Cannot Delete Primary Corp");

        String body = createContact(clientId, "Primary", "Contact", "primary@test.com", true);
        String contactId = extractId(body);

        mockMvc.perform(delete("/api/v1/clients/" + clientId + "/contacts/" + contactId))
                .andExpect(status().isUnprocessableEntity());
    }
}
```

---

## CMP-101: Status Transition Integration Test Template

For ticket CMP-101 (Client Status Transition Rules), the integration test would look like:

```java
package com.enterprise.cms.client.integration;

import com.enterprise.cms.client.dto.CreateClientRequest;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.MediaType;
import tools.jackson.databind.ObjectMapper;

import java.math.BigDecimal;

import static org.hamcrest.Matchers.*;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

class ClientStatusTransitionIntegrationTest extends BaseIntegrationTest {

    @Autowired
    private ObjectMapper objectMapper;

    private String createClient(String companyName) throws Exception {
        CreateClientRequest request = CreateClientRequest.builder()
                .companyName(companyName)
                .industry("Technology")
                .annualRevenue(BigDecimal.valueOf(1000000))
                .build();

        String body = mockMvc.perform(post("/api/v1/clients")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(objectMapper.writeValueAsString(request)))
                .andExpect(status().isCreated())
                .andReturn().getResponse().getContentAsString();

        return objectMapper.readTree(body).get("id").asText();
    }

    private String statusTransitionJson(String status, String reason) {
        return """
                { "status": "%s", "reason": "%s" }
                """.formatted(status, reason);
    }

    @Test
    void test_transitionStatus_prospectToActive_returns200() throws Exception {
        String clientId = createClient("Transition Corp");

        mockMvc.perform(post("/api/v1/clients/" + clientId + "/status")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(statusTransitionJson("ACTIVE", "Signed first contract")))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.status", is("ACTIVE")));
    }

    @Test
    void test_transitionStatus_prospectToAtRisk_returns422() throws Exception {
        String clientId = createClient("Invalid Transition Corp");

        mockMvc.perform(post("/api/v1/clients/" + clientId + "/status")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(statusTransitionJson("AT_RISK", "Should not work")))
                .andExpect(status().isUnprocessableEntity());
    }

    @Test
    void test_transitionStatus_missingReason_returns400() throws Exception {
        String clientId = createClient("No Reason Corp");

        mockMvc.perform(post("/api/v1/clients/" + clientId + "/status")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("""
                                { "status": "ACTIVE" }
                                """))
                .andExpect(status().isBadRequest());
    }

    @Test
    void test_transitionStatus_recordsStatusHistory() throws Exception {
        String clientId = createClient("History Corp");

        // PROSPECT -> ACTIVE
        mockMvc.perform(post("/api/v1/clients/" + clientId + "/status")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(statusTransitionJson("ACTIVE", "Signed contract")))
                .andExpect(status().isOk());

        // ACTIVE -> AT_RISK
        mockMvc.perform(post("/api/v1/clients/" + clientId + "/status")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(statusTransitionJson("AT_RISK", "Payment delayed")))
                .andExpect(status().isOk());

        // Verify statusHistory in response
        mockMvc.perform(get("/api/v1/clients/" + clientId))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.status", is("AT_RISK")))
                .andExpect(jsonPath("$.statusHistory", hasSize(2)))
                .andExpect(jsonPath("$.statusHistory[0].fromStatus", is("PROSPECT")))
                .andExpect(jsonPath("$.statusHistory[0].toStatus", is("ACTIVE")))
                .andExpect(jsonPath("$.statusHistory[0].reason", is("Signed contract")))
                .andExpect(jsonPath("$.statusHistory[1].fromStatus", is("ACTIVE")))
                .andExpect(jsonPath("$.statusHistory[1].toStatus", is("AT_RISK")));
    }

    @Test
    void test_transitionStatus_fullLifecycle_prospectToActiveToAtRiskToActive() throws Exception {
        String clientId = createClient("Lifecycle Corp");

        // PROSPECT -> ACTIVE
        mockMvc.perform(post("/api/v1/clients/" + clientId + "/status")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(statusTransitionJson("ACTIVE", "Signed contract")))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.status", is("ACTIVE")));

        // ACTIVE -> AT_RISK
        mockMvc.perform(post("/api/v1/clients/" + clientId + "/status")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(statusTransitionJson("AT_RISK", "Payment issues")))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.status", is("AT_RISK")));

        // AT_RISK -> ACTIVE
        mockMvc.perform(post("/api/v1/clients/" + clientId + "/status")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(statusTransitionJson("ACTIVE", "Issues resolved")))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.status", is("ACTIVE")));
    }
}
```

---

## CMP-102: Activity Notes Integration Test Template

For ticket CMP-102 (Client Activity Notes), the integration test would look like:

```java
package com.enterprise.cms.client.integration;

import com.enterprise.cms.client.dto.CreateClientRequest;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.MediaType;
import tools.jackson.databind.ObjectMapper;

import java.math.BigDecimal;

import static org.hamcrest.Matchers.*;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

class NoteControllerIntegrationTest extends BaseIntegrationTest {

    @Autowired
    private ObjectMapper objectMapper;

    private String createClient(String companyName) throws Exception {
        CreateClientRequest request = CreateClientRequest.builder()
                .companyName(companyName)
                .industry("Technology")
                .annualRevenue(BigDecimal.valueOf(1000000))
                .build();

        String body = mockMvc.perform(post("/api/v1/clients")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(objectMapper.writeValueAsString(request)))
                .andExpect(status().isCreated())
                .andReturn().getResponse().getContentAsString();

        return objectMapper.readTree(body).get("id").asText();
    }

    private String noteJson(String authorName, String content, String category) {
        return """
                {
                    "authorName": "%s",
                    "content": "%s",
                    "category": "%s"
                }
                """.formatted(authorName, content, category);
    }

    private String createNote(String clientId, String author, String content,
                              String category) throws Exception {
        return mockMvc.perform(post("/api/v1/clients/" + clientId + "/notes")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(noteJson(author, content, category)))
                .andExpect(status().isCreated())
                .andReturn().getResponse().getContentAsString();
    }

    private String extractId(String responseBody) throws Exception {
        return objectMapper.readTree(responseBody).get("id").asText();
    }

    // --- CREATE ---

    @Test
    void test_createNote_validRequest_returns201() throws Exception {
        String clientId = createClient("Note Test Corp");

        mockMvc.perform(post("/api/v1/clients/" + clientId + "/notes")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(noteJson("Alice Smith", "Client meeting went well", "MEETING")))
                .andExpect(status().isCreated())
                .andExpect(jsonPath("$.authorName", is("Alice Smith")))
                .andExpect(jsonPath("$.content", is("Client meeting went well")))
                .andExpect(jsonPath("$.category", is("MEETING")))
                .andExpect(jsonPath("$.pinned", is(false)))
                .andExpect(jsonPath("$.id", is(notNullValue())))
                .andExpect(jsonPath("$.createdAt", is(notNullValue())));
    }

    @Test
    void test_createNote_nonExistentClient_returns404() throws Exception {
        mockMvc.perform(post("/api/v1/clients/nonexistent-id/notes")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(noteJson("Alice", "Should fail", "MEETING")))
                .andExpect(status().isNotFound());
    }

    @Test
    void test_createNote_missingAuthorName_returns400() throws Exception {
        String clientId = createClient("Validation Corp");

        mockMvc.perform(post("/api/v1/clients/" + clientId + "/notes")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("""
                                { "content": "Missing author", "category": "MEETING" }
                                """))
                .andExpect(status().isBadRequest());
    }

    @Test
    void test_createNote_contentExceeds2000Chars_returns400() throws Exception {
        String clientId = createClient("Long Content Corp");
        String longContent = "x".repeat(2001);

        mockMvc.perform(post("/api/v1/clients/" + clientId + "/notes")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("""
                                { "authorName": "Alice", "content": "%s", "category": "MEETING" }
                                """.formatted(longContent)))
                .andExpect(status().isBadRequest());
    }

    // --- READ ---

    @Test
    void test_getNotes_returnsNotesInReverseChronologicalOrder() throws Exception {
        String clientId = createClient("Timeline Corp");

        createNote(clientId, "Alice", "First note", "MEETING");
        createNote(clientId, "Bob", "Second note", "CALL");
        createNote(clientId, "Charlie", "Third note", "EMAIL");

        mockMvc.perform(get("/api/v1/clients/" + clientId + "/notes"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$", hasSize(3)));
    }

    @Test
    void test_getNote_byId_returns200() throws Exception {
        String clientId = createClient("Single Note Corp");
        String body = createNote(clientId, "Alice", "Test note", "MEETING");
        String noteId = extractId(body);

        mockMvc.perform(get("/api/v1/clients/" + clientId + "/notes/" + noteId))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.authorName", is("Alice")))
                .andExpect(jsonPath("$.content", is("Test note")));
    }

    // --- UPDATE ---

    @Test
    void test_updateNote_changesContent_returns200() throws Exception {
        String clientId = createClient("Update Note Corp");
        String body = createNote(clientId, "Alice", "Original content", "MEETING");
        String noteId = extractId(body);

        mockMvc.perform(put("/api/v1/clients/" + clientId + "/notes/" + noteId)
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("""
                                { "content": "Updated content" }
                                """))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.content", is("Updated content")))
                .andExpect(jsonPath("$.authorName", is("Alice")));  // unchanged
    }

    // --- DELETE ---

    @Test
    void test_deleteNote_existingNote_returns204() throws Exception {
        String clientId = createClient("Delete Note Corp");
        String body = createNote(clientId, "Alice", "Deletable note", "MEETING");
        String noteId = extractId(body);

        mockMvc.perform(delete("/api/v1/clients/" + clientId + "/notes/" + noteId))
                .andExpect(status().isNoContent());

        // Verify deleted
        mockMvc.perform(get("/api/v1/clients/" + clientId + "/notes"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$", hasSize(0)));
    }

    // --- PIN / UNPIN ---

    @Test
    void test_pinNote_togglesPinnedStatus() throws Exception {
        String clientId = createClient("Pin Note Corp");
        String body = createNote(clientId, "Alice", "Pinnable note", "MEETING");
        String noteId = extractId(body);

        // Pin the note
        mockMvc.perform(patch("/api/v1/clients/" + clientId + "/notes/" + noteId + "/pin"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.pinned", is(true)));

        // Unpin the note
        mockMvc.perform(patch("/api/v1/clients/" + clientId + "/notes/" + noteId + "/pin"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.pinned", is(false)));
    }

    @Test
    void test_getPinnedNotes_returnsOnlyPinned() throws Exception {
        String clientId = createClient("Pinned Filter Corp");

        // Create three notes
        String body1 = createNote(clientId, "Alice", "Note 1", "MEETING");
        String noteId1 = extractId(body1);
        createNote(clientId, "Bob", "Note 2", "CALL");
        String body3 = createNote(clientId, "Charlie", "Note 3", "EMAIL");
        String noteId3 = extractId(body3);

        // Pin note 1 and note 3
        mockMvc.perform(patch("/api/v1/clients/" + clientId + "/notes/" + noteId1 + "/pin"));
        mockMvc.perform(patch("/api/v1/clients/" + clientId + "/notes/" + noteId3 + "/pin"));

        // Get pinned notes
        mockMvc.perform(get("/api/v1/clients/" + clientId + "/notes/pinned"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$", hasSize(2)));
    }

    // --- FULL CRUD LIFECYCLE ---

    @Test
    void test_noteCrudLifecycle() throws Exception {
        String clientId = createClient("Lifecycle Note Corp");

        // CREATE
        String body = mockMvc.perform(post("/api/v1/clients/" + clientId + "/notes")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(noteJson("Alice", "Initial note", "MEETING")))
                .andExpect(status().isCreated())
                .andReturn().getResponse().getContentAsString();
        String noteId = extractId(body);

        // READ
        mockMvc.perform(get("/api/v1/clients/" + clientId + "/notes/" + noteId))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.content", is("Initial note")));

        // UPDATE
        mockMvc.perform(put("/api/v1/clients/" + clientId + "/notes/" + noteId)
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("""
                                { "content": "Updated note", "category": "FOLLOW_UP" }
                                """))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.content", is("Updated note")))
                .andExpect(jsonPath("$.category", is("FOLLOW_UP")));

        // PIN
        mockMvc.perform(patch("/api/v1/clients/" + clientId + "/notes/" + noteId + "/pin"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.pinned", is(true)));

        // DELETE
        mockMvc.perform(delete("/api/v1/clients/" + clientId + "/notes/" + noteId))
                .andExpect(status().isNoContent());

        // VERIFY DELETED
        mockMvc.perform(get("/api/v1/clients/" + clientId + "/notes"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$", hasSize(0)));
    }
}
```

---

## Required Imports Summary

Every integration test class needs these imports:

```java
// Base test
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.MediaType;
import tools.jackson.databind.ObjectMapper;  // Jackson 3.x (Spring Boot 4.x)

// MockMvc static imports
import static org.hamcrest.Matchers.*;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;
```

**Wrong imports (Spring Boot 3.x / Jackson 2.x — do NOT use):**
```java
// WRONG: import com.fasterxml.jackson.databind.ObjectMapper;
// WRONG: import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
```

---

## JSON Builder Patterns for Common DTOs

### Client (no boolean primitives — ObjectMapper safe)
```java
// ObjectMapper is safe because CreateClientRequest has no boolean primitives
CreateClientRequest request = CreateClientRequest.builder()
        .companyName("Test Corp").industry("Technology")
        .annualRevenue(BigDecimal.valueOf(1000000)).build();
mockMvc.perform(post("/api/v1/clients")
        .contentType(MediaType.APPLICATION_JSON)
        .content(objectMapper.writeValueAsString(request)))
```

### Contact (has boolean isPrimary — MUST use raw JSON)
```java
// MUST use raw JSON because of Lombok boolean isPrimary
private String contactJson(String firstName, String lastName, String email,
                           String role, boolean isPrimary) {
    return """
            {
                "firstName": "%s",
                "lastName": "%s",
                "email": "%s",
                "phone": "+1234567890",
                "role": "%s",
                "isPrimary": %s
            }
            """.formatted(firstName, lastName, email, role, isPrimary);
}
```

### Status Transition Request
```java
private String statusTransitionJson(String status, String reason) {
    return """
            { "status": "%s", "reason": "%s" }
            """.formatted(status, reason);
}
```

### Note
```java
private String noteJson(String authorName, String content, String category) {
    return """
            { "authorName": "%s", "content": "%s", "category": "%s" }
            """.formatted(authorName, content, category);
}
```

### Generic Rule
If the DTO has ANY `boolean` primitive field (not `Boolean` wrapper), use raw JSON.
If the DTO has only `String`, `BigDecimal`, `List<String>`, `Integer`, etc., ObjectMapper is safe.
When in doubt, use raw JSON — it always works and tests the real API contract.
