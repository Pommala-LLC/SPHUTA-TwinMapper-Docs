# TwinMapper

TwinMapper is a compile-time-first, schema-driven code generation and mapping platform for Java. It generates strongly typed DTOs, enums, binders, validators, metadata, and typed object mappers from fixed YAML, JSON, and BPMN definitions. It provides runtime document binding and typed layer-to-layer object mapping without reflection-heavy inference or heuristic mapping as the core behavior.

---

## What TwinMapper solves

Most Java applications need two things that are painful to do consistently at scale:

1. Binding structured documents (YAML, JSON, BPMN) into typed Java models reliably and safely.
2. Copying and transforming data between layers — entity to domain, domain to DTO, request to command — without writing repetitive boilerplate or relying on runtime reflection.

TwinMapper solves both through a single compile-time-first platform with a Spring-native integration model.

---

## Two engines

### Document Engine
Reads fixed YAML, JSON, and BPMN definitions at build time. Generates DTOs, enums, binders, validators, registries, and metadata. At runtime, binds actual YAML, JSON, and BPMN XML documents into generated DTOs using a `NodeCursor` abstraction. No reflection at binding time.

### Object Engine
Generates typed Java-to-Java mappers for layer-crossing flows. Supports entity↔domain, domain↔DTO, DTO↔entity, entity↔projection, request/command↔domain, and domain↔event/view/response. Three mapper modes: CREATE, UPDATE, and PATCH.

---

## Core features

The baseline product. Everything here is present by default.

- **Compile-time-first design** — DTOs, binders, validators, and mappers are generated at build time from fixed definitions. Nothing is inferred at runtime.
- **Document binding engine** — Binds YAML, JSON, and BPMN XML documents into generated DTOs at runtime using format-specific parsers and a NodeCursor abstraction.
- **Typed object mapping engine** — Generates create, update, patch, and inverse mappers for typed Java layer-to-layer mapping flows.
- **Code generation** — Generates DTOs, enums, binders, validators, registries, metadata descriptors, and object mappers from the internal definition model.
- **YAML, JSON, and BPMN support** — Three first-class supported formats for both definitions and runtime documents.
- **TwinMapper-native YAML DSL** — A purpose-built YAML definition language covering types, fields, constraints, aliases, defaults, mappings, profiles, and converters.
- **SPI and extension model** — Definition reader SPI, runtime parser SPI, validator extension SPI, value converter SPI, and codegen customizer SPI.
- **Spring Boot integration** — First-class auto-configuration starter with `TwinMapperProperties`, conditional bean loading, `ConversionService` integration, Spring `Validator` wrapping, Actuator support, and test utilities.
- **Strict validation by default** — STRICT mode rejects unknown fields, invalid enums, missing required fields, incompatible types, and ambiguous mappings. COMPATIBLE and LENIENT modes are available.
- **Gradle and Maven plugins** — Build-time definition scanning, validation, source generation, and generated source root registration.

---

## Additional features

Required completeness items for a production-ready Spring-first baseline. Not optional product features — these must be present before implementation is considered complete.

- Shared Spring reflection helpers (`AnnotationUtils`, `ReflectionUtils`) and resource-loading abstractions (`ResourceUtils`, `PathMatchingResourcePatternResolver`) in the foundation.
- Deterministic SPI ordering via `OrderComparator`, `PriorityOrdered`, `@Order`, and `Ordered`.
- Deterministic `DefinitionSet` runtime identity mechanism.
- Generated bean disambiguation with `@Qualifier`, `@Primary`, and `@Conditional`.
- `basePackage` configuration for generated code layout.
- Full Spring validation integration: `ConstraintValidatorFactory`, `HandlerMethodArgumentResolver`, `ResponseEntityExceptionHandler`, `BindingResult`, `WebDataBinder`, `LocalValidatorFactoryBean`, `MethodValidationPostProcessor`.
- `TwinMapperRuntimeConfigurer` for Spring-idiomatic programmatic customization.
- `ConditionalGenericConverter` for type-aware conversion logic.
- Proxy-safe reflection via `AopUtils`, `AopProxyUtils`, and `ProxyUtils`.
- JSON naming consistency via `@JsonNaming`, `@JsonAlias`, `PropertyNamingStrategies`, and `Jackson2ObjectMapperBuilderCustomizer`.
- Secure SnakeYAML construction mandate and `spring.config.import` support.
- XXE-safe `XMLInputFactory` settings for BPMN parsing.
- Starter completeness: `@ConditionalOnClass`, `@ConditionalOnMissingBean`, `@ConditionalOnProperty`, `AutoConfiguration.imports`, `spring-configuration-metadata.json`, named binding/mapping events, Actuator `InfoContributor` and `HealthIndicator`, `@AutoConfigureTwinMapper` test annotation.
- CLI hardening with `WebApplicationType.NONE`.
- IDE-compatible generated source registration in both Gradle and Maven plugins.

---

## Optional features

Non-default capabilities. All are explicit opt-in, property-gated, default `false`, and must never replace generated binders or mappers as the primary path.

| Feature | Property Key |
|---|---|
| Controlled convention-based mapping mode | `twinmapper.objectmap.convention-mapping.enabled` |
| Annotation-heavy usage mode | `twinmapper.annotations.enabled` |
| Reflection-based compatibility helpers | `twinmapper.objectmap.reflection-compat.enabled` |
| Bounded runtime discovery helpers | `twinmapper.binding.bounded-discovery.enabled` |
| Alias/fallback compatibility resolution | `twinmapper.compatibility-resolution.enabled` |
| Sample-to-definition assistant tooling | CLI flag only — not a runtime property |

---

## What TwinMapper does not do

- No uncontrolled live schema learning
- No silent hidden field guessing
- No runtime-first architecture
- No domain-specific semantics in the core platform
- No customer-specific domain packs (those are external extensions built using TwinMapper SPIs)

---

## Spring ecosystem alignment

TwinMapper is designed to feel native in a Spring Boot application.

- Jackson for JSON — Spring Boot default
- SnakeYAML for YAML — Spring Boot bundled
- JDK StAX for BPMN — no vendor dependency
- Spring `ConversionService` as the converter backend
- Spring `Validator` and JSR-380 for validation
- Spring `ResourceLoader` and `PathMatchingResourcePatternResolver` for resource access
- Spring Boot `@AutoConfiguration` and `AutoConfiguration.imports` for the starter
- `TwinMapperProperties` via `@ConfigurationProperties(prefix = "twinmapper")` for all configuration

---

## Module families

| Family | Modules |
|---|---|
| Core | `twinmapper-core`, `twinmapper-definition-model`, `twinmapper-codegen`, `twinmapper-validation` |
| Runtime | `twinmapper-runtime`, `twinmapper-runtime-binding`, `twinmapper-runtime-objectmap` |
| Format | `twinmapper-format-json`, `twinmapper-format-yaml`, `twinmapper-format-bpmn` |
| Tooling | `twinmapper-gradle-plugin`, `twinmapper-maven-plugin`, `twinmapper-cli` |
| Integration | `twinmapper-spring-boot-starter` |

---

## Deferred from V1

- Union/discriminator-capable types
- JSON-based object-mapping DSL
- Arbitrary additional definition DSL formats
- Annotation-first architecture