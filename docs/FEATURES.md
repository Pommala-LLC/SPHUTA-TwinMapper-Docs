# TwinMapper — Feature Reference

This document covers every feature in TwinMapper organized by category. Features are classified as Core, Additional, or Optional. Core and Additional features are present by default. Optional features require explicit property-based activation.

---

## Feature Classification

| Classification | Meaning |
|---|---|
| Core | Baseline product capability, present by default |
| Additional | Required completeness item for production-ready Spring-first baseline |
| Optional | Explicit opt-in, property-gated, default false |

---

## Document Engine Features

### Core

**Compile-time DTO generation**
DTOs are generated from fixed YAML, JSON, or BPMN definitions at build time. Never generated from runtime payloads. Never inferred from live input samples.

**Enum generation**
Enums generated from declared definition values. Strict enum validation. Enum-to-code and code-to-enum mapping. String-to-enum and enum-to-string mapping. Deprecated enum alias support when configured.

**Metadata generation**
Field descriptors, type descriptors, alias descriptors, default value metadata, required/optional metadata, deprecated/forbidden metadata, version metadata, source-to-target metadata, generation metadata.

**Binder generation**
Format-aware binder classes generated at compile time. Separate binder for JSON, YAML, and BPMN. Constructor and record-based population. Nested object binding. Collection and map binding. Alias-aware field reading. Default value population. Required-field enforcement. Strict unknown-field rejection.

**Validator generation**
Schema-constraint validator classes generated from definition rules. Required, default, enum, forbidden, deprecated, conditional, one-of, mutually exclusive, min, max, minLength, maxLength, regex/pattern, unknown-field, and collection/map type checks.

**Registry generation**
Binder registry, object mapper registry, metadata registry, converter registry, and type lookup registry — all generated and populated at compile time.

**Runtime YAML binding**
Parses YAML documents using SnakeYAML into `NodeCursor`, dispatches to generated binders, populates generated DTOs.

**Runtime JSON binding**
Parses JSON documents using Jackson into `NodeCursor`, dispatches to generated binders, populates generated DTOs.

**Runtime BPMN binding**
Parses BPMN XML using JDK StAX with XXE disabled into `BpmnIntermediateModel`, adapts to `NodeCursor`, dispatches to generated binders, populates generated DTOs.

**NodeCursor abstraction**
Format-neutral document traversal. Supports field existence checks, required and optional scalar reads, enum reads, nested object access, list children, map children, path tracking, and type mismatch reporting.

---

### Additional

**JSON naming consistency**
`@JsonNaming`, `@JsonAlias`, `PropertyNamingStrategies` on generated DTOs. `Jackson2ObjectMapperBuilderCustomizer` registered by starter for alignment between TwinMapper-generated DTOs and Spring Boot's auto-configured `ObjectMapper`.

**`MappingJackson2HttpMessageConverter` integration**
When TwinMapper DTOs are used as Spring MVC REST request or response bodies, the standard Spring MVC converter is the integration point.

**Secure SnakeYAML construction**
`new Yaml(new SafeConstructor(new LoaderOptions()))` mandatory. Default SnakeYAML constructor allows arbitrary Java object deserialization and is prohibited.

**`spring.config.import` support**
`ConfigDataLoader` and `ConfigDataLocationResolver` for Spring Boot 2.4+ allow TwinMapper definitions to be imported via `spring.config.import=twinmapper:classpath:/definitions/`.

**XXE-safe BPMN parsing**
`XMLInputFactory.IS_SUPPORTING_EXTERNAL_ENTITIES` set to `false` and `XMLInputFactory.SUPPORT_DTD` set to `false` — both required, omitting either is a security vulnerability.

**Mandatory `ResourceLoader` for BPMN**
All BPMN classpath file resolution uses Spring `ResourceLoader` — required for correctness in OSGi, GraalVM native image, and application server deployments.

---

### Optional

