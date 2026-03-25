# SPHUTA TwinMapper — Phase 1 Foundation Specification

**Version:** Final Lock  
**Platform:** SPHUTA — Enterprise Integration Platform  
**Component:** TwinMapper — Compile-time-first, schema-driven code generation and mapping framework  
**Baseline:** Java 21 / Spring Framework 7.x / Spring Boot 4.x  

---

## 1. Purpose of Phase 1

Phase 1 establishes the shared language and the canonical internal model of TwinMapper.

These two modules exist to solve two foundational problems: how all TwinMapper modules communicate consistently, and what a TwinMapper definition means internally — independent of YAML, JSON, or BPMN syntax.

Without these modules, every later phase would make local decisions about errors, identity, source traceability, and internal structure, and the platform would fragment.

Phase 1 defines:

- the shared technical vocabulary of TwinMapper
- the canonical definition vocabulary of TwinMapper
- the minimal SPI boundary needed for future reader modules
- the invariants that later phases can trust

---

## 2. Scope

| Category | Items |
|----------|-------|
| **In scope** | `twinmapper-core`, `twinmapper-definition-model` |
| **Out of scope** | YAML / JSON / BPMN parsing implementations, normalization pipelines beyond the canonical model boundary, code generation classes and strategies, runtime binding and object mapping engines, Spring Boot autoconfiguration, CLI behavior, annotation processing, compatibility and reflection helpers, import resolution and multi-file linking, extension metadata model |

---

## 3. Global Phase 1 Decisions

### 3.1 Locked decisions

| Decision | Lock |
|----------|------|
| Package root | `net.sphuta.twinmapper` |
| Java version | 21 |
| Spring Framework | 7.x |
| Spring Boot | 4.x |
| Null-safety standard | JSpecify 1.0.0 (`org.jspecify:jspecify:1.0.0`) |
| Null-safety annotation | `@NullMarked` on every `package-info.java`, `@Nullable` from `org.jspecify.annotations` |
| Spring dependency | `spring-core:7.x` compile on `twinmapper-core` only |
| Spring context | Not allowed in Phase 1 — no bean scanning, no autoconfiguration |
| Package granularity | Flat layout — 7 packages in core, 6 packages in definition-model |
| Diagnostic model | Fully consistent `Diagnostic*` naming |
| Diagnostic payload | `TwinMapperDiagnostic` typed record |
| Diagnostic code | `DiagnosticCode` single enum |
| Identity model | Namespace + name + version; imports deferred |
| Type hierarchy | Sealed interface — permits `ObjectTypeDefinition` and `EnumTypeDefinition` |
| Type reference | Identity + nullability only; no collection flag |
| Container semantics | `CollectionDefinition` and `MapDefinition` on `FieldDefinition` |
| Mapping model | Full canonical model — modes, profiles, converters, null policy, inverseOf |
| Mapping type fields | `TypeReference` (not raw String) |
| Mapping path fields | `FieldPath` (not raw String) |
| Extension metadata | Deferred |
| Source traceability | Single `SourceLocation` — no range, no reference |
| VersionDefinition | Dropped — version lives only in `DefinitionSetId` |
| Exception hierarchy | Full hierarchy defined in core; Phase 1 uses subset |
| ConfigurationException | Dropped — no concrete use case |
| Result wrapper | Dropped — use `DiagnosticCollector` directly |
| Builder pattern | Inner classes on records |
| Policy enum placement | Split by domain — `CompatibilityMode` in core; others in definition-model |

### 3.2 Dependency direction

```
twinmapper-core
    ↑
twinmapper-definition-model
```

`definition-model` depends on `core`. `core` must not depend on `definition-model`. No reverse dependency. No circular shortcuts.

### 3.3 Design principles

| Principle | Description |
|-----------|-------------|
| Build-time-first foundation | Foundation serves compile-time code generation as the primary path |
| Format-neutral canonical model | No YAML, JSON, or BPMN concepts in the model |
| Immutable public model objects | All public canonical types are immutable; builders are mutable |
| Typed diagnostic structure | Structured payloads, not raw strings or generic metadata maps |
| Single source-location abstraction | Point-based traceability, not range-based |
| Structured identity | Namespace + name + version, not plain strings |
| Strict phase separation | No later-phase concerns leak into Phase 1 |

---

## 4. Module Specification — `twinmapper-core`

### 4.1 Module purpose

`twinmapper-core` defines the shared technical language of TwinMapper. Every later module uses it to communicate consistently about diagnostics, exceptions, source traceability, naming, identity, paths, and shared SPI contracts.

### 4.2 Responsibility matrix

| Responsible for | Not responsible for |
|----------------|---------------------|
| Shared diagnostic model | Canonical definitions (`DefinitionSet`) |
| Shared exception hierarchy | YAML / JSON / BPMN parsing |
| Source traceability primitives | Code generation logic |
| Identity and naming primitives | Runtime binding logic |
| Type reference abstraction | Runtime mapping logic |
| Field and path primitives | Validation engine logic |
| Diagnostic accumulation contract | Spring bean registration |
| Minimal reader SPI contract | Extension metadata models |
| Shared utility layer | Import / reference resolution |
| Cross-platform policy enums | Mapping-specific or generation-specific policies |

### 4.3 Artifact and package root

| Property | Value |
|----------|-------|
| Artifact | `twinmapper-core` |
| Base package | `net.sphuta.twinmapper.core` |
| Compile dependencies | `spring-core:7.x`, `org.jspecify:jspecify:1.0.0` |

### 4.4 Package layout

| Package | Purpose |
|---------|---------|
| `core.diagnostic` | Diagnostic model, codes, categories, severity, collector, exceptions |
| `core.naming` | Namespace, QualifiedName, TypeReference |
| `core.path` | FieldPath, PathSegment |
| `core.policy` | CompatibilityMode (cross-platform only) |
| `core.source` | SourceLocation |
| `core.spi` | DefinitionReader, DefinitionSource, DocumentFormat |
| `core.util` | Constants, field name conversion, preconditions |

### 4.5 Package: `core.diagnostic`

#### Types

| Type | Kind | Description |
|------|------|-------------|
| `TwinMapperDiagnostic` | Record + Builder | Canonical structured diagnostic payload |
| `DiagnosticCode` | Enum | All diagnostic codes with default severity and category |
| `DiagnosticCategory` | Enum | High-level classification of diagnostics |
| `DiagnosticSeverity` | Enum | Severity level of each diagnostic |
| `DiagnosticCollector` | Interface | Mutable accumulation contract for diagnostics |
| `DefaultDiagnosticCollector` | Class | Default in-memory implementation |
| `TwinMapperException` | Class | Base runtime exception wrapping diagnostics |
| `DefinitionException` | Class | Extends `TwinMapperException` |
| `BindingException` | Class | Extends `TwinMapperException` |
| `MappingException` | Class | Extends `TwinMapperException` |
| `ValidationException` | Class | Extends `TwinMapperException` |
| `CodegenException` | Class | Extends `TwinMapperException` |
| `TwinMapperSecurityException` | Class | Extends `TwinMapperException` |

#### `TwinMapperDiagnostic` fields

| Field | Type | Nullable | Description |
|-------|------|----------|-------------|
| `code` | `DiagnosticCode` | No | Machine-readable diagnostic identifier |
| `severity` | `DiagnosticSeverity` | No | ERROR, WARNING, or INFO |
| `fieldPath` | `String` | No | Dot-separated path to offending field (empty for root-level) |
| `message` | `String` | No | Human-readable description |
| `location` | `SourceLocation` | No | Source file, line, column where diagnostic originated |
| `targetType` | `String` | Yes | Fully qualified Java type being generated or bound into |
| `ruleViolated` | `String` | Yes | Description of the constraint that failed |
| `migrationHint` | `String` | Yes | Guidance for forbidden or deprecated field diagnostics |
| `expectedValue` | `String` | Yes | Expected type or value for type mismatch diagnostics |
| `actualValue` | `String` | Yes | Actual type or value received |

