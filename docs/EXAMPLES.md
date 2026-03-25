# TwinMapper — Examples

## Overview

This document provides end-to-end examples covering all major TwinMapper use cases. Each example includes the definition DSL, generated code shape, and runtime usage.

---

## Example 1 — Basic DTO Generation and YAML Binding

### Definition File

```yaml
# src/main/twinmapper/schemas/product-model.yaml

definitionSet:
  name: product-model
  version: 1.0.0
  package: com.example.product.generated

types:
  Product:
    kind: object
    fields:
      id:
        type: string
        required: true

      name:
        type: string
        required: true
        aliases: [productName, product_name]

      price:
        type: decimal
        required: true
        constraints:
          min: 0

      category:
        type: ProductCategory
        required: true

      tags:
        type: list<string>
        required: false
        default: []

  ProductCategory:
    kind: enum
    values:
      - name: ELECTRONICS
        code: electronics
      - name: CLOTHING
        code: clothing
      - name: FOOD
        code: food
```

### Generated DTO

```java
// Generated — do not edit
public record ProductDto(
    String id,
    String name,
    BigDecimal price,
    ProductCategory category,
    List<String> tags
) {}
```

### Generated Enum

```java
// Generated — do not edit
public enum ProductCategory {
    ELECTRONICS("electronics"),
    CLOTHING("clothing"),
    FOOD("food");

    private final String code;

    ProductCategory(String code) { this.code = code; }
    public String getCode() { return code; }

    public static ProductCategory fromCode(String code) {
        for (ProductCategory v : values()) {
            if (v.code.equals(code)) return v;
        }
        throw new InvalidEnumCodeException(ProductCategory.class, code);
    }
}
```

### Input YAML Document

```yaml
# Runtime input
id: PROD-001
productName: Wireless Headphones   # alias resolved
price: 129.99
category: electronics
```

### Runtime Usage

```java
@Service
public class ProductService {

    private final TwinMapperRuntime runtime;

    public ProductService(TwinMapperRuntime runtime) {
        this.runtime = runtime;
    }

    public ProductDto loadProduct(InputStream yaml) {
        return runtime.readYaml(yaml, ProductDto.class);
    }
}
```

---

## Example 2 — JSON Binding with Validation

### Definition File

```yaml
# src/main/twinmapper/schemas/order-model.yaml

definitionSet:
  name: order-model
  version: 1.0.0
  package: com.example.order.generated

types:
  CreateOrderRequest:
    kind: object
    fields:
      customerId:
        type: string
        required: true
        aliases: [customer_id]

      items:
        type: list<OrderItem>
        required: true
        constraints:
          minLength: 1

      shippingAddress:
        type: Address
        required: true

  OrderItem:
    kind: object
    fields:
      productId:
        type: string
        required: true
      quantity:
        type: integer
        required: true
        constraints:
          min: 1
          max: 100

  Address:
    kind: object
    fields:
      line1:
        type: string
        required: true
      city:
        type: string
        required: true
      postcode:
        type: string
        required: true
        constraints:
          pattern: "^[A-Z0-9 ]{3,8}$"
```

### Spring MVC Controller Usage

```java
@RestController
@RequestMapping("/orders")
public class OrderController {

    private final TwinMapperRuntime runtime;
    private final OrderService orderService;

    @PostMapping(consumes = MediaType.APPLICATION_JSON_VALUE)
    public ResponseEntity<OrderResponse> createOrder(
            HttpServletRequest request) throws IOException {

        BindingResult<CreateOrderRequestDto> result =
            runtime.readJsonSafe(request.getInputStream(), CreateOrderRequestDto.class);

        if (result.hasErrors()) {
            return ResponseEntity.badRequest()
                .body(OrderResponse.validationFailed(result.errors()));
        }

        return ResponseEntity.ok(orderService.create(result.value()));
    }
}
```

---

## Example 3 — Entity to Domain Mapping

### Mapping Definition File

```yaml
# src/main/twinmapper/mappings/customer-mappings.yaml

objectMappings:

  - name: CustomerEntityToDomain
    sourceType: com.example.persistence.CustomerEntity
    targetType: com.example.domain.Customer
    mode: CREATE
    fields:
      - source: id
        target: id
        converter: longToCustomerId

      - source: customerCode
        target: code

      - source: statusCode
        target: status
        converter: statusCodeToCustomerStatus

      - source: city
        target: address.city

      - source: postalCode
        target: address.postcode

      - source: countryCode
        target: address.country

  - name: CustomerDomainToEntity
    sourceType: com.example.domain.Customer
    targetType: com.example.persistence.CustomerEntity
    mode: UPDATE
    nullPolicy: IGNORE_NULLS
    inverseOf: CustomerEntityToDomain
    fields:
      - source: id.value
        target: id

      - source: code
        target: customerCode

      - source: status.code
        target: statusCode

      - source: address.city
        target: city

      - source: address.postcode
        target: postalCode

      - source: address.country
        target: countryCode
```

