# TwinMapper — Migration Guide

## Overview

This guide covers migration scenarios: migrating definition schemas across versions, migrating from other mapping frameworks to TwinMapper, and handling deprecated or renamed fields in existing documents.

---

## Migrating Definition Schemas

### Adding a new optional field (non-breaking)

Adding an optional field to an existing definition is always safe.

```yaml
# Before
types:
  Customer:
    kind: object
    fields:
      id:
        type: string
        required: true
      name:
        type: string
        required: true

# After — adding optional email field
types:
  Customer:
    kind: object
    fields:
      id:
        type: string
        required: true
      name:
        type: string
        required: true
      email:                      # new optional field
        type: string
        required: false
```

Existing documents without `email` bind successfully. The field is absent and null in the generated DTO.

---

### Renaming a field (with alias for backward compatibility)

Add the old name as an alias so existing documents continue to work.

```yaml
# Before
fields:
  customerCode:
    type: string
    required: true

# After — renamed to 'code', old name kept as alias
fields:
  code:
    type: string
    required: true
    aliases: [customerCode]
```

Existing documents using `customerCode` continue to bind. New documents use `code`. The alias is resolved transparently.

---

### Deprecating a field name

When the old name should be discouraged but still works during a transition period:

```yaml
fields:
  code:
    type: string
    required: true
    aliases: [customerCode]
    deprecatedAliases: [cust_code, custCode]
```

- `customerCode` — accepted, no warning
- `cust_code` — accepted with WARNING in COMPATIBLE mode, ERROR in STRICT mode
- Documents migrated to `code` — no issues

---

### Removing a field (with forbidden declaration)

When a field must be rejected entirely:

```yaml
types:
  WorkflowDefinition:
    kind: object
    forbiddenFields:
      - interrupting    # forbidden since v2.0, replaced by interruptingMode
    fields:
      interruptingMode:
        type: InterruptingMode
        required: false
        default: INTERRUPTING
```

Documents containing `interrupting` produce `FORBIDDEN_FIELD` error with migration hint. The migration hint is configured via:

```yaml
forbiddenFieldHints:
  interrupting: "Use 'interruptingMode' instead. See migration guide section 3.2."
```

---

### Versioning a definition set

When introducing breaking changes, bump the version and declare the previous version:

```yaml
definitionSet:
  name: customer-model
  version: 2.0.0
  previousVersion: 1.5.0
  package: com.example.customer.generated
  migration:
    generateBridgeMapper: true
    bridgePackage: com.example.customer.migration
```

TwinMapper generates bridge mappers between `CustomerDtoV1` and `CustomerDtoV2` when `generateBridgeMapper: true` is set.

---

## Migrating from MapStruct

MapStruct is annotation-driven and generates compile-time mapper implementations from annotated interfaces. TwinMapper replaces MapStruct's object-mapping capability with a YAML-driven approach.

### Before (MapStruct)

```java
@Mapper(componentModel = "spring")
public interface CustomerMapper {

    @Mapping(source = "customerCode", target = "code")
    @Mapping(source = "statusCode", target = "status", qualifiedByName = "codeToStatus")
    Customer entityToDomain(CustomerEntity entity);

    @Named("codeToStatus")
    default CustomerStatus codeToStatus(String code) {
        return CustomerStatus.fromCode(code);
    }
}
```

### After (TwinMapper)

```yaml
# src/main/twinmapper/mappings/customer-mappings.yaml
objectMappings:
  - name: CustomerEntityToDomain
    sourceType: com.example.persistence.CustomerEntity
    targetType: com.example.domain.Customer
    mode: CREATE
    fields:
      - source: customerCode
        target: code
      - source: statusCode
        target: status
        converter: statusCodeToEnum
```

```java
// Converter registration replaces @Named methods
@Configuration
public class CustomerConverters implements TwinMapperRuntimeConfigurer {
    @Override
    public void addConverters(ConverterRegistry registry) {
        registry.addConverter(String.class, CustomerStatus.class,
            CustomerStatus::fromCode);
    }
}
```

### Key differences

| MapStruct | TwinMapper |
|---|---|
| Annotation-driven | YAML-driven (annotations optional) |
| Mapper interfaces handwritten | Mapper classes fully generated |
| Custom methods in mapper interface | Custom converters registered separately |
| Single annotation processor | Build plugin + runtime |
| No document binding | Full YAML/JSON/BPMN document binding |

---

## Migrating from ModelMapper

ModelMapper uses runtime reflection and convention-based mapping. TwinMapper replaces this with compile-time generated mappers.

### Before (ModelMapper)