#### `DiagnosticSeverity` values

| Value | Behavior |
|-------|----------|
| `ERROR` | Processing fails. No result produced. |
| `WARNING` | Processing continues. Issue reported. |
| `INFO` | Informational only. No action required. |

#### `DiagnosticCategory` values

| Value | Scope |
|-------|-------|
| `PARSE` | Definition file parsing (YAML, JSON, BPMN syntax) |
| `DEFINITION` | Semantic errors in parsed definitions |
| `VALIDATION` | Constraint and structural validation |
| `GENERATION` | Code generation errors |
| `BINDING` | Runtime document binding |
| `MAPPING` | Runtime object mapping |
| `CONFIGURATION` | Configuration and property errors |
| `SECURITY` | XXE, unsafe YAML construction |

#### `DiagnosticCode` catalog (Phase 1 complete set)

| Code | Default Severity | Category |
|------|-----------------|----------|
| `DEFINITION_PARSE_ERROR` | ERROR | PARSE |
| `DEFINITION_VALIDATION_ERROR` | ERROR | DEFINITION |
| `UNKNOWN_TYPE_REFERENCE` | ERROR | DEFINITION |
| `CIRCULAR_TYPE_REFERENCE` | ERROR | DEFINITION |
| `INVERSE_MAPPING_UNSAFE` | ERROR | DEFINITION |
| `AMBIGUOUS_CONVENTION_MAPPING` | ERROR | DEFINITION |
| `CODEGEN_SPLIT_PACKAGE` | ERROR | GENERATION |
| `MISSING_REQUIRED_FIELD` | ERROR | BINDING |
| `UNKNOWN_FIELD` | ERROR | BINDING |
| `FORBIDDEN_FIELD` | ERROR | BINDING |
| `DEPRECATED_FIELD` | ERROR | BINDING |
| `INVALID_ENUM` | ERROR | BINDING |
| `TYPE_MISMATCH` | ERROR | BINDING |
| `NESTED_BINDING_ERROR` | ERROR | BINDING |
| `BINDING_DEPTH_EXCEEDED` | ERROR | BINDING |
| `CONSTRAINT_VIOLATION` | ERROR | VALIDATION |
| `CONDITIONAL_CONSTRAINT_VIOLATION` | ERROR | VALIDATION |
| `ONE_OF_VIOLATION` | ERROR | VALIDATION |
| `MUTUALLY_EXCLUSIVE_VIOLATION` | ERROR | VALIDATION |
| `VALIDATION_FAILED` | ERROR | VALIDATION |
| `NO_MAPPER_FOUND` | ERROR | MAPPING |
| `MAPPING_ERROR` | ERROR | MAPPING |
| `NULL_REQUIRED_MAPPING_FIELD` | ERROR | MAPPING |
| `DEPRECATED_ALIAS_USED` | WARNING | BINDING |
| `UNKNOWN_FIELD_IGNORED` | WARNING | BINDING |
| `CONVENTION_MAPPING_APPLIED` | WARNING | MAPPING |
| `XXE_REJECTED` | ERROR | SECURITY |
| `UNSAFE_YAML_CONSTRUCTION_REJECTED` | ERROR | SECURITY |

#### `DiagnosticCollector` operations

| Method | Return | Description |
|--------|--------|-------------|
| `add(TwinMapperDiagnostic)` | `void` | Add one diagnostic |
| `addAll(Collection<TwinMapperDiagnostic>)` | `void` | Add multiple diagnostics |
| `hasErrors()` | `boolean` | True if any ERROR severity present |
| `hasWarnings()` | `boolean` | True if any WARNING severity present |
| `errors()` | `List<TwinMapperDiagnostic>` | All ERROR items (immutable snapshot) |
| `warnings()` | `List<TwinMapperDiagnostic>` | All WARNING items (immutable snapshot) |
| `all()` | `List<TwinMapperDiagnostic>` | All items (immutable snapshot) |

#### Exception hierarchy

| Exception | Extends | Purpose |
|-----------|---------|---------|
| `TwinMapperException` | `RuntimeException` | Base; carries `List<TwinMapperDiagnostic>` |
| `DefinitionException` | `TwinMapperException` | Parse and definition errors |
| `BindingException` | `TwinMapperException` | Runtime binding errors |
| `MappingException` | `TwinMapperException` | Runtime object mapping errors |
| `ValidationException` | `TwinMapperException` | Constraint violations |
| `CodegenException` | `TwinMapperException` | Generation errors |
| `TwinMapperSecurityException` | `TwinMapperException` | XXE, unsafe YAML |

#### Diagnostic package rules

- `DiagnosticCollector` is preferred over immediate exception throwing
- Exceptions are boundary constructs, not the primary internal flow
- Every exception must be convertible back to `TwinMapperDiagnostic`
- Diagnostics are accumulated and reported; exceptions are thrown and stop execution

### 4.6 Package: `core.naming`

#### Types

| Type | Kind | Fields | Description |
|------|------|--------|-------------|
| `Namespace` | Record | `value` (String) | Namespace part of identity. `ROOT` sentinel. Validated: non-blank. |
| `QualifiedName` | Record | `namespace` (Namespace), `localName` (String) | Namespace-qualified logical name. Validated: non-null namespace, non-blank localName. |
| `TypeReference` | Record | `qualifiedName` (QualifiedName), `nullable` (boolean) | Normalized reference to a type. Identity + nullability only. No collection flag. |

#### Naming rules

| Rule | Detail |
|------|--------|
| Case sensitivity | Names are case-sensitive unless explicitly normalized elsewhere |
| Name validation | Belongs in this package |
| QualifiedName scope | Does not include file location or source syntax details |
| TypeReference scope | Logical, not file-based. No import semantics in Phase 1. |
| TypeReference container | No collection flag. Container semantics live in `CollectionDefinition` / `MapDefinition`. |

### 4.7 Package: `core.path`

#### Types

| Type | Kind | Description |
|------|------|-------------|
| `FieldPath` | Value object | Normalized logical field path. Supports parse, append, parent, segments, depth, equality. Immutable. Examples: `customer.id`, `address.city`, `lineItems[].amount` |
| `PathSegment` | Record | One normalized segment of a `FieldPath`. |

#### Rules

- Path semantics are logical, not file-format-specific
- Path objects are immutable
- Append returns a new instance; original is unchanged

### 4.8 Package: `core.policy`

#### Types

| Type | Kind | Values | Description |
|------|------|--------|-------------|
| `CompatibilityMode` | Enum | `STRICT`, `COMPATIBLE`, `LENIENT` | Cross-platform strictness behavior |

This is the only policy enum in `twinmapper-core`. All other policy enums live in `twinmapper-definition-model`.

### 4.9 Package: `core.source`

#### Types

| Type | Kind | Fields | Description |
|------|------|--------|-------------|
| `SourceLocation` | Record | `sourceId` (String), `line` (int), `column` (int), `fragmentId` (String, nullable) | Point-based source traceability. `UNKNOWN` sentinel. Factory methods: `of(sourceId, line, column)`, `ofFile(sourceId)`. |

#### Rules

| Rule | Detail |
|------|--------|
| No `SourceRange` | Deferred beyond Phase 1 |
| No `SourceReference` | Deferred beyond Phase 1 |
| Traceability model | Point-based only |
| `UNKNOWN` sentinel | Used for programmatic definitions with no source file |

### 4.10 Package: `core.spi`

#### Types

