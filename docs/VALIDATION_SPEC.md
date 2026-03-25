# TwinMapper — Validation Specification

## Overview

TwinMapper validation operates in two distinct layers. The first layer is generated schema validation, which is part of the TwinMapper platform. The second layer is semantic/business validation, which is consumer-provided and lives outside TwinMapper core.

---

## Two Validation Layers

| Layer | Owner | Mechanism | Scope |
|---|---|---|---|
| Schema validation | TwinMapper (generated) | Generated validator classes | Structural and constraint rules declared in the DSL |
| Semantic validation | Consumer application | `ValidationExtension` SPI or handwritten validators | Business rules, cross-object domain logic, execution semantics |

TwinMapper does not own domain-specific semantics. Schema validation catches what the definition DSL can express. Everything else belongs in consumer-provided semantic validators.

---

## Spring Validation Integration

Generated validators implement Spring's `SmartValidator` interface, making them compatible with all Spring validation integration points.

```java
// Generated
public final class OrderDtoValidator implements SmartValidator {

    @Override
    public boolean supports(Class<?> clazz) {
        return OrderDto.class.isAssignableFrom(clazz);
    }

    @Override
    public void validate(Object target, Errors errors) {
        // Generated constraint checks
    }

    @Override
    public void validate(Object target, Errors errors, Object... validationHints) {
        validate(target, errors);
    }
}
```

### Integration points

| Spring Component | How generated validators integrate |
|---|---|
| `WebDataBinder` | Register via `DataBinder.addValidators(orderDtoValidator)` |
| `LocalValidatorFactoryBean` | Wrap via `SpringValidatorAdapter` for JSR-380 compatibility |
| `MethodValidationPostProcessor` | Use with `@Validated` on controller classes |
| `BindingResult` | Errors from generated validators accumulate into `BindingResult` |
| `HandlerMethodArgumentResolver` | Triggered automatically when `@Valid` is present on method parameters |

---

## Generated Constraint Types

### Required field check

```yaml
fields:
  id:
    type: string
    required: true
```

Generated check:

```java
if (dto.id() == null || dto.id().isBlank()) {
    errors.rejectValue("id", "MISSING_REQUIRED_FIELD",
        "Field 'id' is required");
}
```

---

### Default value application

Applied at bind time, not at validation time. If a field with a declared default is absent from the input, the default is applied by the generated binder before the validator runs. The validator sees the default value as present.

---

### Enum validation

```yaml
fields:
  status:
    type: OrderStatus
    required: true
```

Enum validation occurs at bind time when `requiredEnum` is called on the `NodeCursor`. Invalid enum values produce a `INVALID_ENUM` binding error before the validator runs.

---

### Min / max constraint

```yaml
fields:
  quantity:
    type: integer
    required: true
    constraints:
      min: 1
      max: 999
```

Generated check:

```java
if (dto.quantity() != null) {
    if (dto.quantity() < 1) {
        errors.rejectValue("quantity", "CONSTRAINT_VIOLATION",
            "Field 'quantity' must be >= 1, but was: " + dto.quantity());
    }
    if (dto.quantity() > 999) {
        errors.rejectValue("quantity", "CONSTRAINT_VIOLATION",
            "Field 'quantity' must be <= 999, but was: " + dto.quantity());
    }
}
```

---

### MinLength / maxLength constraint

```yaml
fields:
  name:
    type: string
    required: true
    constraints:
      minLength: 1
      maxLength: 100
```

Generated check:

```java
if (dto.name() != null) {
    if (dto.name().length() < 1) {
        errors.rejectValue("name", "CONSTRAINT_VIOLATION",
            "Field 'name' must have length >= 1");
    }
    if (dto.name().length() > 100) {
        errors.rejectValue("name", "CONSTRAINT_VIOLATION",
            "Field 'name' must have length <= 100");
    }
}
```

---

### Pattern / regex constraint

```yaml
fields:
  postcode:
    type: string
    required: true
    constraints:
      pattern: "^[A-Z0-9 ]{3,8}$"
```

Generated check:

```java
private static final Pattern POSTCODE_PATTERN =
    Pattern.compile("^[A-Z0-9 ]{3,8}$");

if (dto.postcode() != null
        && !POSTCODE_PATTERN.matcher(dto.postcode()).matches()) {
    errors.rejectValue("postcode", "CONSTRAINT_VIOLATION",
        "Field 'postcode' does not match required pattern");
}
```

Pattern is compiled once as a static final field in the generated validator.

---

### One-of constraint

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

Generated check:

```java
long presentCount = Stream.of(dto.email(), dto.phone())
    .filter(v -> v != null && !v.isBlank())
    .count();

if (presentCount != 1) {
    errors.reject("ONE_OF_VIOLATION",
        "Exactly one of [email, phone] must be present, but " + presentCount + " were present");
}
```

---

### Mutually exclusive constraint

```yaml
constraints:
  mutuallyExclusive:
    - [legacyId, newId]
```

Generated check:

