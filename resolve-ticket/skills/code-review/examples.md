# Code Review Examples

Real examples demonstrating self-review patterns and common findings.

---

## Acceptance Criteria Mapping

For ticket **CMS-42: Add client activity summary endpoint**, map each acceptance criterion to code and tests:

| Acceptance Criterion | Implemented In | Tested In |
|---------------------|---------------|-----------|
| "Returns activity summary for a client" | `ClientActivityService.getSummary()` | `ClientActivityServiceTest.test_getSummary_*` |
| "Includes contract count and total value" | `ActivitySummaryMapper.toResponse()` | `ActivitySummaryMapperTest.test_toResponse_*` |
| "Returns 404 for unknown client ID" | `GlobalExceptionHandler.handleEntityNotFound()` | `ClientActivityIntegrationTest.test_getSummary_unknownClient_returns404` |
| "Validates client ID format" | `@Pattern` on `clientId` path variable | `ClientActivityIntegrationTest.test_getSummary_invalidIdFormat_returns400` |

If any row is missing either column, the work is not complete. Go back and add the missing implementation or test before committing.

---

## Common Review Findings

### Debug Code Left In

**Before** (violation of CR03):

```java
public Invoice generateInvoice(String contractId) {
    Contract contract = contractRepository.findById(contractId)
            .orElseThrow(() -> new EntityNotFoundException("Contract", contractId));

    System.out.println("DEBUG: contract status = " + contract.getStatus());

    Invoice invoice = invoiceService.createFromContract(contract);

    // TODO: add discount logic later
    // invoice.applyDiscount(discountService.calculate(contract));

    System.out.println("DEBUG: invoice total = " + invoice.getTotal());
    return invoice;
}
```

**After** (clean):

```java
public Invoice generateInvoice(String contractId) {
    Contract contract = contractRepository.findById(contractId)
            .orElseThrow(() -> new EntityNotFoundException("Contract", contractId));

    log.info("Generating invoice for contract: {}", contractId);

    Invoice invoice = invoiceService.createFromContract(contract);

    return invoice;
}
```

Removed: `System.out.println` debug statements, commented-out code block, `// TODO` comment. Replaced with structured logging using the contract ID (not PII).

---

### Inconsistent Naming

**Before** (violation of CR05):

```java
public class ClientService {

    public Client findByStatus(ClientStatus status) { /* ... */ }

    public Client statusFind(ClientStatus status) { /* ... */ }  // wrong pattern

    public List<Contract> getClientContracts(String cId) { /* ... */ }  // abbreviated parameter
}
```

**After** (consistent):

```java
public class ClientService {

    public Client findByStatus(ClientStatus status) { /* ... */ }

    public List<Contract> findContractsByClientId(String clientId) { /* ... */ }
}
```

Fixed: method name follows `verbNoun` pattern, parameter uses full name `clientId` instead of `cId`, removed duplicate method.

---

### Missing Error Handling

**Before** (violation of CR09):

```java
public Contract renewContract(String contractId) {
    Contract contract = contractRepository.findById(contractId).get();
    contract.setStatus(ContractStatus.RENEWAL);
    contract.setEndDate(contract.getEndDate().plusYears(1));
    return contractRepository.save(contract);
}
```

**After** (complete error handling):

```java
public Contract renewContract(String contractId) {
    Contract contract = contractRepository.findById(contractId)
            .orElseThrow(() -> new EntityNotFoundException("Contract", contractId));

    if (contract.getStatus() != ContractStatus.ACTIVE) {
        throw new InvalidContractTransitionException(
                contract.getStatus(), ContractStatus.RENEWAL);
    }

    contract.setStatus(ContractStatus.RENEWAL);
    contract.setEndDate(contract.getEndDate().plusYears(1));

    log.info("Contract renewed: {}", contractId);
    return contractRepository.save(contract);
}
```

Fixed: `Optional.get()` replaced with `orElseThrow`, added business rule validation for status transition, added logging with ID only.

---

### Sensitive Data in Logs

**Before** (violation of CR07):

```java
public Client createClient(CreateClientRequest request) {
    log.info("Creating client: name={}, email={}, phone={}",
            request.getName(), request.getEmail(), request.getPhone());

    Client client = clientMapper.toEntity(request);
    Client saved = clientRepository.save(client);

    log.info("Client created: {}", saved);
    return saved;
}
```

**After** (no PII in logs):

```java
public Client createClient(CreateClientRequest request) {
    log.info("Creating client");

    Client client = clientMapper.toEntity(request);
    Client saved = clientRepository.save(client);

    log.info("Client created: id={}", saved.getId());
    return saved;
}
```

Fixed: removed name, email, and phone from log statements. Removed `toString()` dump of entire entity. Logged only the generated ID after creation.

---

## Security Review Examples

### Mass Assignment Vulnerability in DTO

**Before** (violation of CR06 — mass assignment):

```java
public class UpdateClientRequest {
    private String name;
    private String email;
    private String phone;
    private String id;           // should not be settable by user
    private String role;         // should not be settable by user
    private String internalNote; // internal field exposed to API
    // getters and setters for all fields
}
```

**After** (safe DTO):

```java
public class UpdateClientRequest {

    @NotBlank
    private String name;

    @Email
    @NotBlank
    private String email;

    @Pattern(regexp = "^\\+?[0-9\\-\\s]{7,15}$")
    private String phone;

    // id, role, and internalNote are NOT included — they are
    // set by the system, never accepted from user input.
}
```