| Type | Kind | Description |
|------|------|-------------|
| `DefinitionSource` | Record | Abstract source of definition content. Fields: `location` (String), `formatHint` (DocumentFormat), `inputStreamSupplier` (Supplier). |
| `DocumentFormat` | Enum | Definition source format: `YAML`, `JSON`, `BPMN`, `UNKNOWN`. |
| `DefinitionReader<R>` | Interface (generic) | Stable reader SPI boundary. Generic because `core` must not depend on `definition-model`. Contract: `supports(DefinitionSource)`, `read(DefinitionSource, DiagnosticCollector)`. |

#### SPI rules

| Rule | Detail |
|------|--------|
| Minimal scope | No reader registry, resolution engine, or orchestration logic |
| No post-processors | Deferred to later phases |
| No normalizers | Deferred to later phases |
| No codegen SPIs | Deferred to Phase 3 |
| Generic reader | `R` is bound to the canonical result type by implementing modules |

### 4.11 Package: `core.util`

#### Types

| Type | Kind | Description |
|------|------|-------------|
| `TwinMapperConstants` | Final class | Shared constants: `GENERATED_SOURCE_DIR`, `DEFAULT_BASE_PACKAGE`, `DEFAULT_MAX_DEPTH`, `VERSION` |
| `FieldNameUtils` | Utility class | Bidirectional conversion: camelCase ↔ kebab-case ↔ snake_case |
| `Preconditions` | Utility class | `requireNonBlank`, `requirePositive`, `requireNonEmpty` |

#### Allowed Spring utilities

| Utility | Source |
|---------|--------|
| `Assert` | `spring-core` |
| `StringUtils` | `spring-core` |
| `ObjectUtils` | `spring-core` |
| `Ordered` | `spring-core` |
| `PriorityOrdered` | `spring-core` |
| `@Order` | `spring-core` |
| `OrderComparator` | `spring-core` |

#### Utility rules

- No Spring context usage
- No resource scanning
- No application context interactions
- No bean definitions

### 4.12 Core invariants

| Invariant | Description |
|-----------|-------------|
| Diagnostic completeness | `TwinMapperDiagnostic` is always structurally complete (all required fields non-null) |
| Code integrity | `DiagnosticSeverity` and `DiagnosticCategory` are always present on every code |
| Name validity | `QualifiedName` always has a valid namespace and non-blank local name |
| Path normalization | `FieldPath` is always normalized |
| Source identity | `SourceLocation` always has a source identifier or is `UNKNOWN` |
| SPI isolation | SPI interfaces must not depend on `definition-model` |

### 4.13 Core completion criteria

`twinmapper-core` is complete when:

- every shared diagnostic concept is defined once
- every shared naming, source, and path primitive is defined once
- the minimal SPI boundary exists without creating dependency inversion
- later modules can consistently report and aggregate diagnostics through it

---

## 5. Module Specification — `twinmapper-definition-model`

### 5.1 Module purpose

`twinmapper-definition-model` defines the canonical internal meaning of TwinMapper definitions. It is the format-neutral model that later phases will produce and consume. It answers one question: once authoring syntax is stripped away, what does a TwinMapper definition actually look like?

### 5.2 Responsibility matrix

| Responsible for | Not responsible for |
|----------------|---------------------|
| Root canonical container (`DefinitionSet`) | YAML / JSON / BPMN syntax DTOs |
| Sealed type system (Object, Enum) | Parser-specific structures |
| Field declarations | Import resolution behavior |
| Constraint declarations | Multi-file linking behavior |
| Object mapping declarations (full model) | Extension metadata structures |
| Mapping-specific policy enums | Runtime execution logic |
| Generation options | Code generation logic |
| Builders for immutable model objects | Spring-specific logic |
| Canonical model validation and invariants | Plugin logic |
| Identity via `DefinitionSetId` | Reflection compatibility models |

### 5.3 Artifact and package root

| Property | Value |
|----------|-------|
| Artifact | `twinmapper-definition-model` |
| Base package | `net.sphuta.twinmapper.definition.model` |
| Compile dependencies | `twinmapper-core` |

### 5.4 Package layout

| Package | Purpose |
|---------|---------|
| `definition.model` | `DefinitionSet`, `DefinitionSetId`, `GenerationOptionsDefinition`, `ImmutabilityStrategy` |
| `definition.model.type` | Sealed `TypeDefinition`, `ObjectTypeDefinition`, `EnumTypeDefinition`, `EnumValueDefinition` |
| `definition.model.field` | `FieldDefinition`, `CollectionDefinition`, `MapDefinition`, `AliasDefinition`, `DefaultValueDefinition`, `DeprecationDefinition`, `ForbiddenFieldDefinition` |
| `definition.model.constraint` | `TypeConstraints`, `FieldConstraints`, `ConditionalConstraintDefinition` |
| `definition.model.mapping` | `ObjectMappingDefinition`, `FieldMappingDefinition`, `MapperMode`, `ProfileDefinition`, `ConverterDefinition`, `NullPolicy`, `UnmappedTargetPolicy` |
| `definition.model.validation` | `DefinitionModelValidator` |

### 5.5 Root package types

#### `DefinitionSet`

| Property | Detail |
|----------|--------|
| Kind | Record + inner Builder |
| Role | Canonical aggregate root. One normalized TwinMapper definition package. |
| Immutability | Fully immutable. All collection fields use `List.copyOf()`. |
| No format-specific constructs | Must not contain YAML, JSON, or BPMN concepts. |

| Field | Type | Nullable | Description |
|-------|------|----------|-------------|
| `id` | `DefinitionSetId` | No | Identity of the definition set |
| `objectTypes` | `List<ObjectTypeDefinition>` | No | Declared object types (immutable) |
| `enumTypes` | `List<EnumTypeDefinition>` | No | Declared enum types (immutable) |
| `objectMappings` | `List<ObjectMappingDefinition>` | No | Declared mappings (immutable) |
| `profiles` | `List<ProfileDefinition>` | No | Reusable mapping profiles (immutable) |
| `converters` | `List<ConverterDefinition>` | No | Declared converters (immutable) |
| `generationOptions` | `GenerationOptionsDefinition` | Yes | Model-level generation options |
| `sourceLocation` | `SourceLocation` | No | Origin of the definition set |

#### `DefinitionSetId`

| Field | Type | Nullable | Description |
|-------|------|----------|-------------|
| `namespace` | `Namespace` | No | Namespace part of identity |
| `name` | `String` | No | Logical name |
| `version` | `String` | No | Version string |

Rules: file path is not identity. Source syntax is not identity. Imports are not part of Phase 1 identity.

#### `GenerationOptionsDefinition`

| Field | Type | Nullable | Default | Description |
|-------|------|----------|---------|-------------|
| `immutableDtos` | `boolean` | No | `true` | Generate immutable records by default |
| `springBeanMode` | `boolean` | No | `false` | Add `@Component` on generated beans |
| `basePackage` | `String` | Yes | `null` | Override base package for generated code |
| `generateValidators` | `boolean` | No | `true` | Generate validator classes |
| `generateMappers` | `boolean` | No | `true` | Generate mapper classes |
| `sourceLocation` | `SourceLocation` | No | — | Origin |

#### `ImmutabilityStrategy`

| Value | Description |
|-------|-------------|
| `RECORD` | Generate immutable Java records |
| `MUTABLE_CLASS` | Generate mutable classes with getters/setters |

### 5.6 Type package

#### `TypeDefinition` (sealed interface)

| Property | Detail |
|----------|--------|
| Kind | Sealed interface |
| Permits | `ObjectTypeDefinition`, `EnumTypeDefinition` |
| Common contract | `name()`, `qualifiedName()`, `sourceLocation()` |

#### `ObjectTypeDefinition`

