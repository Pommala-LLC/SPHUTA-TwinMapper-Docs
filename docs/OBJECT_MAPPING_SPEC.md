# TwinMapper — Object Mapping Specification

## Overview

The Object Engine generates and executes typed Java-to-Java mappers for layer-crossing flows. This document specifies the full object mapping model, all supported mapping flows, mapper types, field mapping rules, null policies, conversion, profile behavior, and the inverse mapping contract.

---

## Supported Mapping Flows

| Source | Target | Typical Use Case |
|---|---|---|
| DTO | DTO | API response reshaping, view model construction |
| Entity | Domain | Load aggregate from persistence |
| Domain | Entity | Persist aggregate |
| Domain | DTO | API response serialization |
| DTO | Domain | Command construction from request |
| Entity | DTO | Read-model / projection query shortcut |
| DTO | Entity | Direct persistence write from API |
| Entity | Projection | Read-model construction from persistence entity |
| Projection | DTO | Read-model API serialization |
| Request/Command | Domain | Domain command construction from HTTP request |
| Domain | Response/View/Event | CQRS projection, event payload, or API response |
| Patch DTO | Entity | Partial update to persistence entity |
| Patch DTO | Domain | Partial update to domain aggregate |

---

## Mapper Types

### TwinObjectMapper — CREATE mode

Builds a new target object from a source object.

```java
public interface TwinObjectMapper<S, T> {
    T map(S source);
}
```

- Source null returns null.
- Default null policy: `SET_NULLS` — null source fields set the corresponding target field to null.

---

### TwinUpdateMapper — UPDATE mode

Applies source field values onto an existing target object.

```java
public interface TwinUpdateMapper<S, T> {
    T update(S source, T target);
}
```

- Source null returns target unchanged.
- Default null policy: `IGNORE_NULLS` — null source fields leave the target field unchanged.
- Identity and audit fields (createdAt, createdBy, version) are preserved unless explicitly mapped.

---

### TwinPatchMapper — PATCH mode

Applies only non-null source field values onto an existing target object.

```java
public interface TwinPatchMapper<S, T> {
    T patch(S source, T target);
}
```

- Source null returns target unchanged.
- Null policy is always `IGNORE_NULLS` — this is not configurable for PATCH mode.
- Only fields present and non-null in the source are applied.

---

### TwinInverseMapper

A generated mapper in the reverse direction of a declared mapping.

```java
public interface TwinObjectMapper<T, S> { // reversed type parameters
    S map(T source);
}
```

Declared via `inverseOf` in the mapping DSL. Build-time failure if the inverse is ambiguous or unsafe (e.g., flattening with no reverse expansion, or value object conversion with no declared reverse converter).

---

## Field Mapping Rules

### Explicit field mapping

```yaml
fields:
  - source: customerCode
    target: code
```

Direct source-to-target field assignment. Source path and target path can be simple field names or nested paths.

### Nested path mapping (flattening)

```yaml
fields:
  - source: address.city
    target: city
  - source: address.postcode
    target: postalCode
```

Maps a nested source field to a flat target field. Null safety: if `address` is null, `city` and `postalCode` on the target are set according to the null policy.

### Nested path mapping (expansion)

```yaml
fields:
  - source: city
    target: address.city
  - source: postalCode
    target: address.postcode
```

Maps flat source fields into a nested target object. The nested target object is created if any of its source fields are non-null.

### Converter-assisted mapping

```yaml
fields:
  - source: statusCode
    target: status
    converter: statusCodeToEnum
```

The named converter is applied to transform the source field value before assignment to the target field.

### Source path expressions

Dotted paths for nested access are supported:

```yaml
fields:
  - source: id.value          # accesses CustomerId.value()
    target: id
  - source: address.city      # accesses Address.city()
    target: city
```

---

## Null Policies

### IGNORE_NULLS

Null source fields are skipped. Target retains its current value or constructor default. Default for UPDATE and PATCH modes.

### SET_NULLS

Null source fields set the corresponding target field to null. Default for CREATE mode.

### FAIL_ON_NULL

Null source field on a mapped field throws `MappingError` with the field path. Used for required field enforcement in strict mapping scenarios.

### Policy declaration

Null policy can be declared at three levels. More specific declarations override less specific ones.

```yaml
# Definition-level default
objectMappings:
  - name: CustomerUpdate
    mode: UPDATE
    nullPolicy: IGNORE_NULLS     # applies to all fields in this mapping

    fields:
      - source: code
        target: customerCode
        # inherits IGNORE_NULLS

      - source: email
        target: emailAddress
        nullPolicy: FAIL_ON_NULL  # overrides for this field
```

Via reusable profile:

```yaml
profiles:
  - name: strictUpdate
    nullPolicy: FAIL_ON_NULL

objectMappings:
  - name: OrderUpdate
    profile: strictUpdate
    # inherits FAIL_ON_NULL from profile
```

---

## Reusable Profiles

Profiles declare shared mapping defaults. Multiple mappings can reference the same profile.

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

  - name: strictCreate
    nullPolicy: FAIL_ON_NULL
    unmappedTargetPolicy: ERROR
