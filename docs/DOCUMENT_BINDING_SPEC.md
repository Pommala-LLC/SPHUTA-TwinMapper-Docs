# TwinMapper — Document Binding Specification

## Overview

The Document Engine binds runtime YAML, JSON, and BPMN XML documents into generated DTOs. Binding is performed by generated binder classes using a `NodeCursor` abstraction. No reflection is used in the binding path.

---

## Binding Pipeline

```
Runtime Document (YAML / JSON / BPMN XML)
    │
    ▼
Format Parser
(SnakeYAML / Jackson / JDK StAX)
    │
    ▼
NodeCursor
(format-neutral traversal abstraction)
    │
    ▼
Generated Binder
(dispatched by GeneratedBinderRegistry)
    │
    ▼
Generated DTO
    │
    ▼
BindingResult<T>
(populated DTO or accumulated errors)
```

---

## NodeCursor Contract

`NodeCursor` is the core runtime abstraction separating format parsing from binding logic. Every format module provides its own implementation. Generated binders depend only on `NodeCursor` — they have no direct dependency on Jackson, SnakeYAML, or StAX.

```java
public interface NodeCursor {

    // Field existence
    boolean hasField(String name);
    boolean hasField(String name, List<String> aliases);

    // Required scalar reads — throw MissingFieldException if absent
    String requiredString(String name);
    String requiredString(String name, List<String> aliases);
    Integer requiredInteger(String name);
    Long requiredLong(String name);
    Double requiredDouble(String name);
    Boolean requiredBoolean(String name);
    BigDecimal requiredDecimal(String name);
    <E extends Enum<E>> E requiredEnum(String name, Class<E> enumType);
    <E extends Enum<E>> E requiredEnum(String name, List<String> aliases, Class<E> enumType);

    // Optional scalar reads — return Optional.empty() if absent
    Optional<String> optionalString(String name);
    Optional<String> optionalString(String name, List<String> aliases);
    Optional<Integer> optionalInteger(String name);
    Optional<Long> optionalLong(String name);
    Optional<Boolean> optionalBoolean(String name);
    Optional<BigDecimal> optionalDecimal(String name);
    <E extends Enum<E>> Optional<E> optionalEnum(String name, Class<E> enumType);

    // Nested object access
    NodeCursor requiredChild(String name);
    Optional<NodeCursor> optionalChild(String name);

    // Collection access
    List<NodeCursor> children(String name);

    // Map access
    Map<String, NodeCursor> mapChildren(String name);

    // Diagnostics
    SourceLocation location();
    String fieldPath();
    Set<String> fieldNames();

    // Depth
    int depth();
}
```

---

## BindingContext

`BindingContext` carries the configuration and state for a single binding operation.

```java
public interface BindingContext {

    CompatibilityMode strictnessMode();
    AliasResolutionPolicy aliasResolutionPolicy();
    GeneratedBinderRegistry binderRegistry();
    ErrorAccumulator errors();
    int maxDepth();

    static BindingContext strict() { ... }
    static BindingContext compatible() { ... }
}
```

---

## BindingResult

`BindingResult<T>` is the output of every binding operation. Contains either a successfully populated DTO or a list of accumulated errors.

```java
public interface BindingResult<T> {
    boolean hasErrors();
    T value();                          // throws if hasErrors()
    T valueOrNull();                    // returns null if hasErrors()
    List<BindingError> errors();
}
```

### BindingError

```java
public record BindingError(
    String errorCode,
    String fieldPath,
    String sourceFile,
    int line,
    int column,
    String message,
    @Nullable String migrationHint
) {}
```

---

## Runtime API

### Strict binding (throws on error)

```java
ProductDto product = runtime.readYaml(inputStream, ProductDto.class);
CustomerDto customer = runtime.readJson(inputStream, CustomerDto.class);
WorkflowDto workflow = runtime.readBpmn(inputStream, WorkflowDto.class);
```

### Safe binding (returns BindingResult)

```java
BindingResult<ProductDto> result = runtime.readYamlSafe(inputStream, ProductDto.class);

if (result.hasErrors()) {
    result.errors().forEach(e ->
        log.warn("Binding error at {}: {}", e.fieldPath(), e.message()));
    return;
}

ProductDto product = result.value();
```

---

## Alias Resolution

Aliases allow alternative source field names to bind to the canonical target field. Alias resolution is transparent — no consumer code changes are needed.

### DSL declaration

```yaml
fields:
  name:
    type: string
    required: true
    aliases: [productName, product_name, ProductName]
```

### Generated binder behavior

```java
// Generated — the binder tries all aliases before reporting a missing field
String name = cursor.requiredString("name", List.of("productName", "product_name", "ProductName"));
```

Resolution order: canonical name first, then aliases in declaration order.

---

## Default Value Application

Absent optional fields with a declared default are populated at bind time.

