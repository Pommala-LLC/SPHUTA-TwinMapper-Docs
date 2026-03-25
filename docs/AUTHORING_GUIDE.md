# TwinMapper — Authoring Guide

## Overview

This guide covers how to write TwinMapper definition files and object-mapping DSL files. All definitions are written in TwinMapper's native YAML DSL. The DSL is purpose-built for code generation and object mapping — it is not JSON Schema.

Definition files live in `src/main/twinmapper/schemas/` by default. Mapping files live in `src/main/twinmapper/mappings/`. Both directories are scanned by the build plugin.

---

## Definition File Structure

Every definition file begins with a `definitionSet` block that declares identity and generation options, followed by a `types` block containing object and enum type declarations.

```yaml
definitionSet:
  name: order-model
  version: 1.0.0
  package: com.example.order.generated
  strictness: STRICT

types:
  Order:
    kind: object
    fields:
      # field declarations

  OrderStatus:
    kind: enum
    values:
      # enum value declarations
```

---

## definitionSet Block

| Field | Required | Description |
|---|---|---|
| `name` | Yes | Unique identifier for this definition set |
| `version` | Yes | Semantic version string |
| `package` | Yes | Java package for all generated types in this set |
| `strictness` | No | Overrides global strictness for this definition set. STRICT \| COMPATIBLE \| LENIENT |

---

## Object Type Definition

Declares a DTO or domain object that TwinMapper will generate a Java class or record for.

```yaml
types:
  Customer:
    kind: object
    description: Represents a customer in the order management domain
    fields:
      id:
        type: string
        required: true
        description: Unique customer identifier

      name:
        type: string
        required: true
        targetName: fullName
        description: Customer full name

      status:
        type: CustomerStatus
        required: true

      address:
        type: Address
        required: false

      tags:
        type: list<string>
        required: false
        default: []

      metadata:
        type: map<string, string>
        required: false
```

---

## Field Definition

Each field inside an object type supports the following properties.

| Property | Required | Description |
|---|---|---|
| `type` | Yes | Java type, reference to another declared type, `list<T>`, or `map<K,V>` |
| `required` | No | `true` or `false`. Default is `false` |
| `default` | No | Default value applied at bind time when field is absent |
| `targetName` | No | Override the generated Java field name. Source name is preserved in metadata |
| `aliases` | No | List of alternative source field names resolved to this field at bind time |
| `deprecatedAliases` | No | List of deprecated source names that produce a warning or error depending on strictness |
| `description` | No | Human-readable field description emitted into generated Javadoc |
| `constraints` | No | Block of constraint declarations |

---

## Primitive Types

| DSL Type | Generated Java Type |
|---|---|
| `string` | `String` |
| `integer` | `Integer` / `int` |
| `long` | `Long` / `long` |
| `double` | `Double` / `double` |
| `boolean` | `Boolean` / `boolean` |
| `decimal` | `java.math.BigDecimal` |
| `date` | `java.time.LocalDate` |
| `datetime` | `java.time.LocalDateTime` |
| `instant` | `java.time.Instant` |
| `uuid` | `java.util.UUID` |
| `list<T>` | `java.util.List<T>` |
| `map<K,V>` | `java.util.Map<K,V>` |

---

## Enum Type Definition

```yaml
types:
  OrderStatus:
    kind: enum
    description: Lifecycle status of an order
    values:
      - name: PENDING
        code: pending
        description: Order received but not yet confirmed

      - name: CONFIRMED
        code: confirmed
        description: Order confirmed and being prepared

      - name: SHIPPED
        code: shipped
        description: Order dispatched to carrier

      - name: DELIVERED
        code: delivered
        description: Order received by customer

      - name: CANCELLED
        code: cancelled
        description: Order cancelled before fulfilment
        deprecated: true
        deprecationMessage: Use REFUNDED instead
```

| Property | Required | Description |
|---|---|---|
| `name` | Yes | Java enum constant name |
| `code` | No | String code used in source documents and for code/enum conversion |
| `description` | No | Emitted into generated Javadoc |
| `deprecated` | No | Marks this value as deprecated. Warning in COMPATIBLE, error in STRICT |
| `deprecationMessage` | No | Message included in deprecation warning |

---

## Constraints

Constraints are declared inside a field's `constraints` block.

```yaml
fields:
  email:
    type: string
    required: true
    constraints:
      pattern: "^[^@]+@[^@]+\\.[^@]+$"
      maxLength: 255

  quantity:
    type: integer
    required: true
    constraints:
      min: 1
      max: 9999

  name:
    type: string
    required: true
    constraints:
      minLength: 1
      maxLength: 100
```

### One-Of Constraint

Exactly one field from the declared group must be present.

```yaml
types:
  ContactInfo:
    kind: object
    constraints:
      oneOf:
        - [email, phone]
    fields:
      email:
        type: string
        required: false
      phone:
        type: string
        required: false
```

