# TwinMapper — Development Phases

This document defines the development phases for TwinMapper. Phases are ordered by dependency, stability risk, and integration readiness. Each phase has a clear entry condition, a defined scope, and a clear exit condition before the next phase begins.

The two open implementation decisions must be resolved before Phase 1 begins:
- Source writer tool for `twinmapper-codegen`
- `DefinitionSet` runtime identity mechanism

---

## Phase 1 — Foundation and Definition Model

**Goal:** Establish the shared foundation and the canonical internal meta-model that all other phases depend on.

**Entry condition:** Open implementation decisions resolved.

### Modules

- `twinmapper-core`
- `twinmapper-definition-model`

### Scope

**`twinmapper-core`**
- Common error model and result model
- Shared constants and base exception hierarchy
- Base SPI interfaces
- Shared diagnostics structures
- Common utility helpers and marker interfaces
- Path and reference abstractions
- Shared policy enums
- Spring utility integration: `Assert`, `StringUtils`, `ObjectUtils`, `ClassUtils`, `AnnotationUtils`, `ReflectionUtils`
- Resource-loading abstractions: `ResourceUtils`, `PathMatchingResourcePatternResolver`
- Deterministic SPI ordering: `OrderComparator`, `PriorityOrdered`, `@Order`, `Ordered`

**`twinmapper-definition-model`**
- All meta-model types: `DefinitionSet`, `TypeDefinition`, `ObjectTypeDefinition`, `EnumTypeDefinition`, `FieldDefinition`, `CollectionDefinition`, `MapDefinition`, `AliasDefinition`, `ConstraintDefinition`, `ConditionalConstraintDefinition`, `DefaultValueDefinition`, `DeprecationDefinition`, `ForbiddenFieldDefinition`, `VersionDefinition`, `ObjectMappingDefinition`, `ProfileDefinition`, `ConverterDefinition`, `GenerationOptionsDefinition`
- Deterministic `DefinitionSet` runtime identity mechanism
- Format-neutral design with no Spring container dependency

### Exit condition

`twinmapper-core` and `twinmapper-definition-model` compile cleanly. All meta-model types are defined and unit-tested. No format module or codegen module depends on anything not in these two modules.

---

## Phase 2 — Format Readers

**Goal:** Enable all three formats to read definition files into the internal meta-model at build time.

**Entry condition:** Phase 1 complete and verified.

### Modules

- `twinmapper-format-yaml`
- `twinmapper-format-json`
- `twinmapper-format-bpmn`

### Scope

**`twinmapper-format-yaml`**
- TwinMapper-native YAML DSL reader
- YAML object-mapping DSL reader
- SnakeYAML with mandatory safe construction
- `YamlPropertiesFactoryBean`, `YamlMapFactoryBean` alignment
- `ConfigDataLoader` and `ConfigDataLocationResolver` for `spring.config.import` support
- DSL constructs: `definitionSet`, types, fields, constraints, aliases, profiles, converters, generation options

**`twinmapper-format-json`**
- JSON definition reader
- Jackson `ObjectMapper` and `JsonMapper.builder()`
- Read fixed JSON definitions into internal meta-model

**`twinmapper-format-bpmn`**
- BPMN support model reader (build time)
- JDK StAX `XMLInputFactory` with mandatory XXE-safe settings
- `BpmnDocumentParser` producing `BpmnIntermediateModel`
- `BpmnIntermediateModelMapper` mapping into internal definition model
- Full supported BPMN vocabulary defined and mapped
- `ResourceLoader` integration mandatory

### Exit condition

All three format readers can load valid definition files and produce correct `DefinitionSet` instances. Invalid definitions produce clear, path-aware errors. Secure parsing requirements pass security review.

---

## Phase 3 — Code Generation

**Goal:** Generate all Java artifacts from the internal meta-model.

**Entry condition:** Phase 2 complete. Format readers produce stable `DefinitionSet` instances. Source writer tool decision finalized.

### Modules

- `twinmapper-codegen`

### Scope

- DTO generation: immutable records by default, mutable classes configurable, Lombok-free
- Enum generation from declared values
- Metadata descriptor generation
- Document binder generation for YAML, JSON, and BPMN
- Schema validator generation covering all v1 constraint types
- Registry generation: binder, mapper, metadata, converter, type lookup
- Object mapper generation: create, update, patch, inverse
- Collection and nested mapper generation
- Optional `@Component` on generated beans when Spring bean mode is enabled
- `@Qualifier`, `@Primary`, `@Conditional` support for disambiguation
- `basePackage` configuration option
- Java-friendly field name generation with source names preserved in metadata

