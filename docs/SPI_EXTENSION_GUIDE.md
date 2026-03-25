# TwinMapper — SPI and Extension Guide

## Overview

TwinMapper provides a controlled set of Service Provider Interfaces (SPIs) that allow consuming applications and library authors to extend the platform without modifying core modules. All SPIs follow Spring's ordering model and are resolved deterministically using `OrderComparator`, `PriorityOrdered`, `@Order`, and `Ordered`.

Extensions must never bypass the compile-time, definition-first architecture. They extend behavior within the established model — they do not replace it.

---

## SPI Inventory

| SPI Interface | Purpose | Module |
|---|---|---|
| `DefinitionReader` | Load definition sources into `DefinitionSet` | `twinmapper-core` |
| `RuntimeDocumentParser` | Parse runtime documents into `NodeCursor` | `twinmapper-runtime` |
| `ValidationExtension` | Additional runtime validation after generated validators | `twinmapper-validation` |
| `ValueConverter` | Convert between declared Java types | `twinmapper-runtime` |
| `CodegenCustomizer` | Modify generated type specs before source is written | `twinmapper-codegen` |
| `TwinMapperRuntimeConfigurer` | Programmatic runtime customization | `twinmapper-runtime` |
| `ConventionMappingResolver` | Convention-based field resolution (optional mode only) | `twinmapper-runtime-objectmap` |

---

## DefinitionReader SPI

`DefinitionReader` allows adding support for new definition source formats or custom definition loading strategies.

```java
public interface DefinitionReader {

    /**
     * Returns true if this reader can handle the given source.
     */
    boolean supports(DefinitionSource source);

    /**
     * Reads the source and produces a DefinitionSet.
     * Throws DefinitionReadException on parse or validation failure.
     */
    DefinitionSet read(DefinitionSource source);
}
```

### Registration

Implement `DefinitionReader` and register it as a Spring bean or via `TwinMapperRuntimeConfigurer.addDefinitionReaders(...)`. When multiple readers are present, resolution order follows `@Order` or `PriorityOrdered`.

```java
@Component
@Order(10)
public class CustomDefinitionReader implements DefinitionReader {

    @Override
    public boolean supports(DefinitionSource source) {
        return source.getLocation().endsWith(".twdef");
    }

    @Override
    public DefinitionSet read(DefinitionSource source) {
        // Parse custom format into DefinitionSet
    }
}
```

---

## RuntimeDocumentParser SPI

`RuntimeDocumentParser` allows adding runtime document parsing support for new formats. Each format module provides its own implementation for YAML, JSON, and BPMN.

```java
public interface RuntimeDocumentParser {

    /**
     * Returns true if this parser supports the given document format.
     */
    boolean supports(DocumentFormat format);

    /**
     * Parses the input stream and returns a NodeCursor for traversal.
     * Throws DocumentParseException on malformed input.
     */
    NodeCursor parse(InputStream input, ParsingContext context);
}
```

### Registration

```java
@Component
@Order(20)
public class CustomFormatParser implements RuntimeDocumentParser {

    @Override
    public boolean supports(DocumentFormat format) {
        return format == DocumentFormat.CUSTOM;
    }

    @Override
    public NodeCursor parse(InputStream input, ParsingContext context) {
        // Parse and return NodeCursor implementation
    }
}
```

---

## ValidationExtension SPI

`ValidationExtension` allows adding validation logic that runs after all generated schema validators have completed. This is for cross-field or domain-level checks that are not expressible in the schema DSL.

```java
public interface ValidationExtension {

    /**
     * Returns true if this extension applies to the given target type.
     */
    boolean supports(Class<?> targetType);

    /**
     * Performs additional validation on the bound target.
     * Adds errors to the ValidationContext.
     */
    void validate(Object target, ValidationContext context);
}
```

### Registration

```java
@Component
@Order(100)
public class WorkflowSemanticValidator implements ValidationExtension {

    @Override
    public boolean supports(Class<?> targetType) {
        return WorkflowDefinitionDto.class.isAssignableFrom(targetType);
    }

    @Override
    public void validate(Object target, ValidationContext context) {
        WorkflowDefinitionDto dto = (WorkflowDefinitionDto) target;
        if (dto.states().isEmpty()) {
            context.addError("states", "Workflow must have at least one state");
        }
    }
}
```

---

## ValueConverter SPI

`ValueConverter` is the TwinMapper abstraction over Spring's `Converter`, `ConverterFactory`, and `GenericConverter`. In Spring applications, converters implementing Spring's native SPI contracts are automatically available. The `ValueConverter` SPI is for non-Spring environments or converters requiring TwinMapper-specific metadata.

```java
public interface ValueConverter {

    /**
     * Returns true if this converter can convert from sourceType to targetType.
     */
    boolean supports(TypeDescriptor sourceType, TypeDescriptor targetType);

    /**
     * Converts the value. May return null only if the target type allows null.
     */
    Object convert(Object value, TypeDescriptor sourceType, TypeDescriptor targetType, ConversionContext context);
}
```

### Spring Integration

In Spring Boot applications, prefer implementing Spring's `Converter<S, T>` directly and registering via the application context. TwinMapper's `ConversionService` integration picks these up automatically.

```java
@Component
public class CustomerIdConverter implements Converter<Long, CustomerId> {

    @Override
    public CustomerId convert(Long source) {
        return source != null ? new CustomerId(source) : null;
    }
}
```