| Field | Type | Nullable | Description |
|-------|------|----------|-------------|
| `name` | `String` | No | Logical type name |
| `namespace` | `Namespace` | No | Namespace |
| `fields` | `List<FieldDefinition>` | No | Declared fields (immutable, order preserved) |
| `typeConstraints` | `TypeConstraints` | No | Object-level constraints |
| `deprecated` | `boolean` | No | Deprecation flag |
| `sourceLocation` | `SourceLocation` | No | Origin |

Kind: Record + inner Builder.

#### `EnumTypeDefinition`

| Field | Type | Nullable | Description |
|-------|------|----------|-------------|
| `name` | `String` | No | Logical type name |
| `namespace` | `Namespace` | No | Namespace |
| `values` | `List<EnumValueDefinition>` | No | Declared values (immutable) |
| `sourceLocation` | `SourceLocation` | No | Origin |

Kind: Record + inner Builder.

#### `EnumValueDefinition`

| Field | Type | Nullable | Description |
|-------|------|----------|-------------|
| `name` | `String` | No | Value name |
| `code` | `String` | No | Wire code |
| `description` | `String` | Yes | Human-readable description |
| `deprecated` | `boolean` | No | Deprecation flag |
| `deprecationMessage` | `String` | Yes | Deprecation message |

### 5.7 Field package

#### `FieldDefinition`

| Field | Type | Nullable | Description |
|-------|------|----------|-------------|
| `sourceName` | `String` | No | Name in source document |
| `targetName` | `String` | No | Name in generated Java type |
| `typeReference` | `TypeReference` | No | Referenced type |
| `required` | `boolean` | No | Required flag |
| `aliases` | `List<AliasDefinition>` | No | Declared aliases (immutable) |
| `deprecatedAliases` | `List<DeprecationDefinition>` | No | Deprecated aliases (immutable) |
| `defaultValue` | `DefaultValueDefinition` | Yes | Declared default |
| `constraints` | `FieldConstraints` | No | Field-level constraints |
| `collectionDef` | `CollectionDefinition` | Yes | Collection typing if applicable |
| `mapDef` | `MapDefinition` | Yes | Map typing if applicable |
| `sourceLocation` | `SourceLocation` | No | Origin |

Kind: Record + inner Builder.

#### `CollectionDefinition`

| Field | Type | Nullable | Description |
|-------|------|----------|-------------|
| `elementType` | `TypeReference` | No | Type of collection elements |
| `minSize` | `Integer` | Yes | Minimum collection size |
| `maxSize` | `Integer` | Yes | Maximum collection size |

#### `MapDefinition`

| Field | Type | Nullable | Description |
|-------|------|----------|-------------|
| `keyType` | `TypeReference` | No | Type of map keys |
| `valueType` | `TypeReference` | No | Type of map values |

#### `AliasDefinition`

| Field | Type | Nullable | Description |
|-------|------|----------|-------------|
| `alias` | `String` | No | Alias string |
| `canonical` | `String` | No | Canonical field name |

#### `DefaultValueDefinition`

| Field | Type | Nullable | Description |
|-------|------|----------|-------------|
| `rawValue` | `String` | No | Default value as string |
| `valueType` | `String` | No | Expected type of value |

#### `DeprecationDefinition`

| Field | Type | Nullable | Description |
|-------|------|----------|-------------|
| `message` | `String` | No | Deprecation message |
| `replacementField` | `String` | Yes | Suggested replacement |
| `sinceVersion` | `String` | Yes | Version when deprecated |
| `sourceLocation` | `SourceLocation` | No | Origin |

#### `ForbiddenFieldDefinition`

| Field | Type | Nullable | Description |
|-------|------|----------|-------------|
| `fieldName` | `String` | No | Forbidden field name |
| `migrationHint` | `String` | No | Migration guidance |
| `sourceLocation` | `SourceLocation` | No | Origin |

### 5.8 Constraint package

#### `TypeConstraints`

| Field | Type | Nullable | Description |
|-------|------|----------|-------------|
| `oneOfGroups` | `List<List<String>>` | No | One-of field groups (immutable) |
| `mutuallyExclusiveGroups` | `List<List<String>>` | No | Mutually exclusive groups (immutable) |
| `forbiddenFields` | `List<ForbiddenFieldDefinition>` | No | Forbidden fields (immutable) |

#### `FieldConstraints`

| Field | Type | Nullable | Description |
|-------|------|----------|-------------|
| `min` | `Number` | Yes | Minimum numeric value |
| `max` | `Number` | Yes | Maximum numeric value |
| `minLength` | `Integer` | Yes | Minimum string length |
| `maxLength` | `Integer` | Yes | Maximum string length |
| `pattern` | `String` | Yes | Regex pattern |
| `conditionalConstraints` | `List<ConditionalConstraintDefinition>` | No | Conditional constraints (immutable) |

#### `ConditionalConstraintDefinition`

| Field | Type | Nullable | Description |
|-------|------|----------|-------------|
| `discriminatorField` | `String` | No | Field that activates the constraint |
| `activatingValues` | `List<String>` | No | Values that trigger the constraint (immutable) |
| `required` | `boolean` | No | Whether the target field becomes required |
| `sourceLocation` | `SourceLocation` | No | Origin |

### 5.9 Mapping package

#### `ObjectMappingDefinition`

| Field | Type | Nullable | Description |
|-------|------|----------|-------------|
| `name` | `String` | No | Mapping name |
| `sourceType` | `TypeReference` | No | Source type (typed, not raw String) |
| `targetType` | `TypeReference` | No | Target type (typed, not raw String) |
| `mode` | `MapperMode` | No | CREATE, UPDATE, or PATCH |
| `nullPolicy` | `NullPolicy` | Yes | Null handling override |
| `profile` | `String` | Yes | Referenced profile name |
| `inverseOf` | `String` | Yes | Name of mapping to invert |
| `fields` | `List<FieldMappingDefinition>` | No | Field mappings (immutable) |
| `sourceLocation` | `SourceLocation` | No | Origin |

Kind: Record + inner Builder.

#### `FieldMappingDefinition`

| Field | Type | Nullable | Description |
|-------|------|----------|-------------|
| `sourcePath` | `FieldPath` | No | Source field path (typed, not raw String) |
| `targetPath` | `FieldPath` | No | Target field path (typed, not raw String) |
| `converter` | `String` | Yes | Named converter reference |
| `nullPolicy` | `NullPolicy` | Yes | Per-field null policy override |
| `sourceLocation` | `SourceLocation` | No | Origin |

#### `MapperMode`

| Value | Description |
|-------|-------------|
| `CREATE` | Build new target object. Default null policy: SET_NULLS. |
| `UPDATE` | Apply source values onto existing target. Default null policy: IGNORE_NULLS. |
| `PATCH` | Apply only non-null source values. Null policy always IGNORE_NULLS. |

#### `NullPolicy`

| Value | Description |
|-------|-------------|
| `IGNORE_NULLS` | Null source fields skipped; target retains current value |
| `SET_NULLS` | Null source fields set target to null |
| `FAIL_ON_NULL` | Null source field on required mapping throws error |

#### `UnmappedTargetPolicy`

| Value | Description |
|-------|-------------|
| `ERROR` | Unmapped target fields cause build-time error |
| `WARN` | Unmapped target fields produce warning |
| `IGNORE` | Unmapped target fields silently ignored |

#### `ProfileDefinition`

| Field | Type | Nullable | Description |
|-------|------|----------|-------------|
| `name` | `String` | No | Profile name |
| `nullPolicy` | `NullPolicy` | Yes | Default null policy |
| `unmappedTargetPolicy` | `UnmappedTargetPolicy` | Yes | Unmapped target handling |
| `ignoreTargets` | `List<String>` | No | Target fields to exclude (immutable) |
| `converters` | `List<String>` | No | Available converter names (immutable) |
| `sourceLocation` | `SourceLocation` | No | Origin |