### Converter Registration

```java
@Configuration
public class CustomerConverterConfig implements TwinMapperRuntimeConfigurer {

    @Override
    public void addConverters(ConverterRegistry registry) {
        registry.addConverter(Long.class, CustomerId.class,
            id -> id != null ? new CustomerId(id) : null);

        registry.addConverter(String.class, CustomerStatus.class,
            CustomerStatus::fromCode);
    }
}
```

### Runtime Usage

```java
@Service
@Transactional
public class CustomerRepository {

    private final TwinObjectMapperRegistry mapperRegistry;
    private final JpaCustomerRepository jpa;

    public Customer findById(CustomerId id) {
        CustomerEntity entity = jpa.findById(id.value())
            .orElseThrow(() -> new CustomerNotFoundException(id));

        TwinObjectMapper<CustomerEntity, Customer> mapper =
            mapperRegistry.mapperFor(CustomerEntity.class, Customer.class);

        return mapper.map(entity);
    }

    public void save(Customer customer) {
        CustomerEntity existing = jpa.findById(customer.id().value())
            .orElse(new CustomerEntity());

        TwinUpdateMapper<Customer, CustomerEntity> mapper =
            mapperRegistry.updateMapperFor(Customer.class, CustomerEntity.class);

        CustomerEntity updated = mapper.update(customer, existing);
        jpa.save(updated);
    }
}
```

---

## Example 4 — Patch DTO Mapping

### Mapping Definition

```yaml
objectMappings:
  - name: CustomerPatch
    sourceType: com.example.dto.CustomerPatchDto
    targetType: com.example.persistence.CustomerEntity
    mode: PATCH
    profile: patchProfile
    fields:
      - source: name
        target: customerCode
      - source: email
        target: emailAddress
      - source: status
        target: statusCode
        converter: customerStatusToCode
```

### Patch DTO

```java
// Handwritten — all fields optional for patch semantics
public record CustomerPatchDto(
    @Nullable String name,
    @Nullable String email,
    @Nullable CustomerStatus status
) {}
```

### Usage

```java
@PatchMapping("/{id}")
public ResponseEntity<Void> patchCustomer(
        @PathVariable Long id,
        @RequestBody CustomerPatchDto patch) {

    CustomerEntity existing = customerRepo.findById(id)
        .orElseThrow(CustomerNotFoundException::new);

    TwinPatchMapper<CustomerPatchDto, CustomerEntity> patchMapper =
        mapperRegistry.patchMapperFor(CustomerPatchDto.class, CustomerEntity.class);

    CustomerEntity patched = patchMapper.patch(patch, existing);
    customerRepo.save(patched);
    return ResponseEntity.noContent().build();
}
```

---

## Example 5 — BPMN Document Binding

### BPMN File (Runtime Input)

```xml
<!-- src/main/resources/workflows/leave-approval.bpmn -->
<?xml version="1.0" encoding="UTF-8"?>
<definitions xmlns="http://www.omg.org/spec/BPMN/20100524/MODEL"
             id="leave-approval"
             name="Leave Approval Process">
  <process id="leave-approval" isExecutable="true">
    <startEvent id="start" name="Leave Request Submitted"/>
    <userTask id="manager-review" name="Manager Review"/>
    <exclusiveGateway id="decision" name="Decision"/>
    <endEvent id="approved" name="Approved"/>
    <endEvent id="rejected" name="Rejected"/>
  </process>
</definitions>
```

### Runtime Binding

```java
@Service
public class WorkflowLoader {

    private final TwinMapperRuntime runtime;

    public WorkflowDefinitionDto loadFromBpmn(String classpathPath) {
        Resource resource = new ClassPathResource(classpathPath);
        try (InputStream is = resource.getInputStream()) {
            return runtime.readBpmn(is, WorkflowDefinitionDto.class);
        } catch (IOException e) {
            throw new WorkflowLoadException(classpathPath, e);
        }
    }
}
```

---

## Example 6 — Reusable Profiles

### Profile Definition