**Bounded runtime discovery helpers**
Adapter-specific, explicit opt-in only, produces traceable artifacts, never replaces fixed-definition mode.
Property: `twinmapper.binding.bounded-discovery.enabled`

**Alias/fallback compatibility resolution**
Resolves deprecated aliases, legacy field names, and renamed fields during binding.
Property: `twinmapper.compatibility-resolution.enabled`

---

## Object Engine Features

### Core

**Create mapping**
Generates a new target object by reading all declared field mappings from the source object.

**Update mapping**
Applies source field values onto an existing target object. Preserves identity and audit fields unless explicitly mapped. Default null policy: `IGNORE_NULLS`.

**Patch mapping**
Applies only non-null source field values onto an existing target object. Mandatory null policy: `IGNORE_NULLS`.

**Inverse mapping**
Generates a reverse mapper from the same mapping definition. Build-time failure if inverse generation is ambiguous or unsafe.

**Nested mapping**
Maps nested source property paths to nested target property paths and vice versa.

**Flattening**
Maps nested source properties into flat target fields. Example: `address.city` → `city`.

**Expansion**
Maps flat source fields into a nested target object. Example: `city` + `zip` → `address`.

**Collection mapping**
`List<S>` → `List<T>`, `Set<S>` → `Set<T>`. Element mapper dispatched per item.

**Map mapping**
`Map<K1,V1>` → `Map<K2,V2>`. Key converter and value mapper applied per entry.

**Converter-assisted mapping**
Field-level and type-level converter support. Value object converters. Enum/code converters. Scalar conversions when declared. Registered via definition config, programmatic API, or optional annotation.

**Null-policy-aware mapping**
`IGNORE_NULLS`, `SET_NULLS`, `FAIL_ON_NULL`. Configurable per mapping, per profile, or per field.

**Reusable mapping profiles**
Shared null/update policies, unmapped target policy, shared converters, shared ignore rules, and common mapping defaults. Reduces repetition across enterprise-scale mapping configurations.

**Supported mapping flows**
All of the following are first-class supported:

| Source | Target |
|---|---|
| DTO | DTO |
| Entity | Domain |
| Domain | Entity |
| Domain | DTO |
| DTO | Domain |
| Entity | DTO |
| DTO | Entity |
| Entity | Projection |
| Projection | DTO |
| Request / Command | Domain |
| Domain | Response / View / Event |
| Patch DTO | Entity |
| Patch DTO | Domain |

---

### Additional

**`ConversionService` integration**
Spring `ConversionService`, `GenericConversionService`, `FormattingConversionService`, `DefaultConversionService` as the converter backend in Spring applications. `Converter`, `ConverterFactory`, `GenericConverter`, `ConditionalGenericConverter` as the SPI contracts for custom converters.

**`BeanWrapper` usage**
Used in UPDATE and PATCH mode when mutating an existing Spring-managed bean target. Handles nested property paths, type conversion, and JPA-loaded proxy targets transparently.

**`DirectFieldAccessor` usage**
Used when the target has no setters — for example, value objects with final fields.

**Proxy-safe reflection**
`AopUtils.getTargetClass()`, `AopProxyUtils.ultimateTargetClass()`, `ProxyUtils.getUserClass()` mandatory for any reflection-based operation on Spring-managed beans. Omitting these causes silent failures on proxied targets.

**Generated bean disambiguation**
`@Qualifier`, `@Primary`, `@Conditional` optionally emitted on generated mapper beans for Spring applications with multiple mappers targeting the same type pair.

---

### Optional

**Controlled convention-based mapping mode**
Enabled only when explicitly configured. Source and target types must be strongly compatible. Ambiguity causes build-time or runtime failure — never silent resolution.
Property: `twinmapper.objectmap.convention-mapping.enabled`

**Reflection-based compatibility helpers**
Limited reflection-assisted mapping for migration and legacy onboarding. Never the default runtime path. Proxy-aware via `AopUtils`/`AopProxyUtils`/`ProxyUtils`.
Property: `twinmapper.objectmap.reflection-compat.enabled`