#### `ConverterDefinition`

| Field | Type | Nullable | Description |
|-------|------|----------|-------------|
| `name` | `String` | No | Converter name |
| `sourceType` | `String` | No | Source Java type |
| `targetType` | `String` | No | Target Java type |
| `sourceLocation` | `SourceLocation` | No | Origin |

### 5.10 Validation package

#### `DefinitionModelValidator`

| Property | Detail |
|----------|--------|
| Kind | Class |
| Role | Validates canonical model invariants. Reports through `DiagnosticCollector`. |
| Distinction | Readers validate syntax. This validates the normalized model itself. |

#### Validation rules

| Target | Rule | Diagnostic if violated |
|--------|------|----------------------|
| `DefinitionSet` | `id` must be present | ERROR |
| `DefinitionSet` | Namespace must be valid | ERROR |
| `DefinitionSet` | Object type names must be unique | ERROR |
| `DefinitionSet` | Enum type names must be unique | ERROR |
| `DefinitionSet` | No object type name collides with an enum type name | ERROR |
| `DefinitionSet` | Mapping names must be unique | ERROR |
| `ObjectTypeDefinition` | Fields must not be null | ERROR |
| `ObjectTypeDefinition` | Field names must be unique within type | ERROR |
| `ObjectTypeDefinition` | Duplicate aliases must not exist within a field | ERROR |
| `EnumTypeDefinition` | Enum values must be unique | ERROR |
| `EnumTypeDefinition` | Value names must not be blank | ERROR |
| `FieldDefinition` | Field name must be present | ERROR |
| `FieldDefinition` | Type reference must be present | ERROR |
| `FieldDefinition` | Duplicate aliases not allowed | ERROR |
| `FieldDefinition` | Invalid default declaration rejected | ERROR |
| `ObjectMappingDefinition` | Source type must be present | ERROR |
| `ObjectMappingDefinition` | Target type must be present | ERROR |
| `ObjectMappingDefinition` | Field mapping target paths must not conflict | ERROR |
| `ObjectMappingDefinition` | Source and target paths must be valid `FieldPath` | ERROR |

### 5.11 Deferred from Phase 1

| Deferred item | Reason |
|---------------|--------|
| `BindingDefinition` | Binders derived from type definitions; no separate model type needed |
| `ImportDefinition` | Multi-file linking deferred |
| `ReferenceDefinition` | Cross-file references deferred |
| `ExtensionDefinition` | Extension metadata deferred |
| `TypeKind` enum | Sealed interface used instead |
| `VersionDefinition` | Version lives in `DefinitionSetId` only |
| Union / value / collection top-level types | Not in v1 scope |
| Runtime converter contracts | Phase 4+ |
| Executable transformation expressions | Phase 3+ |
| Code generation annotations or hints | Phase 3 |
| Parser-specific node models | Phase 2 |

### 5.12 Definition-model completion criteria

`twinmapper-definition-model` is complete when:

- a `DefinitionSet` can represent the canonical TwinMapper meaning of a definition
- that model is immutable
- that model is format-neutral
- that model has structured identity
- that model can be validated consistently
- downstream modules could consume it without knowing anything about YAML, JSON, or BPMN syntax

---

## 6. Relationship Between the Two Modules

| Module | Defines | Vocabulary |
|--------|---------|------------|
| `twinmapper-core` | How TwinMapper modules speak | Diagnostic language, naming language, path language, source language, SPI language |
| `twinmapper-definition-model` | What TwinMapper definitions mean | Canonical root, types, fields, constraints, mappings |

In one sentence: `twinmapper-core` defines the shared technical vocabulary, and `twinmapper-definition-model` defines the shared semantic vocabulary.

---

## 7. Test Plan

### 7.1 Test principles

| Principle | Detail |
|-----------|--------|
| All tests are pure unit or invariant tests | No I/O, no parsing, no Spring context |
| No later-phase module coupling | Tests depend only on the two Phase 1 modules |
| Coverage intent is locked, not class count | Parameterized tests and nested groups are encouraged |
| `DefinitionReader<R>` has no Phase 1 test | Interface only; no implementation until Phase 2 |

### 7.2 `twinmapper-core` test scenarios

#### `core.diagnostic` — DiagnosticCode

| # | Test scenario | Type | Assertion |
|---|--------------|------|-----------|
| 1 | Every code has non-null `defaultSeverity()` | Unit | Iterate all values, assert non-null |
| 2 | Every code has non-null `category()` | Unit | Iterate all values, assert non-null |
| 3 | No duplicate code names | Unit | Collect names into set, assert size matches `values().length` |
| 4 | All expected categories are covered | Unit | Group by category, verify at least one code per category |
| 5 | Specific expected codes exist | Unit | Assert `MISSING_REQUIRED_FIELD`, `XXE_REJECTED`, etc. are present |

#### `core.diagnostic` — TwinMapperDiagnostic

| # | Test scenario | Type | Assertion |
|---|--------------|------|-----------|
| 1 | Builder produces complete record | Unit | Build with all fields, verify each accessor |
| 2 | Builder rejects null `code` | Unit | Expect `NullPointerException` |
| 3 | Builder rejects null `severity` | Unit | Expect `NullPointerException` |
| 4 | Builder rejects null `fieldPath` | Unit | Expect `NullPointerException` |
| 5 | Builder rejects null `message` | Unit | Expect `NullPointerException` |
| 6 | Builder rejects null `location` | Unit | Expect `NullPointerException` |
| 7 | `isError()` true when severity is ERROR | Unit | Build with ERROR, assert `isError()` |
| 8 | `isWarning()` true when severity is WARNING | Unit | Build with WARNING, assert `isWarning()` |
| 9 | `category()` delegates to code | Unit | Assert `diagnostic.category() == code.category()` |
| 10 | Nullable fields accept null | Unit | Build with null `targetType`, `ruleViolated`, etc. |
| 11 | Record equality | Unit | Two identically built diagnostics are equal |
| 12 | Record hashCode consistency | Unit | Same inputs produce same hashCode |

#### `core.diagnostic` — DiagnosticCollector / DefaultDiagnosticCollector

| # | Test scenario | Type | Assertion |
|---|--------------|------|-----------|
| 1 | `add()` makes item retrievable via `all()` | Unit | Add one, assert `all()` contains it |
| 2 | `hasErrors()` false when empty | Unit | New collector, assert false |
| 3 | `hasErrors()` true after adding ERROR | Unit | Add ERROR diagnostic, assert true |
| 4 | `hasWarnings()` true after adding WARNING | Unit | Add WARNING diagnostic, assert true |
| 5 | `hasErrors()` false when only warnings | Unit | Add only WARNING items, assert `hasErrors()` false |
| 6 | `errors()` returns only ERROR items | Unit | Add mixed, assert `errors()` filtered |
| 7 | `warnings()` returns only WARNING items | Unit | Add mixed, assert `warnings()` filtered |
| 8 | `addAll()` adds multiple | Unit | Add collection, verify all present |
| 9 | Returned lists are immutable snapshots | Unit | Get list, add to collector, verify original list unchanged |

#### `core.diagnostic` — Exception hierarchy

| # | Test scenario | Type | Assertion |
|---|--------------|------|-----------|
| 1 | `TwinMapperException` carries diagnostics list | Unit | Construct with list, verify `diagnostics()` |
| 2 | `TwinMapperException` with empty list | Unit | Construct, verify empty list |
| 3 | `TwinMapperException` with single diagnostic | Unit | Construct, verify size 1 |
| 4 | `diagnostics()` returns immutable list | Unit | Attempt modification, expect failure |
| 5 | `DefinitionException` instanceof `TwinMapperException` | Unit | Assert instanceof |
| 6 | `BindingException` instanceof `TwinMapperException` | Unit | Assert instanceof |
| 7 | `MappingException` instanceof `TwinMapperException` | Unit | Assert instanceof |
| 8 | `ValidationException` instanceof `TwinMapperException` | Unit | Assert instanceof |
| 9 | `CodegenException` instanceof `TwinMapperException` | Unit | Assert instanceof |
| 10 | `TwinMapperSecurityException` instanceof `TwinMapperException` | Unit | Assert instanceof |
| 11 | Each subclass is not instanceof any sibling | Unit | Assert `DefinitionException` is not `BindingException`, etc. |
| 12 | Message propagation from single diagnostic | Unit | Construct from diagnostic, verify `getMessage()` |