```yaml
profiles:
  - name: auditPreservingUpdate
    nullPolicy: IGNORE_NULLS
    unmappedTargetPolicy: WARN
    ignoreTargets:
      - createdAt
      - createdBy
      - version
      - lastModifiedAt

  - name: fullReplace
    nullPolicy: SET_NULLS
    unmappedTargetPolicy: ERROR

objectMappings:
  - name: InvoiceUpdate
    sourceType: com.example.domain.Invoice
    targetType: com.example.persistence.InvoiceEntity
    mode: UPDATE
    profile: auditPreservingUpdate
    fields:
      - source: number
        target: invoiceNumber
      - source: amount.value
        target: amountCents
        converter: moneyToCents
      - source: status
        target: statusCode
        converter: statusToCode
```

---

## Example 7 — Validation in Spring MVC

### Controller with Validation

```java
@RestController
@RequestMapping("/customers")
public class CustomerController {

    private final TwinMapperRuntime runtime;
    private final CustomerDtoValidator validator;
    private final CustomerService service;

    @PostMapping
    public ResponseEntity<CustomerResponse> create(
            @RequestBody @Valid CreateCustomerRequest request,
            BindingResult bindingResult) {

        if (bindingResult.hasErrors()) {
            return ResponseEntity.unprocessableEntity()
                .body(CustomerResponse.invalid(bindingResult));
        }

        // Map request to domain command
        TwinObjectMapper<CreateCustomerRequest, CreateCustomerCommand> mapper =
            mapperRegistry.mapperFor(
                CreateCustomerRequest.class,
                CreateCustomerCommand.class
            );

        CreateCustomerCommand command = mapper.map(request);
        Customer created = service.create(command);
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(CustomerResponse.of(created));
    }
}
```

---

## Example 8 — Spring Boot Test with @AutoConfigureTwinMapper

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.NONE)
@AutoConfigureTwinMapper
class ProductBindingTest {

    @Autowired
    TwinMapperRuntime runtime;

    @Test
    void shouldBindProductYaml() {
        String yaml = """
            id: PROD-001
            productName: Wireless Headphones
            price: 129.99
            category: electronics
            """;

        ProductDto product = runtime.readYaml(
            new ByteArrayInputStream(yaml.getBytes()),
            ProductDto.class
        );

        assertThat(product.id()).isEqualTo("PROD-001");
        assertThat(product.name()).isEqualTo("Wireless Headphones");
        assertThat(product.category()).isEqualTo(ProductCategory.ELECTRONICS);
        assertThat(product.tags()).isEmpty();    // default applied
    }

    @Test
    void shouldRejectMissingRequiredField() {
        String yaml = """
            name: Headphones
            price: 99.99
            category: electronics
            """; // missing 'id'

        assertThatThrownBy(() ->
            runtime.readYaml(new ByteArrayInputStream(yaml.getBytes()), ProductDto.class))
            .isInstanceOf(BindingException.class)
            .hasMessageContaining("MISSING_REQUIRED_FIELD")
            .hasMessageContaining("id");
    }
}
```

---

## Example 9 — Custom SPI Registration

```java
@Configuration
public class TwinMapperExtensions implements TwinMapperRuntimeConfigurer {

    @Override
    public void addConverters(ConverterRegistry registry) {
        registry.addConverter(new MoneyConverter());
        registry.addConverter(new CurrencyCodeConverter());
        registry.addConverterFactory(new EnumCodeConverterFactory());
    }

    @Override
    public void addValidationExtensions(ValidationExtensionRegistry registry) {
        registry.add(new InvoiceBusinessRulesValidator());
        registry.add(new CustomerCreditLimitValidator());
    }
}
```

---

## Example 10 — Gradle Configuration

```groovy
// build.gradle
plugins {
    id 'org.springframework.boot' version '3.2.0'
    id 'com.twinmapper.codegen' version '1.0.0'
}

dependencies {
    implementation 'com.twinmapper:twinmapper-spring-boot-starter:1.0.0'
    testImplementation 'com.twinmapper:twinmapper-spring-boot-starter:1.0.0'
}

twinmapper {
    schemaDir = 'src/main/twinmapper/schemas'
    mappingDir = 'src/main/twinmapper/mappings'
    outputPackage = 'com.example.generated'
    strictness = 'STRICT'
    immutableRecords = true
}
```

```yaml
# application.yaml
twinmapper:
  strictness: STRICT
  definition-locations:
    - classpath:/twinmapper/schemas/
    - classpath:/twinmapper/mappings/
  generation:
    base-package: com.example.generated
    immutable-dtos: true
    generate-validators: true
    generate-mappers: true
```