### Registration via Configurer

```java
@Configuration
public class ConverterConfig implements TwinMapperRuntimeConfigurer {

    @Override
    public void addConverters(ConverterRegistry registry) {
        registry.addConverter(new CustomerIdConverter());
        registry.addConverter(new CustomerStatusConverter());
        registry.addConverterFactory(new StatusCodeConverterFactory());
    }
}
```

---

## CodegenCustomizer SPI

`CodegenCustomizer` allows post-processing of generated type specifications before Java source is written to disk. This is for advanced use cases such as adding custom annotations, modifying generated method signatures, or injecting additional imports.

```java
public interface CodegenCustomizer {

    /**
     * Returns true if this customizer applies to the given type definition.
     */
    boolean supports(TypeDefinition definition);

    /**
     * Called with the TypeSpecBuilder before the source file is written.
     * Modifications are applied to the generated source.
     */
    void customize(TypeSpecBuilder typeSpec, TypeDefinition definition, CodegenContext context);
}
```

### Example

```java
public class JsonSerializableCustomizer implements CodegenCustomizer {

    @Override
    public boolean supports(TypeDefinition definition) {
        return definition instanceof ObjectTypeDefinition;
    }

    @Override
    public void customize(TypeSpecBuilder typeSpec, TypeDefinition definition, CodegenContext context) {
        typeSpec.addAnnotation(JsonSerialize.class);
        typeSpec.addAnnotation(JsonDeserialize.class);
    }
}
```

### Registration

Register in the Gradle or Maven plugin configuration. `CodegenCustomizer` implementations are build-time only — they are not Spring beans.

```groovy
twinmapper {
    codegenCustomizers = [
        'com.example.codegen.JsonSerializableCustomizer'
    ]
}
```

---

## TwinMapperRuntimeConfigurer

`TwinMapperRuntimeConfigurer` is the primary Spring-idiomatic extension point for programmatic runtime customization. It follows the same pattern as `WebMvcConfigurer` in Spring MVC.

```java
public interface TwinMapperRuntimeConfigurer {

    default void addConverters(ConverterRegistry registry) {}

    default void addDefinitionReaders(DefinitionReaderRegistry registry) {}

    default void addValidationExtensions(ValidationExtensionRegistry registry) {}

    default void configureStrictness(StrictnessConfigurer configurer) {}

    default void configureNullPolicy(NullPolicyConfigurer configurer) {}

    default void addDocumentParsers(DocumentParserRegistry registry) {}
}
```

Multiple `TwinMapperRuntimeConfigurer` implementations are allowed. They are collected and applied in `@Order` sequence.

```java
@Configuration
@Order(1)
public class CoreTwinMapperConfig implements TwinMapperRuntimeConfigurer {

    @Override
    public void addConverters(ConverterRegistry registry) {
        registry.addConverter(new MoneyConverter());
    }
}

@Configuration
@Order(2)
public class DomainTwinMapperConfig implements TwinMapperRuntimeConfigurer {

    @Override
    public void addValidationExtensions(ValidationExtensionRegistry registry) {
        registry.add(new OrderSemanticValidator());
    }
}
```

---

## ConventionMappingResolver SPI (Optional)

This SPI is only relevant when `twinmapper.objectmap.convention-mapping.enabled=true`. It provides the logic for resolving field mappings by naming convention when no explicit mapping is declared.

```java
public interface ConventionMappingResolver {

    /**
     * Attempts to resolve a target field name from a source field name.
     * Returns Optional.empty() if no convention match is found.
     * Must never guess silently — ambiguity must throw ConventionResolutionException.
     */
    Optional<String> resolve(String sourceFieldName, Class<?> sourceType, Class<?> targetType, ConventionContext context);
}
```

The `ConventionMatchingPolicy` configures how strict convention resolution must be. The policy always has `failOnAmbiguity=true` — this is not configurable.

---

## SPI Ordering Rules

All TwinMapper SPIs are resolved in deterministic order. Follow these rules.

- Use `@Order(n)` on Spring bean implementations. Lower values run first.
- For non-Spring implementations, implement `PriorityOrdered` and return the priority from `getOrder()`.
- TwinMapper's own built-in implementations use `@Order(0)`. Custom implementations should use values of 10 or higher to avoid collisions.
- When two implementations have the same order value, registration order is used as a tiebreaker.

```java
// Built-in TwinMapper implementation
@Order(0)
class DefaultJsonParser implements RuntimeDocumentParser { ... }

// Custom implementation runs after built-in
@Component
@Order(10)
public class CustomJsonParser implements RuntimeDocumentParser { ... }
```

---

## Extension Pack Model

TwinMapper SPIs are the foundation for building domain-specific extension packs. An extension pack is a separate library — not part of TwinMapper itself — that registers one or more SPI implementations for a specific domain.

Extension packs are built and owned outside TwinMapper. TwinMapper provides the SPI contracts and packaging model. Consuming applications add extension packs as dependencies alongside the TwinMapper core modules.

```xml
<!-- Example: adding a hypothetical finance extension pack -->
<dependency>
    <groupId>com.example</groupId>
    <artifactId>twinmapper-finance-pack</artifactId>
    <version>1.0.0</version>
</dependency>
```

The finance pack registers its own `DefinitionReader`, `ValidationExtension`, and `ValueConverter` implementations via Spring auto-configuration. No changes to TwinMapper core are needed.