#### `core.naming` — Namespace

| # | Test scenario | Type | Assertion |
|---|--------------|------|-----------|
| 1 | Valid construction | Unit | Create, verify `value()` |
| 2 | `ROOT` sentinel exists | Unit | Assert `Namespace.ROOT` non-null |
| 3 | Blank string rejected | Unit | Expect exception |
| 4 | Null rejected | Unit | Expect exception |
| 5 | Equality: same value | Unit | Two with same string are equal |
| 6 | HashCode: same value | Unit | Same string produces same hash |
| 7 | Different values not equal | Unit | Different strings not equal |
| 8 | `toString()` returns value | Unit | Assert format |

#### `core.naming` — QualifiedName

| # | Test scenario | Type | Assertion |
|---|--------------|------|-----------|
| 1 | Valid construction | Unit | Create, verify namespace + localName |
| 2 | Null namespace rejected | Unit | Expect exception |
| 3 | Null localName rejected | Unit | Expect exception |
| 4 | Blank localName rejected | Unit | Expect exception |
| 5 | Equality: same namespace + name | Unit | Assert equal |
| 6 | Same name, different namespace not equal | Unit | Assert not equal |
| 7 | HashCode consistency | Unit | Same inputs, same hash |
| 8 | `toString()` format | Unit | Assert expected format |

#### `core.naming` — TypeReference

| # | Test scenario | Type | Assertion |
|---|--------------|------|-----------|
| 1 | Valid construction | Unit | Create, verify fields |
| 2 | Null qualifiedName rejected | Unit | Expect exception |
| 3 | `nullable` flag: true and false | Unit | Verify flag value |
| 4 | Equality: same name + same flag | Unit | Assert equal |
| 5 | Same name, different nullable not equal | Unit | Assert not equal |
| 6 | HashCode consistency | Unit | Same inputs, same hash |

#### `core.path` — FieldPath

| # | Test scenario | Type | Assertion |
|---|--------------|------|-----------|
| 1 | Parse simple path `"customer"` | Unit | Verify single segment |
| 2 | Parse dotted path `"customer.address.city"` | Unit | Verify three segments |
| 3 | Parse indexed path `"items[].amount"` | Unit | Verify segments with index marker |
| 4 | `segments()` returns correct list | Unit | Match expected segments |
| 5 | `depth()` matches segment count | Unit | Assert depth equals `segments().size()` |
| 6 | `append(segment)` produces longer path | Unit | Append, verify new depth |
| 7 | `parent()` removes last segment | Unit | Parent of `a.b.c` is `a.b` |
| 8 | `parent()` on single-segment path | Unit | Define behavior: exception or empty |
| 9 | `isRoot()` on single segment | Unit | Assert true |
| 10 | Equality: same path string | Unit | Assert equal |
| 11 | Immutability: append returns new instance | Unit | Original unchanged after append |
| 12 | Empty string rejected | Unit | Expect exception |
| 13 | `toString()` reproduces dotted path | Unit | Assert format |

#### `core.path` — PathSegment

| # | Test scenario | Type | Assertion |
|---|--------------|------|-----------|
| 1 | Construction from name | Unit | Verify `name()` |
| 2 | Indexed segment | Unit | Verify index flag |
| 3 | Equality | Unit | Same name equals |

#### `core.policy` — CompatibilityMode

| # | Test scenario | Type | Assertion |
|---|--------------|------|-----------|
| 1 | Has STRICT, COMPATIBLE, LENIENT | Unit | Assert all three present |
| 2 | Exactly three values | Unit | Assert `values().length == 3` |

#### `core.source` — SourceLocation

| # | Test scenario | Type | Assertion |
|---|--------------|------|-----------|
| 1 | `UNKNOWN` sentinel is non-null | Unit | Assert non-null |
| 2 | `UNKNOWN` has expected defaults | Unit | Verify null/zero fields |
| 3 | `of(sourceId, line, column)` factory | Unit | Verify all fields |
| 4 | `ofFile(sourceId)` factory | Unit | Verify sourceId, line=0, column=0 |
| 5 | `hasPosition()` true when line > 0 | Unit | Assert true |
| 6 | `hasPosition()` false when line = 0 | Unit | Assert false |
| 7 | `hasSourceFile()` true when sourceId present | Unit | Assert true |
| 8 | `hasSourceFile()` false on UNKNOWN | Unit | Assert false |
| 9 | `toString()` full: `"file.yaml:10:5"` | Unit | Assert format |
| 10 | `toString()` file-only: `"file.yaml"` | Unit | Assert format |
| 11 | `toString()` UNKNOWN: `"(unknown)"` | Unit | Assert format |
| 12 | Nullable `fragmentId` accepted | Unit | Build with null, no exception |
| 13 | Equality and hashCode | Unit | Same inputs equal, same hash |

#### `core.spi` — DefinitionSource

| # | Test scenario | Type | Assertion |
|---|--------------|------|-----------|
| 1 | Construction with location and format hint | Unit | Verify accessors |
| 2 | `formatHint()` returns expected `DocumentFormat` | Unit | Assert match |
| 3 | `location()` returns path string | Unit | Assert match |
| 4 | Stream supplier invocation | Unit | Use `ByteArrayInputStream`, verify readable |
| 5 | `DocumentFormat` has YAML, JSON, BPMN, UNKNOWN | Unit | Assert all four present |

#### `core.util` — FieldNameUtils

| # | Test scenario | Type | Assertion |
|---|--------------|------|-----------|
| 1 | camelCase → kebab-case | Unit | `"fieldName"` → `"field-name"` |
| 2 | kebab-case → camelCase | Unit | `"field-name"` → `"fieldName"` |
| 3 | camelCase → snake_case | Unit | `"fieldName"` → `"field_name"` |
| 4 | snake_case → camelCase | Unit | `"field_name"` → `"fieldName"` |
| 5 | Round-trip: camel → kebab → camel | Unit | Original restored |
| 6 | Round-trip: camel → snake → camel | Unit | Original restored |
| 7 | Single word unchanged | Unit | `"name"` → `"name"` |
| 8 | Already correct format | Unit | No-op |
| 9 | Consecutive uppercase `"XMLParser"` | Unit | Define and verify behavior |
| 10 | Empty string | Unit | Define behavior |
| 11 | Leading/trailing separators | Unit | Define behavior |

#### `core.util` — Preconditions

| # | Test scenario | Type | Assertion |
|---|--------------|------|-----------|
| 1 | `requireNonBlank` passes on valid | Unit | No exception |
| 2 | `requireNonBlank` throws on null | Unit | Expect exception |
| 3 | `requireNonBlank` throws on empty | Unit | Expect exception |
| 4 | `requireNonBlank` throws on whitespace-only | Unit | Expect exception |
| 5 | `requirePositive` passes on positive | Unit | No exception |
| 6 | `requirePositive` throws on zero | Unit | Expect exception |
| 7 | `requirePositive` throws on negative | Unit | Expect exception |
| 8 | `requireNonEmpty` passes on non-empty collection | Unit | No exception |
| 9 | `requireNonEmpty` throws on empty collection | Unit | Expect exception |
| 10 | `requireNonEmpty` throws on null | Unit | Expect exception |