---

## Validation Features

### Core

**Required field validation** — Missing required fields produce `BindingError` with full field path.

**Optional field handling** — Absent optional fields trigger default application when a default is declared.

**Default value application** — Static and expression-derived defaults applied at bind time.

**Enum validation** — Unknown enum values produce `BindingError` listing valid values.

**Forbidden field detection** — Forbidden or legacy field present produces `BindingError` with migration hint.

**Deprecated field handling** — Warning in COMPATIBLE mode, error in STRICT mode.

**Conditional required rules** — Requirement active only when a discriminator field matches a declared value.

**One-of rules** — Exactly one of a declared group of fields must be present.

**Mutually exclusive rules** — Declared fields must not appear together.

**Min / max validation** — Numeric range constraints on field values.

**MinLength / maxLength validation** — String length constraints on field values.

**Regex / pattern validation** — Pattern constraints on string field values.

**Unknown field validation** — Unknown fields rejected in STRICT mode, warned in COMPATIBLE.

**Collection and map type validation** — Element and entry type checks on collection and map fields.

**Path-aware aggregated reporting** — All validation errors collected into `ValidationReport` with full field paths, file names, and line/column where available.

---

### Additional

**Spring validation chain integration**
Generated validators implementable as Spring `Validator`. `ConstraintValidatorFactory` for Spring-injected custom constraint validators. `HandlerMethodArgumentResolver` for Spring MVC method parameter binding. `ResponseEntityExceptionHandler` alignment for clean translation of `MethodArgumentNotValidException` and `ConstraintViolationException`.

---

## Strictness Modes

### STRICT (default)
Rejects unknown fields, invalid enum values, missing required fields, ambiguous mappings, incompatible types without declared converter, unsafe inverse generation.

### COMPATIBLE
Allows deprecated aliases, certain compatibility warnings, controlled backward-compatible interpretation.

### LENIENT
Migration tooling only. Not the default. Not for production use.

---

## Code Generation Features

### Core

**DSL strategy**
TwinMapper-native YAML DSL. Not JSON Schema. Purpose-built for code generation and object mapping.

**DSL constructs supported:**
- `definitionSet` (name, version, package, strict mode)
- Object types and enum types
- Fields with type, required, optional, default, targetName, aliases, deprecatedAliases, description, constraints
- Constraints: min, max, minLength, maxLength, pattern, oneOf, mutuallyExclusive
- Forbidden and deprecated field declarations
- Object mappings with name, sourceType, targetType, mode, profile, inverseOf, field mappings
- Profiles with nullPolicy, unmappedTargetPolicy, converters, ignoreTargets
- Converters with name, sourceType, targetType
- Generation options for DTO style, builder support, binder/validator/mapper generation

**Build-time pipeline:**
1. Scan definition directories
2. Parse definitions into internal meta-model
3. Validate definitions
4. Normalize into format-neutral meta-model
5. Generate Java sources into `build/generated/sources/twinmapper`
6. Compile generated and handwritten code together

---

### Additional

**`basePackage` configuration**
Aligns generated code package structure with the consuming application's package to avoid Java module system split-package issues.

**Generated source registration**
Gradle: `sourceSets.main.java.srcDir(generatedDir)`. Maven: `project.addCompileSourceRoot(generatedDir)`. Both register the generated source root for IDE and compilation.

---

## Spring Boot Integration Features

### Core

**`TwinMapperProperties`**
`@ConfigurationProperties(prefix = "twinmapper")` covering strictness mode, definition locations, generation options, optional feature flags, and compatibility toggles. Foundational — every `@ConditionalOnProperty` in the starter depends on it.

**Auto-configuration**
`@AutoConfiguration`, `@ConditionalOnClass`, `@ConditionalOnMissingBean`, `@ConditionalOnProperty`, `AutoConfiguration.imports` for Spring Boot 3.x/4.x.

---

### Additional