```yaml
fields:
  tags:
    type: list<string>
    required: false
    default: []

  status:
    type: OrderStatus
    required: false
    default: PENDING
```

Generated binder applies defaults after checking for field presence:

```java
List<String> tags = cursor.optionalString("tags").isPresent()
    ? cursor.children("tags").stream()
        .map(c -> c.requiredString("$"))
        .collect(Collectors.toList())
    : List.of();   // default applied

OrderStatus status = cursor.optionalEnum("status", OrderStatus.class)
    .orElse(OrderStatus.PENDING);  // default applied
```

---

## Required Field Enforcement

Required fields produce a `MISSING_REQUIRED_FIELD` error if absent and not resolvable via any alias.

In STRICT and COMPATIBLE modes, required field errors accumulate and all errors are reported before failing. In LENIENT mode, absent required fields produce a warning and binding continues with null.

---

## Unknown Field Handling

| Mode | Behavior |
|---|---|
| STRICT (default) | `UNKNOWN_FIELD` error — binding fails |
| COMPATIBLE | `UNKNOWN_FIELD` warning — field ignored, binding continues |
| LENIENT | Unknown fields silently ignored |

Unknown field detection uses `NodeCursor.fieldNames()` to enumerate all fields present in the source document and compares against the set of declared field names and aliases.

---

## Forbidden Field Handling

Fields declared as `forbiddenFields` produce a `FORBIDDEN_FIELD` error regardless of strictness mode. A migration hint is included in the error.

```yaml
types:
  WorkflowDefinition:
    kind: object
    forbiddenFields:
      - interrupting
    fields:
      interruptingMode:
        type: InterruptingMode
        required: false
```

Binding a document containing `interrupting: true` produces:

```
FORBIDDEN_FIELD at 'interrupting': Field 'interrupting' is forbidden.
Migration hint: Use 'interruptingMode' instead.
```

---

## Nested Binding

Nested objects are bound recursively. The `BindingContext` is passed through the recursive call. Error paths are prefixed with the parent field path.

```java
// Generated nested binding
Optional<NodeCursor> addressNode = cursor.optionalChild("address");
AddressDto address = addressNode
    .map(child -> context.binderRegistry()
        .binderFor(AddressDto.class)
        .bind(child, context.nested("address")))
    .orElse(null);
```

Error reported as `address.city: MISSING_REQUIRED_FIELD` rather than just `city`.

---

## Collection Binding

```java
// Generated list binding
List<NodeCursor> itemNodes = cursor.children("items");
List<OrderItemDto> items = itemNodes.stream()
    .map(itemNode -> context.binderRegistry()
        .binderFor(OrderItemDto.class)
        .bind(itemNode, context))
    .collect(Collectors.toList());
```

Empty list in source: produces empty `List.of()`. Absent collection field with default `[]`: default is applied.

---

## Depth Limit

`NodeCursor` enforces a maximum traversal depth (default: 64). Documents exceeding this depth produce a `BINDING_DEPTH_EXCEEDED` error. This prevents stack overflow attacks via deeply nested documents.

```java
if (cursor.depth() > context.maxDepth()) {
    throw new BindingDepthExceededException(cursor.fieldPath(), context.maxDepth());
}
```

---

## Format-Specific Behavior

### YAML

- SnakeYAML parses the document into a `MappingNode`.
- `YamlNodeCursor` wraps the `MappingNode` providing `NodeCursor` methods.
- Kebab-case field names (`trigger-type`) are matched against camelCase (`triggerType`) and all declared aliases.
- Safe construction is mandatory: `new Yaml(new SafeConstructor(new LoaderOptions()))`.
- Line and column numbers are available from SnakeYAML's `Mark` for error reporting.

### JSON

- Jackson parses the document into a `JsonNode` tree.
- `JsonNodeCursor` wraps the `ObjectNode` providing `NodeCursor` methods.
- Jackson's `JsonParser` location provides line and column numbers for error reporting.
- `JsonMapper.builder()` is used for consistent Jackson configuration.

### BPMN XML

- JDK StAX parses the XML with XXE disabled.
- The raw XML is first parsed into a `BpmnIntermediateModel`.
- `BpmnIntermediateModel` is adapted into a `BpmnNodeCursor` for `NodeCursor`-compatible traversal.
- Line numbers are available from the StAX `XMLStreamReader.getLocation()`.
- Unsupported BPMN elements are skipped at the intermediate model level with a diagnostic warning.

---

## Error Accumulation

Errors accumulate across a binding operation and are all reported together. Binding does not fail-fast unless a required field error prevents further traversal.

```java
BindingResult<OrderDto> result = runtime.readJsonSafe(input, OrderDto.class);

if (result.hasErrors()) {
    // All errors, not just the first
    result.errors().forEach(error -> {
        System.out.printf("[%s] %s at %s (line %d, col %d)%n",
            error.errorCode(),
            error.message(),
            error.fieldPath(),
            error.line(),
            error.column());
    });
}
```