#### `core.util` — TwinMapperConstants

| # | Test scenario | Type | Assertion |
|---|--------------|------|-----------|
| 1 | Constants are non-null and non-blank | Unit | Assert each constant |
| 2 | Values are as expected | Unit | Assert specific values |

### 7.3 `twinmapper-definition-model` test scenarios

#### Root — DefinitionSet

| # | Test scenario | Type | Assertion |
|---|--------------|------|-----------|
| 1 | Build via builder with all fields | Unit | Verify all accessors |
| 2 | Builder defaults: empty lists, null generationOptions | Unit | Build minimal, verify defaults |
| 3 | `id()` returns `DefinitionSetId` | Unit | Assert non-null, correct identity |
| 4 | `objectTypes()` returns immutable list | Unit | Attempt add, expect exception |
| 5 | `enumTypes()` returns immutable list | Unit | Attempt add, expect exception |
| 6 | `objectMappings()` returns immutable list | Unit | Attempt add, expect exception |
| 7 | `profiles()` returns immutable list | Unit | Attempt add, expect exception |
| 8 | `converters()` returns immutable list | Unit | Attempt add, expect exception |
| 9 | `sourceLocation()` accessible | Unit | Assert non-null |
| 10 | Equality: two identically built sets | Unit | Assert equal |

#### Root — DefinitionSetId

| # | Test scenario | Type | Assertion |
|---|--------------|------|-----------|
| 1 | Construction from namespace + name + version | Unit | Verify accessors |
| 2 | Null namespace rejected | Unit | Expect exception |
| 3 | Null/blank name rejected | Unit | Expect exception |
| 4 | Null/blank version rejected | Unit | Expect exception |
| 5 | Equality: same namespace + name + version | Unit | Assert equal |
| 6 | Different version means not equal | Unit | Assert not equal |
| 7 | HashCode consistency | Unit | Same inputs, same hash |
| 8 | `toString()` format | Unit | Assert expected format |

#### Root — GenerationOptionsDefinition

| # | Test scenario | Type | Assertion |
|---|--------------|------|-----------|
| 1 | Default values: immutableDtos=true, springBeanMode=false | Unit | Build defaults, verify |
| 2 | Override values | Unit | Set non-defaults, verify |
| 3 | Nullable `basePackage` accepted | Unit | Build with null, no exception |

#### Type — ObjectTypeDefinition

| # | Test scenario | Type | Assertion |
|---|--------------|------|-----------|
| 1 | Build with name, namespace, fields | Unit | Verify accessors |
| 2 | `fields()` returns immutable list | Unit | Attempt add, expect exception |
| 3 | `typeConstraints()` accessible | Unit | Assert non-null |
| 4 | `deprecated` flag | Unit | Verify true/false |
| 5 | `sourceLocation()` accessible | Unit | Assert non-null |
| 6 | `qualifiedName()` derives from namespace + name | Unit | Verify derivation |
| 7 | Equality | Unit | Same inputs equal |

#### Type — EnumTypeDefinition

| # | Test scenario | Type | Assertion |
|---|--------------|------|-----------|
| 1 | Build with name, namespace, values | Unit | Verify accessors |
| 2 | `values()` returns immutable list | Unit | Attempt add, expect exception |
| 3 | `sourceLocation()` accessible | Unit | Assert non-null |
| 4 | Equality | Unit | Same inputs equal |

#### Type — EnumValueDefinition

| # | Test scenario | Type | Assertion |
|---|--------------|------|-----------|
| 1 | Construction with name, code | Unit | Verify accessors |
| 2 | Nullable `description` | Unit | Build with null, no exception |
| 3 | `deprecated` flag and `deprecationMessage` | Unit | Verify both |

#### Type — TypeDefinition sealed hierarchy

| # | Test scenario | Type | Assertion |
|---|--------------|------|-----------|
| 1 | `ObjectTypeDefinition` is permitted subtype | Unit | `getPermittedSubclasses()` includes it |
| 2 | `EnumTypeDefinition` is permitted subtype | Unit | `getPermittedSubclasses()` includes it |
| 3 | Only two permitted subtypes | Unit | Assert `getPermittedSubclasses().length == 2` |
| 4 | Pattern matching switch compiles exhaustively | Compile | Write switch over both, verify no default needed |

#### Field — FieldDefinition

| # | Test scenario | Type | Assertion |
|---|--------------|------|-----------|
| 1 | Build with all fields | Unit | Verify all accessors |
| 2 | `sourceName()` and `targetName()` | Unit | Assert values |
| 3 | `required` flag | Unit | Verify true/false |
| 4 | `aliases()` returns immutable list | Unit | Attempt add, expect exception |
| 5 | `deprecatedAliases()` returns immutable list | Unit | Attempt add, expect exception |
| 6 | Nullable `defaultValue` | Unit | Build with null, no exception |
| 7 | Nullable `collectionDef` | Unit | Build with null, no exception |
| 8 | Nullable `mapDef` | Unit | Build with null, no exception |
| 9 | `constraints()` accessible | Unit | Assert non-null |
| 10 | Equality | Unit | Same inputs equal |

#### Field — CollectionDefinition

| # | Test scenario | Type | Assertion |
|---|--------------|------|-----------|
| 1 | `elementType` accessible | Unit | Assert non-null |
| 2 | Nullable `minSize` and `maxSize` | Unit | Build with null, no exception |

#### Field — DeprecationDefinition

| # | Test scenario | Type | Assertion |
|---|--------------|------|-----------|
| 1 | `message` accessible | Unit | Assert value |
| 2 | Nullable `replacementField` and `sinceVersion` | Unit | Build with null, no exception |
| 3 | `sourceLocation()` accessible | Unit | Assert non-null |

#### Field — ForbiddenFieldDefinition

| # | Test scenario | Type | Assertion |
|---|--------------|------|-----------|
| 1 | `fieldName` and `migrationHint` | Unit | Assert values |
| 2 | `sourceLocation()` accessible | Unit | Assert non-null |

#### Constraint — TypeConstraints

| # | Test scenario | Type | Assertion |
|---|--------------|------|-----------|
| 1 | Empty defaults: empty lists | Unit | Build defaults, verify empty |
| 2 | `oneOfGroups` with populated groups | Unit | Verify contents |
| 3 | `mutuallyExclusiveGroups` with populated groups | Unit | Verify contents |
| 4 | `forbiddenFields` list | Unit | Verify contents |
| 5 | Immutability of all lists | Unit | Attempt modification, expect exception |

#### Constraint — FieldConstraints

| # | Test scenario | Type | Assertion |
|---|--------------|------|-----------|
| 1 | All nullable fields accept null | Unit | Build with all null, no exception |
| 2 | `min` and `max` population | Unit | Set values, verify |
| 3 | `minLength` and `maxLength` population | Unit | Set values, verify |
| 4 | `pattern` population | Unit | Set value, verify |
| 5 | `conditionalConstraints` list | Unit | Verify contents |
| 6 | Immutability | Unit | Attempt modification, expect exception |

#### Constraint — ConditionalConstraintDefinition

| # | Test scenario | Type | Assertion |
|---|--------------|------|-----------|
| 1 | `discriminatorField` accessible | Unit | Assert value |
| 2 | `activatingValues` list | Unit | Verify contents |
| 3 | `required` flag | Unit | Verify true/false |
| 4 | `sourceLocation()` accessible | Unit | Assert non-null |
| 5 | Immutability of `activatingValues` | Unit | Attempt modification, expect exception |

#### Mapping — ObjectMappingDefinition