Fixed: removed `id`, `role`, and `internalNote` from the request DTO. These fields are set internally by the service layer. Added validation annotations to remaining fields.

---

### Injection via Raw Query

**Before** (violation of CR06 — injection):

```java
@Repository
public class InvoiceCustomRepository {

    @Autowired
    private MongoTemplate mongoTemplate;

    public List<Invoice> searchByNotes(String keyword) {
        String json = "{ 'notes': { '$regex': '" + keyword + "' } }";
        BasicQuery query = new BasicQuery(json);
        return mongoTemplate.find(query, Invoice.class);
    }
}
```

**After** (parameterized query):

```java
@Repository
public class InvoiceCustomRepository {

    @Autowired
    private MongoTemplate mongoTemplate;

    public List<Invoice> searchByNotes(String keyword) {
        Query query = new Query(
                Criteria.where("notes").regex(Pattern.quote(keyword), "i"));
        return mongoTemplate.find(query, Invoice.class);
    }
}
```

Fixed: replaced raw JSON string concatenation with Spring Data `Criteria` API. Used `Pattern.quote()` to escape regex metacharacters in user input.

---

### PII in API Response

**Before** (violation of CR06 — sensitive data exposure):

```java
@GetMapping("/api/v1/contracts/{id}")
public ResponseEntity<Contract> getContract(@PathVariable String id) {
    Contract contract = contractService.getById(id);
    return ResponseEntity.ok(contract);
}
```

Returning the raw domain entity exposes internal fields (`createdBy`, `internalNotes`, `auditTrail`) that should not be visible to API consumers.

**After** (response DTO):

```java
@GetMapping("/api/v1/contracts/{id}")
public ResponseEntity<ContractResponse> getContract(@PathVariable String id) {
    Contract contract = contractService.getById(id);
    ContractResponse response = contractMapper.toResponse(contract);
    return ResponseEntity.ok(response);
}
```

Fixed: map domain entity to a response DTO that includes only the fields intended for the consumer. Internal fields are excluded from `ContractResponse`.

---

## Layer Boundary Violations

### Business Logic in Controller

**Before** (violation of CR08):

```java
@RestController
@RequestMapping("/api/v1/invoices")
public class InvoiceController {

    @Autowired
    private InvoiceRepository invoiceRepository;

    @Autowired
    private ContractRepository contractRepository;

    @PostMapping
    public ResponseEntity<InvoiceResponse> createInvoice(
            @RequestBody CreateInvoiceRequest request) {

        // Business logic does not belong in the controller
        Contract contract = contractRepository.findById(request.getContractId())
                .orElseThrow(() -> new EntityNotFoundException("Contract", request.getContractId()));

        if (contract.getStatus() != ContractStatus.ACTIVE) {
            throw new IllegalStateException("Cannot invoice a non-active contract");
        }

        Invoice invoice = new Invoice();
        invoice.setContractId(contract.getId());
        invoice.setClientId(contract.getClientId());
        invoice.setAmount(contract.getValue().getAmount());
        invoice.setStatus(InvoiceStatus.DRAFT);
        invoice.setIssuedDate(LocalDate.now());

        Invoice saved = invoiceRepository.save(invoice);
        return ResponseEntity.status(HttpStatus.CREATED).body(invoiceMapper.toResponse(saved));
    }
}
```

**After** (controller delegates to service):

```java
@RestController
@RequestMapping("/api/v1/invoices")
@RequiredArgsConstructor
public class InvoiceController {

    private final InvoiceService invoiceService;
    private final InvoiceMapper invoiceMapper;

    @PostMapping
    public ResponseEntity<InvoiceResponse> createInvoice(
            @Valid @RequestBody CreateInvoiceRequest request) {

        Invoice invoice = invoiceService.createFromContract(request.getContractId());
        InvoiceResponse response = invoiceMapper.toResponse(invoice);
        return ResponseEntity.status(HttpStatus.CREATED).body(response);
    }
}
```

Fixed: moved all business logic (contract lookup, status validation, invoice construction) into `InvoiceService`. Controller only handles HTTP concerns: deserializing the request, calling the service, mapping to a response, and setting the status code.

---

### Repository Access from Controller

**Before** (violation of CR08):

```java
@RestController
@RequestMapping("/api/v1/clients")
public class ClientController {

    @Autowired
    private ClientRepository clientRepository;

    @GetMapping("/{id}/contracts")
    public ResponseEntity<List<ContractResponse>> getClientContracts(
            @PathVariable String id) {

        if (!clientRepository.existsById(id)) {
            throw new EntityNotFoundException("Client", id);
        }

        List<Contract> contracts = contractRepository.findByClientId(id);
        return ResponseEntity.ok(contracts.stream()
                .map(contractMapper::toResponse)
                .toList());
    }
}
```

**After** (controller uses service layer only):

```java
@RestController
@RequestMapping("/api/v1/clients")
@RequiredArgsConstructor
public class ClientController {

    private final ClientService clientService;
    private final ContractMapper contractMapper;

    @GetMapping("/{id}/contracts")
    public ResponseEntity<List<ContractResponse>> getClientContracts(
            @PathVariable String id) {

        List<Contract> contracts = clientService.getContractsByClientId(id);
        return ResponseEntity.ok(contracts.stream()
                .map(contractMapper::toResponse)
                .toList());
    }
}
```

Fixed: removed direct `clientRepository` and `contractRepository` imports from the controller. The existence check and contract lookup are now handled inside `ClientService.getContractsByClientId()`, which throws `EntityNotFoundException` if the client does not exist.