### Mutually Exclusive Constraint

The listed fields must not both be present.

```yaml
constraints:
  mutuallyExclusive:
    - [legacyId, newId]
```

### Conditional Required Constraint

A field becomes required when a discriminator field equals a specific value.

```yaml
fields:
  triggerType:
    type: TriggerType
    required: true

  deliveryClass:
    type: DeliveryClass
    required: false
    constraints:
      requiredWhen:
        field: triggerType
        values: [MESSAGE, SIGNAL, UPDATE]
```

---

## Aliases

Aliases allow a source document field with a different name to map to this field. Useful for supporting multiple naming conventions or renaming fields while maintaining backward compatibility.

```yaml
fields:
  createdAt:
    type: datetime
    required: true
    aliases:
      - created_at
      - creation_date
      - createdDate
```

---

## Deprecated Aliases

Deprecated aliases produce a warning in COMPATIBLE mode and an error in STRICT mode.

```yaml
fields:
  customerId:
    type: string
    required: true
    aliases:
      - customer_id
    deprecatedAliases:
      - cust_id
      - custId
```

---

## Forbidden Fields

Forbidden fields produce a binding error when present in the source document, regardless of strictness mode.

```yaml
types:
  WorkflowDefinition:
    kind: object
    forbiddenFields:
      - interrupting        # Replaced by interrupting-mode
      - legacyTimerRef      # Replaced by timer.ref
    fields:
      interruptingMode:
        type: InterruptingMode
        required: false
        default: INTERRUPTING
```

---

## Versioning

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

---

## Object Mapping DSL

Mapping definitions live in `src/main/twinmapper/mappings/` as separate YAML files.

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

      - source: statusCode
        target: status
        converter: statusCodeToEnum

      - source: city
        target: address.city

      - source: zip
        target: address.zip

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

      - source: address.zip
        target: zip
```

---

## Mapping Mode Reference

| Mode | Description |
|---|---|
| `CREATE` | Builds a new target object. Default null policy: `SET_NULLS` |
| `UPDATE` | Applies source values onto an existing target object. Default null policy: `IGNORE_NULLS` |
| `PATCH` | Applies only non-null source values onto an existing target object. Null policy: always `IGNORE_NULLS` |

---

## Null Policy Reference

| Policy | Behavior |
|---|---|
| `IGNORE_NULLS` | Null source field is skipped. Target retains its current value or constructor default |
| `SET_NULLS` | Null source field sets the target field to null |
| `FAIL_ON_NULL` | Null source field on a required mapping throws a `MappingError` |

---

## Reusable Profiles

Profiles declare shared mapping defaults that can be applied across multiple mappings.

```yaml
profiles:
  - name: entityUpdateProfile
    nullPolicy: IGNORE_NULLS
    unmappedTargetPolicy: WARN
    ignoreTargets:
      - createdAt
      - createdBy
      - version

  - name: patchProfile
    nullPolicy: IGNORE_NULLS
    unmappedTargetPolicy: IGNORE

objectMappings:
  - name: CustomerPatchMapping
    sourceType: com.example.dto.CustomerPatchDto
    targetType: com.example.persistence.CustomerEntity
    mode: PATCH
    profile: patchProfile
    fields:
      - source: name
        target: customerName
```

---

## Converters

```yaml
converters:
  - name: longToCustomerId
    sourceType: java.lang.Long
    targetType: com.example.domain.CustomerId

  - name: statusCodeToEnum
    sourceType: java.lang.String
    targetType: com.example.domain.CustomerStatus

  - name: moneyConverter
    sourceType: java.math.BigDecimal
    targetType: com.example.domain.Money
```

Converter implementations are provided as Java classes registered via `TwinMapperRuntimeConfigurer` or as Spring beans implementing `Converter<S, T>`.

---

## Generation Options

Generation options can be declared per definition set to override global defaults.

```yaml
definitionSet:
  name: legacy-model
  version: 1.0.0
  package: com.example.legacy.generated
  generation:
    immutableDtos: false        # Generate mutable classes for legacy JPA compatibility
    springBeanMode: true        # Emit @Component on generated validators and mappers
    generateBinders: true
    generateValidators: true
    generateMappers: false      # Skip mapper generation for this set
    generateMetadata: true
```

---

## File Organization Recommendations

```
src/main/twinmapper/
├── schemas/
│   ├── customer-model.yaml
│   ├── order-model.yaml
│   └── product-model.yaml
├── mappings/
│   ├── customer-mappings.yaml
│   ├── order-mappings.yaml
│   └── product-mappings.yaml
└── converters/
    └── custom-converters.yaml
```

Keep one definition set per file. Keep mapping definitions in separate files from schema definitions. Group by domain area rather than by type.
