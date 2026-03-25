# TwinMapper — Code Generation Contract

## Overview

This document defines the exact contract between TwinMapper's code generation engine and the consuming application. It covers what is generated, the naming rules, the generated class shapes, and the guarantees the generated code must uphold.

All generated code is written to `build/generated/sources/twinmapper` and must not be committed to version control. Generated code is recomputed on every build.

---

## Generation Guarantees

- Generated code is always compilable given a valid definition set.
- Generated code contains no reflection at runtime.
- Generated code uses plain Java method calls for all field access.
- Generated code is Lombok-free by default.
- Generated code is deterministic — identical inputs always produce identical outputs.
- Generated code does not depend on any TwinMapper runtime module at the class level for DTOs and enums. Only binders, validators, and mappers depend on the runtime.

---

## Generated Artifact Types

| Artifact | Description | Naming Pattern |
|---|---|---|
| DTO class | Immutable record or mutable class for each object type | `{TypeName}Dto` |
| Enum class | Java enum for each enum type | `{TypeName}` (as declared) |
| Metadata descriptor | Field registry and descriptor for each object type | `{TypeName}Metadata` |
| Document binder | Binder for reading runtime documents into a DTO | `{TypeName}Binder` |
| Schema validator | Constraint validator for a DTO | `{TypeName}Validator` |
| Create mapper | Creates a new target from a source | `{MappingName}Mapper` |
| Update mapper | Applies source fields onto an existing target | `{MappingName}UpdateMapper` |
| Patch mapper | Applies non-null source fields onto existing target | `{MappingName}PatchMapper` |
| Inverse mapper | Reverse direction of a declared mapping | `{MappingName}InverseMapper` |
| Binder registry | Registry for looking up binders by target type | `TwinMapperBinderRegistry` |
| Mapper registry | Registry for looking up mappers by source and target type | `TwinMapperObjectMapperRegistry` |

---

## DTO Generation

### Immutable Record (Default)

```java
// Source definition
// Customer:
//   kind: object
//   fields:
//     id: { type: string, required: true }
//     name: { type: string, required: true, targetName: fullName }
//     status: { type: CustomerStatus, required: true }
//     address: { type: Address, required: false }

public record CustomerDto(
    String id,
    String fullName,
    CustomerStatus status,
    @Nullable AddressDto address
) {}
```

### Mutable Class (when `immutableDtos: false`)

```java
public class CustomerDto {

    private String id;
    private String fullName;
    private CustomerStatus status;
    private AddressDto address;

    // Generated constructor
    public CustomerDto(String id, String fullName, CustomerStatus status, AddressDto address) {
        this.id = id;
        this.fullName = fullName;
        this.status = status;
        this.address = address;
    }

    // Generated getters and setters
    public String getId() { return id; }
    public void setId(String id) { this.id = id; }
    // ...
}
```

### Spring Bean Mode

When `springBeanMode: true` is enabled in generation options, validators and mappers receive `@Component`:

```java
@Component
public final class CustomerDtoValidator implements Validator {
    // ...
}

@Component
public final class CustomerEntityToDomainMapper
        implements TwinObjectMapper<CustomerEntity, Customer> {
    // ...
}
```

---

## Enum Generation

```java
// Source definition
// OrderStatus:
//   kind: enum
//   values:
//     - name: PENDING, code: pending
//     - name: CONFIRMED, code: confirmed
//     - name: CANCELLED, code: cancelled, deprecated: true

public enum OrderStatus {

    PENDING("pending"),
    CONFIRMED("confirmed"),
    /** @deprecated Use REFUNDED instead */
    @Deprecated
    CANCELLED("cancelled");

    private final String code;

    OrderStatus(String code) {
        this.code = code;
    }

    public String getCode() {
        return code;
    }

    public static OrderStatus fromCode(String code) {
        for (OrderStatus value : values()) {
            if (value.code.equals(code)) return value;
        }
        throw new InvalidEnumCodeException(OrderStatus.class, code);
    }
}
```

---

## Metadata Descriptor Generation

```java
public final class CustomerDtoMetadata {

    public static final String FIELD_ID = "id";
    public static final String FIELD_FULL_NAME = "fullName";
    public static final String FIELD_STATUS = "status";
    public static final String FIELD_ADDRESS = "address";

    public static final String SOURCE_FIELD_ID = "id";
    public static final String SOURCE_FIELD_FULL_NAME = "name";       // original source name
    public static final String SOURCE_FIELD_STATUS = "status";
    public static final String SOURCE_FIELD_ADDRESS = "address";

    public static final List<String> ALIASES_FULL_NAME =
        List.of("name", "fullName", "full_name");

    public static final Set<String> REQUIRED_FIELDS =
        Set.of("id", "name", "status");

    public static final Map<String, Object> DEFAULTS = Map.of();

    public static final Set<String> FORBIDDEN_FIELDS = Set.of();

    public static final String DEFINITION_SET_NAME = "customer-model";
    public static final String DEFINITION_SET_VERSION = "1.0.0";
}
```

---

## Document Binder Generation

```java
public final class CustomerDtoBinder implements TwinBinder<CustomerDto> {

    @Override
    public CustomerDto bind(NodeCursor cursor, BindingContext context) {

        String id = cursor.requiredString("id");

        String fullName = cursor.requiredString(
            CustomerDtoMetadata.SOURCE_FIELD_FULL_NAME,
            CustomerDtoMetadata.ALIASES_FULL_NAME
        );

        CustomerStatus status = cursor.requiredEnum("status", CustomerStatus.class);

        AddressDto address = cursor.optionalChild("address")
            .map(child -> context.bind(child, AddressDto.class))
            .orElse(null);

        return new CustomerDto(id, fullName, status, address);
    }
}
```