```

### Profile fields

| Field | Description |
|---|---|
| `nullPolicy` | Default null policy for all fields in mappings using this profile |
| `unmappedTargetPolicy` | Behavior when a target field has no source mapping: WARN, ERROR, or IGNORE |
| `ignoreTargets` | List of target field names to exclude from all mappings using this profile |
| `converters` | List of converter names available to all field mappings using this profile |

---

## Converters

Converters transform a source field value into a compatible target type before assignment.

### Converter declaration in DSL

```yaml
converters:
  - name: longToCustomerId
    sourceType: java.lang.Long
    targetType: com.example.domain.CustomerId

  - name: statusCodeToEnum
    sourceType: java.lang.String
    targetType: com.example.domain.CustomerStatus
```

### Converter implementation — Spring style

```java
@Component
public class CustomerIdConverter implements Converter<Long, CustomerId> {
    @Override
    public CustomerId convert(Long source) {
        return source != null ? new CustomerId(source) : null;
    }
}
```

### Converter implementation — programmatic registration

```java
@Configuration
public class ConverterConfig implements TwinMapperRuntimeConfigurer {
    @Override
    public void addConverters(ConverterRegistry registry) {
        registry.addConverter(Long.class, CustomerId.class,
            id -> id != null ? new CustomerId(id) : null);
    }
}
```

### ConditionalGenericConverter for complex type matching

For converters that must inspect generic type parameters before deciding whether to convert:

```java
@Component
public class ListItemConverter implements ConditionalGenericConverter {

    @Override
    public Set<ConvertiblePair> getConvertibleTypes() {
        return Set.of(new ConvertiblePair(List.class, List.class));
    }

    @Override
    public boolean matches(TypeDescriptor sourceType, TypeDescriptor targetType) {
        return sourceType.getElementTypeDescriptor() != null
            && targetType.getElementTypeDescriptor() != null;
    }

    @Override
    public Object convert(Object source, TypeDescriptor sourceType, TypeDescriptor targetType) {
        // Element-level conversion logic
    }
}
```

---

## BeanWrapper Usage (UPDATE and PATCH)

UPDATE and PATCH mappers use `BeanWrapper` for target property access in Spring applications. This handles:

- Nested property path resolution: `target.setAddress(...)` resolves `address.city` paths
- JPA-loaded proxy transparency: operates on the real entity behind Hibernate proxies
- Type conversion via the `ConversionService` backend

```java
// Internal generated update mapper logic
BeanWrapper wrapper = PropertyAccessorFactory.forBeanPropertyAccess(target);
wrapper.setConversionService(conversionService);
if (source.code() != null) {
    wrapper.setPropertyValue("customerCode", source.code());
}
```

---

## DirectFieldAccessor Usage

For target objects without setter methods — value objects with final fields — `DirectFieldAccessor` is used instead of `BeanWrapper`.

```java
DirectFieldAccessor accessor = new DirectFieldAccessor(target);
if (source.code() != null) {
    accessor.setPropertyValue("code", source.code());
}
```

---

## Proxy-Safe Type Inspection

All reflection-based operations use proxy-safe type resolution. This is mandatory — not optional.

```java
// Correct — unwraps Spring proxy before type inspection
Class<?> targetClass = AopUtils.getTargetClass(target);

// Correct — for ultimate target in nested proxy chains
Class<?> ultimateClass = AopProxyUtils.ultimateTargetClass(target);

// Incorrect — never use this on Spring-managed beans
Class<?> wrongClass = target.getClass(); // may return proxy class
```

---

## Inverse Mapping

### Declaration

```yaml
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

  - name: CustomerDomainToEntity
    sourceType: com.example.domain.Customer
    targetType: com.example.persistence.CustomerEntity
    mode: UPDATE
    inverseOf: CustomerEntityToDomain
```

### Inverse mapping safety rules

TwinMapper generates the inverse mapper only if the reverse is unambiguous and safe. The following cause a build-time `INVERSE_MAPPING_UNSAFE` error:

- Flattening without declared reverse expansion
- Value object conversion without a declared reverse converter
- One-to-many or many-to-one field relationships
- Computed or derived target fields with no declared source equivalent

For unsafe inversions, write the mapping explicitly rather than using `inverseOf`.

---

## Collection Mapping

Collections are mapped element-by-element using the element mapper for the declared types.

```yaml
fields:
  - source: lineItems
    target: items
    # element mapper List<LineItemEntity> → List<LineItem> resolved automatically
```

Supported collection types:

| Source | Target |
|---|---|
| `List<S>` | `List<T>` |
| `Set<S>` | `Set<T>` |
| `List<S>` | `Set<T>` |
| `Set<S>` | `List<T>` |
| `Map<K1,V1>` | `Map<K2,V2>` |

Null source collections: behavior follows the declared null policy. `IGNORE_NULLS` leaves the target collection unchanged. `SET_NULLS` sets the target collection to null.

---

## Registry Lookup

```java
// Inject the registry
@Autowired
TwinObjectMapperRegistry mapperRegistry;

// Create mapping
TwinObjectMapper<CustomerEntity, Customer> mapper =
    mapperRegistry.mapperFor(CustomerEntity.class, Customer.class);

// Update mapping
TwinUpdateMapper<Customer, CustomerEntity> updateMapper =
    mapperRegistry.updateMapperFor(Customer.class, CustomerEntity.class);

// Patch mapping
TwinPatchMapper<CustomerPatchDto, CustomerEntity> patchMapper =
    mapperRegistry.patchMapperFor(CustomerPatchDto.class, CustomerEntity.class);
```

Or via `TwinMapperRuntime`:

```java
Customer domain = runtime.map(entity, Customer.class);
CustomerEntity updated = runtime.update(domain, existingEntity);
CustomerEntity patched = runtime.patch(patchDto, existingEntity);
```