### Exit condition

Given a valid `DefinitionSet`, the codegen module produces compilable Java source files for all artifact types. Generated code passes a round-trip test: a generated binder can bind a valid input document into a generated DTO, and a generated mapper can map between two generated types.

---

## Phase 4 — Runtime Binding

**Goal:** Enable runtime document binding using the generated binders.

**Entry condition:** Phase 3 complete. Generated binders compile and are structurally correct.

### Modules

- `twinmapper-runtime`
- `twinmapper-runtime-binding`

### Scope

**`twinmapper-runtime`**
- `TwinMapperRuntime` facade interface
- `BindingContext`, `MappingContext`
- Registry interfaces for binders, mappers, converters, and metadata
- `AliasResolutionPolicy` shared contract
- `CompatibilityMode` enum
- `TwinMapperRuntimeConfigurer` interface
- Common runtime policies and result models

**`twinmapper-runtime-binding`**
- `NodeCursor` abstraction for YAML, JSON, and BPMN traversal
- Binder dispatch and execution
- `BindingResult` and error accumulation
- Path-aware diagnostic collection
- Required field enforcement, default application, alias resolution
- YAML document parsing via SnakeYAML into `NodeCursor`
- JSON document parsing via Jackson into `NodeCursor`
- BPMN document parsing via StAX into `BpmnIntermediateModel` into `NodeCursor`
- Strict unknown-field behavior
- Collection and map binding
- Nested object binding
- `PathMatchingResourcePatternResolver` for definition directory scanning
- `ResourceLoader` mandatory for all classpath access
- `Environment` for property-backed resolution

### Exit condition

A valid YAML, JSON, and BPMN runtime document can be fully bound into a generated DTO. Invalid documents produce correct, aggregated, path-aware errors. Strict mode correctly rejects unknown fields, forbidden fields, and invalid enums.

---

## Phase 5 — Validation Engine

**Goal:** Wire the generated validators into a cohesive Spring-native validation pipeline.

**Entry condition:** Phase 4 complete. Binding produces correct DTOs.

### Modules

- `twinmapper-validation`

### Scope

- Generated schema validator execution
- Aggregated `ValidationReport` with path-aware errors
- Required, optional, default, enum, forbidden, deprecated, conditional, one-of, mutually exclusive, min, max, minLength, maxLength, regex/pattern, unknown field, collection/map type checks
- Spring `Validator` and `SmartValidator` integration
- `ConstraintValidatorFactory` for Spring-injected custom constraint validators
- `HandlerMethodArgumentResolver` for Spring MVC method parameter binding
- `ResponseEntityExceptionHandler` alignment
- `@Valid`, `@Validated`, `jakarta.validation` JSR-380 integration
- `BindingResult`, `WebDataBinder`, `LocalValidatorFactoryBean`, `MethodValidationPostProcessor`

### Exit condition

Generated validators run correctly via Spring's `Validator` contract. Validation errors accumulate into `ValidationReport` and translate cleanly into Spring MVC exception types. JSR-380 annotations work alongside generated validators.

---

## Phase 6 — Object Engine

**Goal:** Enable typed Java-to-Java mapping across all supported layer flows.

**Entry condition:** Phase 5 complete. Validation pipeline stable.

### Modules

- `twinmapper-runtime-objectmap`

### Scope

- Generated create, update, patch, inverse mapper execution
- `TwinObjectMapper`, `TwinUpdateMapper`, `TwinPatchMapper`, `TwinObjectMapperRegistry`
- Spring `ConversionService` as converter backend: `GenericConversionService`, `FormattingConversionService`, `DefaultConversionService`
- `Converter`, `ConverterFactory`, `GenericConverter`, `ConditionalGenericConverter` as SPI contracts
- `BeanWrapper` for UPDATE and PATCH mode on Spring-managed bean targets
- `DirectFieldAccessor` for targets without setters
- Proxy-safe reflection via `AopUtils.getTargetClass()`, `AopProxyUtils.ultimateTargetClass()`, `ProxyUtils.getUserClass()`
- All supported mapping flows operational
- Null-policy enforcement: `IGNORE_NULLS`, `SET_NULLS`, `FAIL_ON_NULL`
- Reusable profiles operational
- Collection and nested mapping operational
- Flattening and expansion operational