```java
if (dto.legacyId() != null && dto.newId() != null) {
    errors.reject("MUTUALLY_EXCLUSIVE_VIOLATION",
        "Fields 'legacyId' and 'newId' are mutually exclusive and must not both be present");
}
```

---

### Conditional required constraint

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

Generated check:

```java
if (dto.triggerType() == TriggerType.MESSAGE
        || dto.triggerType() == TriggerType.SIGNAL
        || dto.triggerType() == TriggerType.UPDATE) {
    if (dto.deliveryClass() == null) {
        errors.rejectValue("deliveryClass", "CONDITIONAL_CONSTRAINT_VIOLATION",
            "Field 'deliveryClass' is required when 'triggerType' is MESSAGE, SIGNAL, or UPDATE");
    }
}
```

---

### Forbidden field validation

Forbidden field detection occurs at bind time via `NodeCursor.fieldNames()` comparison before the binder processes the document. Forbidden fields produce a `FORBIDDEN_FIELD` binding error and binding fails regardless of strictness mode.

---

### Deprecated field handling

Deprecated alias usage is detected during alias resolution at bind time. In STRICT mode it produces an error. In COMPATIBLE mode it produces a warning. In LENIENT mode it is silently accepted.

---

## Unknown Field Validation

Unknown fields are detected by comparing the set of field names present in the source document against the set of declared field names and aliases.

```java
Set<String> knownNames = CustomerDtoMetadata.ALL_FIELD_NAMES;   // generated constant
Set<String> presentNames = cursor.fieldNames();

Set<String> unknownNames = new HashSet<>(presentNames);
unknownNames.removeAll(knownNames);

for (String unknown : unknownNames) {
    if (context.strictnessMode() == CompatibilityMode.STRICT) {
        errors.rejectValue(unknown, "UNKNOWN_FIELD",
            "Unknown field '" + unknown + "' is not declared in definition");
    } else {
        // Warning only
    }
}
```

---

## ValidationReport

`ValidationReport` aggregates all validation errors from generated validators, `ValidationExtension` implementations, and binding errors into one structured result.

```java
public interface ValidationReport {
    boolean hasErrors();
    boolean hasWarnings();
    List<ValidationError> errors();
    List<ValidationError> warnings();
    ValidationError errorFor(String fieldPath);  // null if none
    String summary();
}
```

```java
public record ValidationError(
    String errorCode,
    ErrorSeverity severity,
    String fieldPath,
    String message,
    @Nullable String sourceFile,
    int line,
    int column
) {}
```

---

## ConstraintValidatorFactory Integration

When custom JSR-380 constraint validators require Spring bean injection, `ConstraintValidatorFactory` is used to delegate validator instantiation to the Spring container.

```java
// In auto-configuration
@Bean
public LocalValidatorFactoryBean validatorFactory(
        ConstraintValidatorFactory constraintValidatorFactory) {
    LocalValidatorFactoryBean factory = new LocalValidatorFactoryBean();
    factory.setConstraintValidatorFactory(constraintValidatorFactory);
    return factory;
}
```

Custom constraint validators annotated with `@Component` can receive injected Spring beans:

```java
@Component
public class UniqueEmailValidator
        implements ConstraintValidator<UniqueEmail, String> {

    @Autowired
    private CustomerRepository customerRepository;

    @Override
    public boolean isValid(String email, ConstraintValidatorContext context) {
        return !customerRepository.existsByEmail(email);
    }
}
```

---

## ValidationExtension SPI

`ValidationExtension` allows adding post-generation validation logic for business rules that cannot be expressed in the schema DSL.

```java
@Component
@Order(100)
public class OrderBusinessRulesValidator implements ValidationExtension {

    @Override
    public boolean supports(Class<?> targetType) {
        return OrderDto.class.isAssignableFrom(targetType);
    }

    @Override
    public void validate(Object target, ValidationContext context) {
        OrderDto order = (OrderDto) target;

        if (order.items().stream().mapToInt(OrderItemDto::quantity).sum() > 1000) {
            context.addError("items", "BUSINESS_RULE_VIOLATION",
                "Total order quantity cannot exceed 1000");
        }

        if (order.shippingAddress() == null && order.storePickup() == null) {
            context.addError("shippingAddress", "BUSINESS_RULE_VIOLATION",
                "Either shippingAddress or storePickup must be provided");
        }
    }
}
```

---

## Strictness and Validation Behavior Summary

| Constraint Type | STRICT | COMPATIBLE | LENIENT |
|---|---|---|---|
| Missing required field | Error | Error | Error |
| Unknown field | Error | Warning | Ignored |
| Forbidden field | Error | Error | Error |
| Deprecated alias | Error | Warning | Ignored |
| Invalid enum | Error | Error | Error |
| Constraint violation (min/max/etc.) | Error | Error | Error |
| One-of violation | Error | Error | Error |
| Mutually exclusive violation | Error | Error | Error |
| Conditional required violation | Error | Error | Error |

Required fields, forbidden fields, and structural constraint violations are always errors. They cannot be suppressed by mode configuration.