| # | Test scenario | Type | Assertion |
|---|--------------|------|-----------|
| 1 | Build with all fields | Unit | Verify all accessors |
| 2 | `sourceType` is `TypeReference` (not String) | Unit | Assert type and value |
| 3 | `targetType` is `TypeReference` (not String) | Unit | Assert type and value |
| 4 | `mode()` returns `MapperMode` | Unit | Assert expected mode |
| 5 | Nullable `nullPolicy`, `profile`, `inverseOf` | Unit | Build with null, no exception |
| 6 | `fields()` returns immutable list | Unit | Attempt add, expect exception |
| 7 | `sourceLocation()` accessible | Unit | Assert non-null |

#### Mapping — FieldMappingDefinition

| # | Test scenario | Type | Assertion |
|---|--------------|------|-----------|
| 1 | `sourcePath` is `FieldPath` (not String) | Unit | Assert type and value |
| 2 | `targetPath` is `FieldPath` (not String) | Unit | Assert type and value |
| 3 | Nullable `converter` | Unit | Build with null, no exception |
| 4 | Nullable `nullPolicy` override | Unit | Build with null, no exception |
| 5 | `sourceLocation()` accessible | Unit | Assert non-null |

#### Mapping — MapperMode

| # | Test scenario | Type | Assertion |
|---|--------------|------|-----------|
| 1 | Has CREATE, UPDATE, PATCH | Unit | Assert all three present |
| 2 | Exactly three values | Unit | Assert `values().length == 3` |

#### Mapping — NullPolicy

| # | Test scenario | Type | Assertion |
|---|--------------|------|-----------|
| 1 | Has IGNORE_NULLS, SET_NULLS, FAIL_ON_NULL | Unit | Assert all three present |
| 2 | Exactly three values | Unit | Assert `values().length == 3` |

#### Mapping — UnmappedTargetPolicy

| # | Test scenario | Type | Assertion |
|---|--------------|------|-----------|
| 1 | Has ERROR, WARN, IGNORE | Unit | Assert all three present |
| 2 | Exactly three values | Unit | Assert `values().length == 3` |

#### Mapping — ProfileDefinition

| # | Test scenario | Type | Assertion |
|---|--------------|------|-----------|
| 1 | `name` accessible | Unit | Assert value |
| 2 | Nullable `nullPolicy` and `unmappedTargetPolicy` | Unit | Build with null, no exception |
| 3 | `ignoreTargets` immutable | Unit | Attempt modification, expect exception |
| 4 | `converters` immutable | Unit | Attempt modification, expect exception |

#### Mapping — ConverterDefinition

| # | Test scenario | Type | Assertion |
|---|--------------|------|-----------|
| 1 | `name`, `sourceType`, `targetType` | Unit | Assert values |
| 2 | `sourceLocation()` accessible | Unit | Assert non-null |

#### Validation — DefinitionModelValidator

| # | Test scenario | Type | Assertion |
|---|--------------|------|-----------|
| 1 | Valid `DefinitionSet` passes with no errors | Invariant | Build correct model, validate, assert no errors |
| 2 | Duplicate object type names rejected | Invariant | Two types same name, assert ERROR diagnostic |
| 3 | Duplicate enum type names rejected | Invariant | Two enums same name, assert ERROR diagnostic |
| 4 | Object type name collides with enum type name | Invariant | Same name on both, assert ERROR diagnostic |
| 5 | Duplicate field names within one object type | Invariant | Two fields same sourceName, assert ERROR diagnostic |
| 6 | Duplicate aliases within one field | Invariant | Two identical aliases, assert ERROR diagnostic |
| 7 | Duplicate mapping names | Invariant | Two mappings same name, assert ERROR diagnostic |
| 8 | Missing `DefinitionSetId` | Invariant | Build with null id, assert ERROR diagnostic |
| 9 | Invalid namespace on id | Invariant | Build with blank namespace, assert ERROR diagnostic |
| 10 | Blank type name | Invariant | Object type with blank name, assert ERROR diagnostic |
| 11 | Blank field name | Invariant | Field with blank sourceName, assert ERROR diagnostic |
| 12 | Missing type reference on field | Invariant | Field with null typeReference, assert ERROR diagnostic |
| 13 | Blank enum value name | Invariant | Enum value with blank name, assert ERROR diagnostic |
| 14 | Duplicate enum value names | Invariant | Two values same name, assert ERROR diagnostic |
| 15 | Missing source type on mapping | Invariant | Mapping with null sourceType, assert ERROR diagnostic |
| 16 | Missing target type on mapping | Invariant | Mapping with null targetType, assert ERROR diagnostic |
| 17 | Duplicate field mapping target paths | Invariant | Two fields same targetPath, assert ERROR diagnostic |
| 18 | Valid complex model passes cleanly | Invariant | Multiple types, enums, mappings — all valid, no diagnostics |

### 7.4 Test summary

| Module | Package | Scenarios | Type |
|--------|---------|-----------|------|
| core | diagnostic — DiagnosticCode | 5 | Pure unit |
| core | diagnostic — TwinMapperDiagnostic | 12 | Pure unit |
| core | diagnostic — DiagnosticCollector | 9 | Pure unit |
| core | diagnostic — Exception hierarchy | 12 | Pure unit |
| core | naming — Namespace | 8 | Pure unit |
| core | naming — QualifiedName | 8 | Pure unit |
| core | naming — TypeReference | 6 | Pure unit |
| core | path — FieldPath | 13 | Pure unit |
| core | path — PathSegment | 3 | Pure unit |
| core | policy — CompatibilityMode | 2 | Pure unit |
| core | source — SourceLocation | 13 | Pure unit |
| core | spi — DefinitionSource + DocumentFormat | 5 | Pure unit |
| core | util — FieldNameUtils | 11 | Pure unit |
| core | util — Preconditions | 10 | Pure unit |
| core | util — TwinMapperConstants | 2 | Pure unit |
| definition-model | root — DefinitionSet | 10 | Pure unit |
| definition-model | root — DefinitionSetId | 8 | Pure unit |
| definition-model | root — GenerationOptionsDefinition | 3 | Pure unit |
| definition-model | type — ObjectTypeDefinition | 7 | Pure unit |
| definition-model | type — EnumTypeDefinition | 4 | Pure unit |
| definition-model | type — EnumValueDefinition | 3 | Pure unit |
| definition-model | type — Sealed hierarchy | 4 | Pure unit / compile |
| definition-model | field — FieldDefinition | 10 | Pure unit |
| definition-model | field — CollectionDefinition | 2 | Pure unit |
| definition-model | field — DeprecationDefinition | 3 | Pure unit |
| definition-model | field — ForbiddenFieldDefinition | 2 | Pure unit |
| definition-model | constraint — TypeConstraints | 5 | Pure unit |
| definition-model | constraint — FieldConstraints | 6 | Pure unit |
| definition-model | constraint — ConditionalConstraintDefinition | 5 | Pure unit |
| definition-model | mapping — ObjectMappingDefinition | 7 | Pure unit |
| definition-model | mapping — FieldMappingDefinition | 5 | Pure unit |
| definition-model | mapping — MapperMode | 2 | Pure unit |
| definition-model | mapping — NullPolicy | 2 | Pure unit |
| definition-model | mapping — UnmappedTargetPolicy | 2 | Pure unit |
| definition-model | mapping — ProfileDefinition | 4 | Pure unit |
| definition-model | mapping — ConverterDefinition | 2 | Pure unit |
| definition-model | validation — DefinitionModelValidator | 18 | Invariant |
| **Total** | | **~232** | |

---

## 8. What Phase 1 Makes Possible

Once these two modules are complete, later phases become straightforward because they no longer need to invent foundation concepts.

Future modules will be able to:

- report diagnostics through one shared structure
- trace issues back through one source-location model
- identify definitions through one naming and identity scheme
- produce or consume one canonical `DefinitionSet`
- rely on one place for structural invariants

That is the actual value of Phase 1.