---

## Validator Generation

```java
public final class CustomerDtoValidator implements SmartValidator {

    @Override
    public boolean supports(Class<?> clazz) {
        return CustomerDto.class.isAssignableFrom(clazz);
    }

    @Override
    public void validate(Object target, Errors errors) {
        CustomerDto dto = (CustomerDto) target;

        // Required field checks
        if (dto.id() == null || dto.id().isBlank()) {
            errors.rejectValue("id", "field.required", "Field 'id' is required");
        }

        if (dto.fullName() == null || dto.fullName().isBlank()) {
            errors.rejectValue("fullName", "field.required", "Field 'fullName' is required");
        }

        // Enum check handled at bind time; re-validated here if needed
        if (dto.status() == null) {
            errors.rejectValue("status", "field.required", "Field 'status' is required");
        }
    }

    @Override
    public void validate(Object target, Errors errors, Object... validationHints) {
        validate(target, errors);
    }
}
```

---

## Object Mapper Generation

### Create Mapper

```java
public final class CustomerEntityToDomainMapper
        implements TwinObjectMapper<CustomerEntity, Customer> {

    private final TwinObjectMapper<AddressEntity, Address> addressMapper;

    public CustomerEntityToDomainMapper(
            TwinObjectMapper<AddressEntity, Address> addressMapper) {
        this.addressMapper = addressMapper;
    }

    @Override
    public Customer map(CustomerEntity source) {
        if (source == null) return null;

        return new Customer(
            source.getId() != null ? new CustomerId(source.getId()) : null,
            source.getCustomerCode(),
            CustomerStatus.fromCode(source.getStatusCode()),
            source.getCity() != null
                ? new Address(source.getCity(), source.getZip())
                : null
        );
    }
}
```

### Update Mapper

```java
public final class CustomerDomainToEntityUpdateMapper
        implements TwinUpdateMapper<Customer, CustomerEntity> {

    @Override
    public CustomerEntity update(Customer source, CustomerEntity target) {
        if (source == null) return target;

        if (source.id() != null) {
            target.setId(source.id().value());
        }
        if (source.code() != null) {
            target.setCustomerCode(source.code());
        }
        if (source.status() != null) {
            target.setStatusCode(source.status().getCode());
        }
        if (source.address() != null) {
            if (source.address().city() != null) target.setCity(source.address().city());
            if (source.address().zip() != null) target.setZip(source.address().zip());
        }
        return target;
    }
}
```

### Patch Mapper

```java
public final class CustomerPatchDtoToEntityMapper
        implements TwinPatchMapper<CustomerPatchDto, CustomerEntity> {

    @Override
    public CustomerEntity patch(CustomerPatchDto source, CustomerEntity target) {
        if (source == null) return target;

        // Only non-null source values are applied
        if (source.name() != null) {
            target.setCustomerCode(source.name());
        }
        if (source.statusCode() != null) {
            target.setStatusCode(source.statusCode());
        }
        return target;
    }
}
```

---

## Registry Generation

### Binder Registry

```java
public final class GeneratedBinderRegistryImpl implements GeneratedBinderRegistry {

    private static final Map<Class<?>, TwinBinder<?>> BINDERS = Map.of(
        CustomerDto.class, new CustomerDtoBinder(),
        AddressDto.class, new AddressDtoBinder(),
        OrderDto.class, new OrderDtoBinder()
    );

    @Override
    @SuppressWarnings("unchecked")
    public <T> TwinBinder<T> binderFor(Class<T> targetType) {
        TwinBinder<?> binder = BINDERS.get(targetType);
        if (binder == null) {
            throw new NoBinderFoundException(targetType);
        }
        return (TwinBinder<T>) binder;
    }
}
```

---

## Naming Conventions

| Element | Convention | Example |
|---|---|---|
| DTO class | `{TypeName}Dto` | `CustomerDto` |
| Enum class | `{TypeName}` as declared | `OrderStatus` |
| Binder class | `{TypeName}Binder` | `CustomerDtoBinder` |
| Validator class | `{TypeName}Validator` | `CustomerDtoValidator` |
| Metadata class | `{TypeName}Metadata` | `CustomerDtoMetadata` |
| Create mapper | `{MappingName}Mapper` | `CustomerEntityToDomainMapper` |
| Update mapper | `{MappingName}UpdateMapper` | `CustomerDomainToEntityUpdateMapper` |
| Patch mapper | `{MappingName}PatchMapper` | `CustomerPatchDtoToEntityMapper` |
| Inverse mapper | `{MappingName}InverseMapper` | `CustomerEntityToDomainInverseMapper` |

---

## Package Layout

All generated types are placed under the configured `basePackage`. Sub-packages follow this structure:

```
{basePackage}/
├── dto/           # Generated DTO classes
├── enums/         # Generated enum classes
├── binders/       # Generated binder classes
├── validators/    # Generated validator classes
├── mappers/       # Generated mapper classes
├── metadata/      # Generated metadata descriptor classes
└── registry/      # Generated registry classes
```

---

## Immutability Contract

Generated DTOs are immutable Java records by default. The following contract is upheld:

- No setters on records
- Constructor validation via `Objects.requireNonNull` on required fields
- No mutable collections — `List.copyOf()` and `Map.copyOf()` used for collection fields
- `equals()`, `hashCode()`, and `toString()` provided by the Java record mechanism

---

## Null Contract

- Required fields — `@NonNull` annotated, constructor throws `NullPointerException` if null
- Optional fields — `@Nullable` annotated, null is a valid value
- Collection fields with default `[]` — generated as empty immutable list, never null
- Nested optional objects — `@Nullable`, null is valid