### Exit condition

All thirteen supported mapping flows work end-to-end with generated mappers. Create, update, and patch modes behave correctly with null policies. Proxy-safe behavior verified against Spring-managed proxied beans. Inverse mapping works or fails with a clear build-time error.

---

## Phase 7 — Build Plugins

**Goal:** Integrate code generation into Gradle and Maven build lifecycles.

**Entry condition:** Phase 6 complete. All generation and runtime modules stable.

### Modules

- `twinmapper-gradle-plugin`
- `twinmapper-maven-plugin`

### Scope

**`twinmapper-gradle-plugin`**
- Scan configured definition directories
- Validate definitions before generation
- Invoke `twinmapper-codegen`
- Register generated sources via `sourceSets.main.java.srcDir(generatedDir)`
- Gradle Plugin API and Worker API integration
- Deterministic CI-friendly output
- Dry-run and diagnostics support
- IDE-compatible source root registration for IntelliJ IDEA and Eclipse

**`twinmapper-maven-plugin`**
- Bind to `generate-sources` lifecycle phase
- Validate definitions before generation
- Invoke `twinmapper-codegen`
- Register generated sources via `project.addCompileSourceRoot(generatedDir)`
- Validation-only and dry-run goals
- IDE-compatible source root registration

### Exit condition

A Spring Boot project with TwinMapper definition files generates correct Java sources as part of `gradle build` and `mvn compile`. Generated sources compile cleanly with handwritten code. IDE tooling recognizes the generated source directory.

---

## Phase 8 — Spring Boot Starter

**Goal:** Provide a first-class Spring Boot auto-configuration experience.

**Entry condition:** Phase 7 complete. All modules stable and build-integrated.

### Modules

- `twinmapper-spring-boot-starter`

### Scope

- `@AutoConfiguration` with `AutoConfiguration.imports` for Spring Boot 3.x/4.x
- `@ConditionalOnClass`, `@ConditionalOnMissingBean`, `@ConditionalOnProperty`
- `@ConditionalOnMissingBean(TwinMapperRuntime.class)` referencing the interface type
- `TwinMapperProperties` via `@ConfigurationProperties(prefix = "twinmapper")`
- `@EnableConfigurationProperties(TwinMapperProperties.class)`
- `spring-configuration-metadata.json` for IDE auto-completion
- Auto-configure `TwinMapperRuntime`, binder registry, object mapper registry
- Wire Spring `ConversionService` into converter registry
- Wrap generated validators as Spring `Validator` implementations
- Named application events: `TwinMapperBindingSuccessEvent`, `TwinMapperBindingFailureEvent`, `TwinMapperMappingSuccessEvent`, `TwinMapperMappingFailureEvent`
- Actuator `InfoContributor` and `HealthIndicator`
- `@ImportAutoConfiguration(TwinMapperAutoConfiguration.class)` support
- `@AutoConfigureTwinMapper` test annotation
- `spring-boot-test-autoconfigure` as test-scoped dependency
- `@EnableTwinMapper`, `@EnableTwinMapperBinding`, `@EnableTwinMapperObjectEngine` for non-Boot contexts
- `Jackson2ObjectMapperBuilderCustomizer` for field-naming consistency

### Exit condition

A Spring Boot 3.x application with `twinmapper-spring-boot-starter` on the classpath auto-configures TwinMapper without any additional configuration. `TwinMapperRuntime`, binder registry, and object mapper registry are available as beans. `application.properties` auto-completion works in IntelliJ IDEA and VS Code. Test slices work with `@AutoConfigureTwinMapper`. Actuator endpoints show TwinMapper metadata.

---

## Phase 9 — CLI and Developer Tooling

**Goal:** Provide standalone developer tooling for definition management and inspection.

**Entry condition:** Phase 8 complete.

### Modules

- `twinmapper-cli`

### Scope

- `SpringApplication.setWebApplicationType(WebApplicationType.NONE)` mandatory
- `ApplicationRunner`, `CommandLineRunner`, `ExitCodeGenerator`, `SpringApplication.exit()`
- Picocli Spring Boot starter if Picocli chosen
- Definition validation command
- Dry-run generation command
- Definition linting command
- Offline inspection and reporting
- Sample-to-definition assistant tooling (CLI flag only, never runtime behavior, requires human review)