```java
ModelMapper mapper = new ModelMapper();
mapper.getConfiguration()
    .setMatchingStrategy(MatchingStrategies.STRICT);

CustomerDto dto = mapper.map(customer, CustomerDto.class);
```

### After (TwinMapper)

1. Define the mapping in the YAML DSL.
2. Let TwinMapper generate the mapper.
3. Inject and use the registry.

```java
@Autowired
TwinObjectMapperRegistry registry;

TwinObjectMapper<Customer, CustomerDto> mapper =
    registry.mapperFor(Customer.class, CustomerDto.class);

CustomerDto dto = mapper.map(customer);
```

### Key differences

| ModelMapper | TwinMapper |
|---|---|
| Runtime reflection and convention matching | Compile-time generated, no reflection |
| Implicit field discovery | Explicit field declarations |
| Configuration via Java API | Configuration via YAML DSL |
| Non-deterministic edge cases | Deterministic — same inputs, same output |
| No document binding | Full document binding |

---

## Migrating from Jackson ObjectMapper for DTO Binding

If you are currently using Jackson's `ObjectMapper` directly to bind YAML/JSON into DTOs, TwinMapper replaces this with generated binders that enforce schema rules.

### Before (Jackson direct binding)

```java
ObjectMapper mapper = new ObjectMapper(new YAMLFactory());
CustomerDto customer = mapper.readValue(yamlInput, CustomerDto.class);
// No schema enforcement, unknown fields silently accepted
```

### After (TwinMapper binding)

```java
CustomerDto customer = runtime.readYaml(yamlInput, CustomerDto.class);
// Schema rules enforced: required fields, enum validation, unknown field rejection
```

### Key differences

| Jackson direct | TwinMapper |
|---|---|
| No schema enforcement | Full schema enforcement via generated validators |
| Unknown fields silently accepted | Unknown fields rejected in STRICT mode |
| No alias support | Alias resolution built-in |
| No default application | Defaults applied at bind time |
| No forbidden field detection | Forbidden fields always rejected |

---

## Migrating Existing YAML/JSON Documents

When updating definition schemas, existing documents may require updating. Use the TwinMapper CLI to assess impact.

```bash
# Check which existing documents would fail against a new schema version
twinmapper validate \
  --schema src/main/twinmapper/schemas/customer-model.yaml \
  --documents src/test/resources/fixtures/yaml/ \
  --report validation-report.txt
```

### Common migration scenarios

**Scenario: field renamed**

1. Add old name as alias in the new definition.
2. Run validation against existing documents — all should pass.
3. Schedule a migration window to update documents to the canonical name.
4. After migration, move old name to `deprecatedAliases`.
5. After full adoption, move to `forbiddenFields` with a migration hint.

**Scenario: field made required**

1. Run validation in COMPATIBLE mode against existing documents to find all documents missing the new required field.
2. Provide a default value to cover documents that legitimately omit the field, or update all documents.
3. After all documents are updated, set `required: true` with no default.

**Scenario: enum value removed**

1. Add the removed value to `forbiddenFields` equivalent for enums (or mark as deprecated).
2. Update all documents to use valid values.
3. Remove the value from the definition.

---

## Bridge Mapper Usage

When `generateBridgeMapper: true` is set, TwinMapper generates a bridge mapper between the previous version's DTO and the current version's DTO.

```java
// Generated bridge
public final class CustomerDtoV1ToV2BridgeMapper
        implements TwinObjectMapper<CustomerDtoV1, CustomerDtoV2> {

    @Override
    public CustomerDtoV2 map(CustomerDtoV1 source) {
        return new CustomerDtoV2(
            source.id(),
            source.customerCode(),  // renamed to code in V2
            CustomerStatus.fromCode(source.rawStatus()),  // type changed in V2
            null  // new field in V2, no V1 equivalent
        );
    }
}
```

Bridge mappers are available in `{bridgePackage}` and can be injected like any other generated mapper.

---

## CLI Migration Tools

The TwinMapper CLI provides migration assistance tools.

```bash
# Validate documents against current schema — reports all errors and warnings
twinmapper validate --schema schemas/ --documents documents/

# Diff two schema versions — shows breaking and non-breaking changes
twinmapper schema-diff \
  --from-version 1.5.0 \
  --to-version 2.0.0 \
  --schema customer-model.yaml

# Sample-to-definition assistant — proposes a draft definition from sample documents
# Always requires human review before use
twinmapper suggest-schema \
  --samples samples/customer-samples/ \
  --output draft-customer-model.yaml
```

All CLI migration tooling is developer tooling only. It does not run as part of the application at runtime.
