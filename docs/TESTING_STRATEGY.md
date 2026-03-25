# TwinMapper — Testing Strategy

## Overview

This document defines how TwinMapper itself is tested (internal testing strategy) and how consuming applications should test their TwinMapper-based code (consumer testing guidance). Both are covered.

---

## Internal Testing Principles

- Every generated artifact must be covered by a round-trip test.
- No test should rely on reflection to verify generated behavior.
- Security-sensitive paths (BPMN XXE, SnakeYAML safe construction) must have explicit security tests.
- SPI ordering must be verified by tests that register multiple implementations.
- All Spring integration points must have Spring Boot integration tests using `@SpringBootTest`.

---

## Test Layers

### Unit Tests

Unit tests cover individual module behavior with no Spring context.

| Module | What to unit test |
|---|---|
| `twinmapper-definition-model` | Meta-model construction, equality, field validation |
| `twinmapper-codegen` | DTO shape from a given `DefinitionSet`, enum values, binder structure |
| `twinmapper-validation` | Each constraint type independently, conditional rules, one-of, mutually exclusive |
| `twinmapper-format-yaml` | YAML DSL parsing, safe construction enforcement, alias resolution |
| `twinmapper-format-json` | JSON definition reading, field mapping, error reporting |
| `twinmapper-format-bpmn` | BPMN intermediate model correctness, XXE rejection, supported element parsing |
| `twinmapper-runtime-binding` | NodeCursor behavior, binder dispatch, error accumulation |
| `twinmapper-runtime-objectmap` | Mapper dispatch, null policies, patch semantics, converter invocation |

### Integration Tests

Integration tests verify module interactions and Spring context wiring.

```java
@SpringBootTest(classes = TwinMapperTestApplication.class)
@AutoConfigureTwinMapper
class CustomerBindingIntegrationTest {

    @Autowired
    TwinMapperRuntime twinMapperRuntime;

    @Test
    void shouldBindValidYamlDocumentToCustomerDto() throws Exception {
        InputStream yaml = getClass().getResourceAsStream("/fixtures/customer-valid.yaml");
        CustomerDto result = twinMapperRuntime.readYaml(yaml, CustomerDto.class);

        assertThat(result.id()).isEqualTo("CUST-001");
        assertThat(result.fullName()).isEqualTo("Jane Smith");
        assertThat(result.status()).isEqualTo(CustomerStatus.ACTIVE);
    }

    @Test
    void shouldRejectUnknownFieldsInStrictMode() {
        InputStream yaml = getClass().getResourceAsStream("/fixtures/customer-unknown-field.yaml");

        assertThatThrownBy(() -> twinMapperRuntime.readYaml(yaml, CustomerDto.class))
            .isInstanceOf(BindingException.class)
            .hasMessageContaining("UNKNOWN_FIELD");
    }
}
```

### Round-Trip Tests

Every generated binder must have a round-trip test verifying that a known valid document produces the expected DTO with correct field values, defaults, and alias resolution.

```java
class CustomerDtoBinderTest {

    private final CustomerDtoBinder binder = new CustomerDtoBinder();

    @Test
    void shouldResolveAliasedFieldName() {
        NodeCursor cursor = YamlNodeCursor.of("""
            id: CUST-001
            name: Jane Smith        # alias for fullName
            status: active
            """);

        CustomerDto dto = binder.bind(cursor, BindingContext.strict());

        assertThat(dto.fullName()).isEqualTo("Jane Smith");
    }

    @Test
    void shouldApplyDefaultForAbsentOptionalField() {
        NodeCursor cursor = YamlNodeCursor.of("""
            id: CUST-002
            name: John Doe
            status: active
            """);

        CustomerDto dto = binder.bind(cursor, BindingContext.strict());

        assertThat(dto.tags()).isEqualTo(List.of());    // default applied
    }
}
```

---

## Security Tests

Security tests are mandatory for BPMN and YAML parsers.

### BPMN XXE Test

```java
class BpmnXxeSecurityTest {

    @Test
    void shouldRejectXxePayloadInBpmnDocument() {
        String xxePayload = """
            <?xml version="1.0" encoding="UTF-8"?>
            <!DOCTYPE foo [
              <!ENTITY xxe SYSTEM "file:///etc/passwd">
            ]>
            <definitions xmlns="http://www.omg.org/spec/BPMN/20100524/MODEL">
              <process id="&xxe;" />
            </definitions>
            """;

        InputStream input = new ByteArrayInputStream(xxePayload.getBytes());

        assertThatThrownBy(() -> new BpmnDocumentParser().parse(input, ParsingContext.defaults()))
            .isInstanceOf(DocumentParseException.class);
    }
}
```

### SnakeYAML Safe Construction Test

```java
class YamlSafeConstructionTest {

    @Test
    void shouldRejectArbitraryJavaObjectInstantiationViaYaml() {
        String maliciousYaml = """
            !!com.example.SensitiveClass
              field: value
            """;

        assertThatThrownBy(() -> new SafeYamlParser().parse(maliciousYaml))
            .isInstanceOf(YAMLException.class);
    }
}
```

---

## Consumer Testing Guidance

### Using @AutoConfigureTwinMapper