### Exit condition

CLI can validate definitions, report linting issues, run dry-run generation, and propose draft definitions from sample inputs. No embedded server starts. All commands exit with correct exit codes.

---

## Phase 10 — Optional Features

**Goal:** Implement the six optional, property-gated capabilities.

**Entry condition:** Phases 1–9 complete and stable. Optional features are explicitly requested and properly scoped.

### Scope and activation

Each optional feature is disabled by default and must be enabled via `TwinMapperProperties`.

**Controlled convention-based mapping mode**
Module: `twinmapper-runtime-objectmap`
Property: `twinmapper.objectmap.convention-mapping.enabled`
Implementation: `ConventionMappingResolver`, `ConventionMatchingPolicy`. Deterministic only. Ambiguity causes failure, never silent resolution.

**Annotation-heavy usage mode**
Modules: `twinmapper-annotations`, `twinmapper-annotation-processor`
Property: `twinmapper.annotations.enabled`
Implementation: `@TwinMap`, `@TwinField`, `@TwinIgnore`, `@TwinConverter`, `@TwinNullPolicy`, `@TwinVersion`. APT processor bridges annotations into the same internal definition model. Definition files remain primary source of truth.

**Reflection-based compatibility helpers**
Module: `twinmapper-runtime-objectmap` or separate `twinmapper-runtime-compat`
Property: `twinmapper.objectmap.reflection-compat.enabled`
Implementation: Migration helper mapper, dev-time compatibility copier, bounded reflection-assisted legacy object reader. Proxy-aware via `AopUtils`/`AopProxyUtils`/`ProxyUtils`. Never the default runtime path.

**Bounded runtime discovery helpers**
Module: `twinmapper-runtime-binding`
Property: `twinmapper.binding.bounded-discovery.enabled`
Implementation: Adapter-specific helpers only. Explicit opt-in. Produces traceable artifacts. Never replaces fixed-definition mode.

**Alias/fallback compatibility resolution**
Modules: `twinmapper-runtime-binding`, `twinmapper-runtime-objectmap`
Property: `twinmapper.compatibility-resolution.enabled`
Implementation: `AliasResolutionPolicy`, `FallbackFieldResolver`, `CompatibilityNameResolver`. Policy-controlled, visible, never hidden guessing.

**Sample-to-definition assistant tooling**
Module: `twinmapper-cli`
Activation: CLI flag only — not a runtime property
Implementation: Offline only. Proposes draft definitions from sample inputs. Human review mandatory. Never automatic live schema learning.

### Exit condition

Each optional feature activates correctly when its property is set to `true` and remains completely inactive when set to `false` or absent. No optional feature affects the behavior of the generated binder or mapper primary path.

---

## Deferred — Post-V1

These items are explicitly out of scope for all phases above and must not be introduced into any phase.

- Union/discriminator-capable types
- JSON-based object-mapping DSL
- Arbitrary additional definition DSL formats
- Annotation-first architecture

---

## Hard Non-Goals — Never Implemented

These must not appear in any phase.

- Uncontrolled live schema learning
- Silent hidden guessing
- Runtime-first architecture
- Domain-specific semantics in the core platform

---

## External — Not Part of Any Phase

- Customer-specific domain extension packs built using TwinMapper SPIs

---

## Phase Summary

| Phase | Modules | Goal |
|---|---|---|
| 1 | `twinmapper-core`, `twinmapper-definition-model` | Foundation and meta-model |
| 2 | `twinmapper-format-yaml/json/bpmn` | Format definition readers |
| 3 | `twinmapper-codegen` | Code generation |
| 4 | `twinmapper-runtime`, `twinmapper-runtime-binding` | Runtime document binding |
| 5 | `twinmapper-validation` | Validation engine |
| 6 | `twinmapper-runtime-objectmap` | Object mapping engine |
| 7 | `twinmapper-gradle-plugin`, `twinmapper-maven-plugin` | Build integration |
| 8 | `twinmapper-spring-boot-starter` | Spring Boot auto-configuration |
| 9 | `twinmapper-cli` | Developer tooling |
| 10 | Optional modules | Optional feature implementation |