**`@EnableConfigurationProperties(TwinMapperProperties.class)`**
Declared in auto-configuration class. Required for property binding to work.

**`spring-configuration-metadata.json`**
Under `META-INF/` in the starter. Provides IDE auto-completion for all `twinmapper.*` properties with descriptions, types, and default values.

**Named application events**
- `TwinMapperBindingSuccessEvent`
- `TwinMapperBindingFailureEvent`
- `TwinMapperMappingSuccessEvent`
- `TwinMapperMappingFailureEvent`

Published via `ApplicationEventPublisher` for observability in Spring applications.

**Actuator `InfoContributor`**
Exposes registered mapper counts, supported formats, and strictness mode.

**Actuator `HealthIndicator`**
Reports whether all required definition files loaded successfully at startup.

**Test support**
`@ImportAutoConfiguration(TwinMapperAutoConfiguration.class)` for test slice inclusion. `@AutoConfigureTwinMapper` test annotation following the `@AutoConfigureMockMvc` pattern. `spring-boot-test-autoconfigure` as test-scoped dependency.

**Explicit activation annotations**
`@EnableTwinMapper`, `@EnableTwinMapperBinding`, `@EnableTwinMapperObjectEngine` for non-Boot or highly controlled Spring contexts where auto-configuration is not used.

---

### Optional

**Annotation-heavy usage mode**
Annotations as alternative configuration style. Definition files remain primary source of truth.
Property: `twinmapper.annotations.enabled`

---

## Developer Tooling Features

### Core

**Definition validation** — Validate definition files without running code generation.

**Dry-run generation** — Run full codegen pipeline and report what would be generated without writing files.

**Definition linting** — Report structural and semantic issues in definition files.

**Offline inspection** — Inspect and report on existing generated output.

**CLI boot hardening** — `SpringApplication.setWebApplicationType(WebApplicationType.NONE)` mandatory to prevent embedded servlet container startup.

---

### Optional

**Sample-to-definition assistant**
Inspects sample YAML, JSON, or BPMN inputs and proposes draft TwinMapper definitions. Requires human review. Never automatic live schema learning. CLI flag activation only — not a runtime property.

---

## Diagnostics and Error Handling

All errors include: error code, definition name, field path, source file, line/column where available, target Java type, rule violated, and migration hint for forbidden/deprecated field errors.

| Error Code | Trigger |
|---|---|
| `MISSING_REQUIRED_FIELD` | Required field absent in input |
| `UNKNOWN_FIELD` | Field not declared in definition (STRICT mode) |
| `FORBIDDEN_FIELD` | Field declared as forbidden or legacy |
| `DEPRECATED_FIELD` | Deprecated alias used (warning or error by mode) |
| `INVALID_ENUM` | Enum value not in declared set |
| `TYPE_MISMATCH` | Field value incompatible with declared type |
| `CONDITIONAL_VIOLATION` | Conditional constraint failed |
| `ONE_OF_VIOLATION` | One-of constraint failed |
| `MUTUALLY_EXCLUSIVE_VIOLATION` | Mutually exclusive fields both present |
| `CONSTRAINT_VIOLATION` | Min/max/length/pattern constraint failed |
| `AMBIGUOUS_MAPPING` | Convention mapping could not resolve uniquely |
| `UNSAFE_INVERSE` | Inverse mapper generation is ambiguous or unsafe |

---

## Hard Non-Goals

These are not features and must not be implemented in any form.

- Uncontrolled live schema learning
- Silent hidden guessing
- Runtime-first architecture
- Domain-specific semantics in the core platform

---

## External to TwinMapper

Not TwinMapper features. Enabled by TwinMapper SPIs but built and owned outside the platform.

- Customer-specific domain extension packs (finance, insurance, workflow, healthcare, and similar)

---

## Deferred from V1

Not part of the current release.

- Union/discriminator-capable types
- JSON-based object-mapping DSL
- Arbitrary additional definition DSL formats
- Annotation-first architecture