`@AutoConfigureTwinMapper` is the recommended way to include TwinMapper in Spring Boot test slices. It activates TwinMapper's auto-configuration without requiring a full application context.

```java
@SpringBootTest
@AutoConfigureTwinMapper
class OrderMappingTest {

    @Autowired
    TwinObjectMapperRegistry mapperRegistry;

    @Test
    void shouldMapOrderEntityToOrderDomain() {
        TwinObjectMapper<OrderEntity, Order> mapper =
            mapperRegistry.mapperFor(OrderEntity.class, Order.class);

        OrderEntity entity = buildTestEntity();
        Order domain = mapper.map(entity);

        assertThat(domain.id().value()).isEqualTo(entity.getId());
        assertThat(domain.status()).isEqualTo(OrderStatus.CONFIRMED);
    }
}
```

### Testing Generated Validators

```java
@SpringBootTest
@AutoConfigureTwinMapper
class CustomerValidatorTest {

    @Autowired
    CustomerDtoValidator validator;

    @Test
    void shouldRejectDtoWithMissingRequiredField() {
        CustomerDto dto = new CustomerDto(null, "Jane", CustomerStatus.ACTIVE, null);

        BeanPropertyBindingResult errors = new BeanPropertyBindingResult(dto, "customerDto");
        validator.validate(dto, errors);

        assertThat(errors.hasErrors()).isTrue();
        assertThat(errors.getFieldError("id")).isNotNull();
    }

    @Test
    void shouldPassValidDto() {
        CustomerDto dto = new CustomerDto("CUST-001", "Jane Smith", CustomerStatus.ACTIVE, null);

        BeanPropertyBindingResult errors = new BeanPropertyBindingResult(dto, "customerDto");
        validator.validate(dto, errors);

        assertThat(errors.hasErrors()).isFalse();
    }
}
```

### Testing Binding Errors

```java
@SpringBootTest
@AutoConfigureTwinMapper
class BindingErrorTest {

    @Autowired
    TwinMapperRuntime runtime;

    @Test
    void shouldProducePathAwareErrorForMissingRequiredField() {
        String yaml = "status: active"; // missing required 'id'

        BindingResult<CustomerDto> result =
            runtime.readYamlSafe(new ByteArrayInputStream(yaml.getBytes()), CustomerDto.class);

        assertThat(result.hasErrors()).isTrue();
        assertThat(result.errors())
            .anyMatch(e -> e.fieldPath().equals("id")
                        && e.errorCode().equals("MISSING_REQUIRED_FIELD"));
    }
}
```

### Testing Update and Patch Mappers

```java
@SpringBootTest
@AutoConfigureTwinMapper
class CustomerUpdateMapperTest {

    @Autowired
    TwinMapperRuntime runtime;

    @Test
    void shouldNotOverwriteExistingValueWithNullInUpdateMode() {
        CustomerEntity existing = new CustomerEntity();
        existing.setId(1L);
        existing.setCustomerCode("CUST-001");
        existing.setStatusCode("active");

        Customer domain = new Customer(
            new CustomerId(1L),
            null,          // code is null in source
            CustomerStatus.SUSPENDED,
            null
        );

        CustomerEntity updated = runtime.update(domain, existing);

        assertThat(updated.getCustomerCode()).isEqualTo("CUST-001");    // preserved
        assertThat(updated.getStatusCode()).isEqualTo("suspended");     // updated
    }
}
```

---

## Test Fixtures

Organize test fixtures by format and scenario.

```
src/test/resources/fixtures/
├── yaml/
│   ├── customer-valid.yaml
│   ├── customer-missing-required.yaml
│   ├── customer-unknown-field.yaml
│   ├── customer-deprecated-alias.yaml
│   └── customer-forbidden-field.yaml
├── json/
│   ├── order-valid.json
│   └── order-invalid-enum.json
└── bpmn/
    ├── simple-workflow.bpmn
    ├── xxe-payload.xml
    └── unsupported-element.bpmn
```

---

## Test Slice Support

TwinMapper provides a dedicated `@AutoConfigureTwinMapper` annotation for use in Spring Boot test slices. This activates only the TwinMapper auto-configuration without triggering unrelated auto-configurations.

```java
// Minimum context for binding tests — no web layer, no database
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.NONE)
@AutoConfigureTwinMapper
class BindingOnlyTest {
    // Only TwinMapper beans loaded
}
```

---

## SPI Ordering Tests

When multiple SPI implementations are registered, verify ordering behavior explicitly.

```java
@SpringBootTest
@AutoConfigureTwinMapper
class SpiOrderingTest {

    @Autowired
    List<RuntimeDocumentParser> parsers;

    @Test
    void shouldResolveHigherPriorityParserFirst() {
        assertThat(parsers.get(0)).isInstanceOf(CustomHighPriorityParser.class);
        assertThat(parsers.get(1)).isInstanceOf(DefaultJsonParser.class);
    }
}
```

---

## CI Recommendations

- Run all unit tests on every commit.
- Run all integration tests on every pull request.
- Run security tests (XXE, SnakeYAML) as part of the integration test suite.
- Run round-trip tests for every supported format.
- Verify generated source determinism: run codegen twice and assert byte-for-byte identical output.
- Include a test that verifies `AutoConfiguration.imports` is correctly formed and all listed classes exist on the classpath.
