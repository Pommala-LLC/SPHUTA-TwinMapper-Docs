# SPHUTA — TwinMapper

> **SPHUTA** — Enterprise Integration Platform  
> **TwinMapper** — Compile-time-first, schema-driven code generation and mapping framework

🟢🟢🟢⚪⚪⚪⚪⚪⚪⚪ **Development Progress:** 3 of 10 phases complete (30%)

---

TwinMapper is a compile-time-first, schema-driven code generation and mapping framework within the SPHUTA platform. It generates strongly typed DTOs, enums, binders, validators, metadata, and typed object mappers from fixed YAML, JSON, and BPMN definitions. It provides runtime document binding and typed layer-to-layer object mapping without reflection-heavy inference or heuristic mapping as the core behavior.

Built on Spring ecosystem defaults — integrates naturally with Spring Boot, Spring Validation, Spring Conversion, and Spring configuration patterns. Rather than assembling multiple external tools, TwinMapper provides a unified Spring-first framework for definition-driven binding, object mapping, validation, and code generation. Its closest direct competitors are MapStruct and ModelMapper.

| Property | Value |
|----------|-------|
| **Base package** | `net.sphuta.twinmapper` |
| **Java** | 21+ |
| **Spring Framework** | 7.x |
| **Spring Boot** | 4.x |
| **Null-safety** | JSpecify 1.0.0 (`@NullMarked`, `@Nullable` from `org.jspecify.annotations`) |

---

## What TwinMapper Solves

Most Java applications need two things that are painful to do consistently at scale:

1. **Binding structured documents** (YAML, JSON, BPMN) into typed Java models reliably and safely.
2. **Mapping data between layers** — entity to domain, domain to DTO, request to command — without writing repetitive boilerplate or relying on runtime reflection.

TwinMapper solves both through a single compile-time-first framework with a Spring-native integration model.

---

## Two Engines

| | Document Engine | Object Engine |
|---|---|---|
| **Input** | YAML / JSON / BPMN definitions | YAML mapping DSL |
| **Generates** | DTOs, enums, binders, validators, registries, metadata | Typed Java-to-Java mappers |
| **Runtime** | Binds documents into generated DTOs via `NodeCursor` | Executes CREATE, UPDATE, PATCH mappers |
| **Flows** | YAML / JSON / BPMN XML → Generated DTO | entity↔domain, domain↔DTO, request↔command |
| **Reflection** | None at binding time | Proxy-safe only (AopUtils) |

---

## Architecture Overview

```
                    ╔══════════════════════════════════════════╗
                    ║           BUILD-TIME PIPELINE            ║
                    ╠══════════════════════════════════════════╣
                    ║                                          ║
                    ║   YAML / JSON / BPMN Definition Files    ║
                    ║               │                          ║
                    ║               ▼                          ║
                    ║   ┌──────────────────────────┐           ║
                    ║   │     Format Readers        │          ║
                    ║   │  yaml · json · bpmn       │          ║
                    ║   └────────────┬─────────────┘           ║
                    ║                ▼                         ║
                    ║   ┌──────────────────────────┐           ║
                    ║   │   DefinitionSet          │           ║
                    ║   │   (Canonical Meta-Model)  │          ║
                    ║   └────────────┬─────────────┘           ║
                    ║                ▼                         ║
                    ║   ┌──────────────────────────┐           ║
                    ║   │   twinmapper-codegen      │          ║
                    ║   │   Generated Artifacts     │          ║
                    ║   └────────────┬─────────────┘           ║
                    ╠════════════════╪═════════════════════════╣
                    ║                ▼  RUNTIME PLATFORM       ║
                    ║   ┌──────────────────────────┐           ║
                    ║   │   twinmapper-runtime      │          ║
                    ║   │   (Shared Contracts)      │          ║
                    ║   └──────┬───────────┬───────┘           ║
                    ║          ▼           ▼                   ║
                    ║   ┌────────────┐ ┌──────────────┐        ║
                    ║   │  binding   │→│  objectmap   │        ║
                    ║   │ NodeCursor │ │ ConversionSvc│        ║
                    ║   └──────┬─────┘ └──────┬───────┘        ║
                    ║          └──────┬───────┘                ║
                    ║                 ▼                        ║
                    ║   ┌──────────────────────────┐           ║
                    ║   │   twinmapper-validation   │          ║
                    ║   │   ValidationReport        │          ║
                    ║   └──────────────────────────┘           ║
                    ╚══════════════════════════════════════════╝
```

---

## Module Architecture

### 🏗️ Architectural Sections

TwinMapper's 14 core modules are organized into **five architectural sections** with clean dependency boundaries:

```
┌─────────────────────────────────────────────────────────────────┐
│  🟣 BUILD-TIME PIPELINE                                        │
│  Authoring input → Canonical model → Generated artifacts        │
│                                                                 │
│  Phase 1: core, definition-model                                │
│  Phase 2: format-yaml, format-json, format-bpmn                 │
│  Phase 3: codegen                                               │
├─────────────────────────────────────────────────────────────────┤
│  🔵 RUNTIME PLATFORM                                           │
│  Execution of generated artifacts                               │
│                                                                 │
│  Phase 4: runtime                                               │
│  Phase 5: runtime-binding, runtime-objectmap                    │
│  Phase 6: validation                                            │
├───────────────────────────┬─────────────────────────────────────┤
│  🟢 CONSUMPTION A        │  🟠 CONSUMPTION B                   │
│  Build-system entry       │  Application entry point            │
│                           │                                     │
│  Phase 7: gradle-plugin,  │  Phase 8: spring-boot-starter       │
│           maven-plugin    │                                     │
│  Phase 9: cli             │                                     │
├───────────────────────────┴─────────────────────────────────────┤
│  ⚪ OPTIONAL EXTENSIONS                                        │
│  Non-primary, opt-in, property-gated features                   │
│                                                                 │
│  Phase 10: annotations, annotation-processor, runtime-compat    │
└─────────────────────────────────────────────────────────────────┘
```

---

### 🔗 Dependency Spine

The locked dependency chain — all modules depend downward only, no circular dependencies permitted.

**Primary spine (runtime path):**

```
core
 └→ definition-model
     └→ format-yaml / format-json / format-bpmn
         └→ codegen
             └→ runtime
                 └→ runtime-binding
                     └→ runtime-objectmap
                         └→ validation
                             └→ spring-boot-starter
```

**Orthogonal consumers (build path):**

```
codegen
 ├→ gradle-plugin
 ├→ maven-plugin
 └→ cli
```

> ⚠️ **Key distinction:** Plugins and CLI consume the **build-time pipeline** (codegen). The Spring Boot starter consumes the **runtime/validation platform**. These are architecturally distinct delivery channels with different dependency directions.

---

### 📦 Module Families

| Family | Modules | Role |
|--------|---------|------|
| **Core** | `twinmapper-core`, `twinmapper-definition-model`, `twinmapper-codegen`, `twinmapper-validation` | Foundation, meta-model, generation, validation |
| **Runtime** | `twinmapper-runtime`, `twinmapper-runtime-binding`, `twinmapper-runtime-objectmap` | Contracts, document binding, object mapping |
| **Format** | `twinmapper-format-json`, `twinmapper-format-yaml`, `twinmapper-format-bpmn` | Definition readers + runtime parsers |
| **Tooling** | `twinmapper-gradle-plugin`, `twinmapper-maven-plugin`, `twinmapper-cli` | Build integration + developer tooling |
| **Integration** | `twinmapper-spring-boot-starter` | Spring Boot auto-configuration |

---

### 🧱 Grouped Modules by Phase

| Section | Phase | Modules | Status |
|---------|-------|---------|--------|
| 🟣 **Build-Time Pipeline** | 1 — Foundation | `twinmapper-core`, `twinmapper-definition-model` | ✅ |
| 🟣 **Build-Time Pipeline** | 2 — Definition Readers | `twinmapper-format-yaml`, `twinmapper-format-json`, `twinmapper-format-bpmn` | ✅ |
| 🟣 **Build-Time Pipeline** | 3 — Code Generation | `twinmapper-codegen` | ✅ |
| 🔵 **Runtime Platform** | 4 — Runtime Base | `twinmapper-runtime` | ⬚ |
| 🔵 **Runtime Platform** | 5 — Runtime Engines | `twinmapper-runtime-binding`, `twinmapper-runtime-objectmap` | ⬚ |
| 🔵 **Runtime Platform** | 6 — Validation | `twinmapper-validation` | ⬚ |
| 🟢 **Build Consumption** | 7 — Build Plugins | `twinmapper-gradle-plugin`, `twinmapper-maven-plugin` | ⬚ |
| 🟢 **Build Consumption** | 9 — CLI | `twinmapper-cli` | ⬚ |
| 🟠 **App Consumption** | 8 — Starter | `twinmapper-spring-boot-starter` | ⬚ |
| ⚪ **Optional** | 10 — Extensions | `twinmapper-annotations`, `twinmapper-annotation-processor`, `twinmapper-runtime-compat` | ⬚ |

---

## Guiding Principles

> **1. TwinMapper is build-time-first.**
> Its primary path is definition-driven code generation, not reflection-first runtime mapping.

> **2. BPMN is an authoring format, not a workflow engine.**
> BPMN definitions are parsed into the canonical meta-model and do not define TwinMapper runtime semantics.

> **3. The runtime platform is layered.**
> `runtime-binding` handles traversal and extraction; `runtime-objectmap` handles typed object mapping on top of that foundation. They are stacked, not peers.

> **4. Plugins, starter, and CLI are delivery channels.**
> They expose TwinMapper to build systems and applications, but are not part of the core engine.

---

## Core Features

The baseline product. Everything here is present by default.

- **Compile-time-first design** — DTOs, binders, validators, and mappers are generated at build time from fixed definitions. Nothing is inferred at runtime.
- **Document binding engine** — Binds YAML, JSON, and BPMN XML documents into generated DTOs at runtime using format-specific parsers and a `NodeCursor` abstraction.
- **Typed object mapping engine** — Generates create, update, patch, and inverse mappers for typed Java layer-to-layer mapping flows.
- **Code generation** — Generates DTOs, enums, binders, validators, registries, metadata descriptors, and object mappers from the internal definition model.
- **YAML, JSON, and BPMN support** — Three first-class supported formats for both definitions and runtime documents.
- **TwinMapper-native YAML DSL** — A purpose-built YAML definition language covering types, fields, constraints, aliases, defaults, mappings, profiles, and converters.
- **SPI and extension model** — Definition reader SPI, runtime parser SPI, validator extension SPI, value converter SPI, and codegen customizer SPI.
- **Spring Boot integration** — First-class auto-configuration starter with `TwinMapperProperties`, conditional bean loading, `ConversionService` integration, Spring `Validator` wrapping, Actuator support, and test utilities.
- **Strict validation by default** — STRICT mode rejects unknown fields, invalid enums, missing required fields, incompatible types, and ambiguous mappings. COMPATIBLE and LENIENT modes are available.
- **Gradle and Maven plugins** — Build-time definition scanning, validation, source generation, and generated source root registration.

---

## Additional Features

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

## Optional Features

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

## How TwinMapper Compares

### Direct competitors

| Tool | What it does | What TwinMapper adds |
|---|---|---|
| **MapStruct** | Compile-time generated object mappers from annotated interfaces | All of MapStruct's object mapping, plus document binding, schema-driven DTO generation, BPMN support, and a YAML-first definition model. MapStruct is the quality benchmark for TwinMapper's object engine. |
| **ModelMapper** | Runtime reflection and convention-based object mapping | TwinMapper is the architectural opposite: generated at compile time, no heuristic guessing, explicit definitions over conventions, fails loudly on ambiguity. |

### Replacement targets

These are tools that TwinMapper makes unnecessary for its target use cases. They are not direct competitors — they are tools handling separate concerns that TwinMapper unifies.

| Tool | Replaced by |
|---|---|
| **jsonschema2pojo** | TwinMapper's Document Engine generates DTOs from YAML/JSON definitions with full Spring-native output and no Jackson annotation coupling |
| **OpenAPI Generator** | TwinMapper's codegen pipeline for DTO and binder generation from structured definitions — scoped to this use case, not OpenAPI Generator's full client/server generation remit |
| **Camunda BPMN tooling** | TwinMapper's `twinmapper-format-bpmn` owns BPMN parsing and binding via JDK StAX with no vendor dependency |

### Supporting integration technology

| Tool | Relationship |
|---|---|
| **Apache Avro** | A wire format and schema registry concern at a different layer. TwinMapper operates at the Java application layer. Avro is a potential integration point, not a competitor. |

### Legacy mapping frameworks

Dozer and Orika are older reflection-based mappers still present in legacy codebases. See [MIGRATION_GUIDE.md](docs/MIGRATION_GUIDE.md) for migration paths away from both.

---

## What TwinMapper Does Not Do

- ❌ No uncontrolled live schema learning
- ❌ No silent hidden field guessing
- ❌ No runtime-first architecture
- ❌ No domain-specific semantics in the core platform
- ❌ No customer-specific domain packs (those are external extensions built using TwinMapper SPIs)
- ❌ Not a Micronaut alternative — TwinMapper is Spring-first and does not target the Micronaut ecosystem

---

## Spring Ecosystem Alignment

TwinMapper is designed to feel native in a Spring Boot application.

| Concern | Spring Tool | Notes |
|---------|-------------|-------|
| JSON parsing | Jackson | Spring Boot default |
| YAML parsing | SnakeYAML | Spring Boot bundled |
| BPMN parsing | JDK StAX | No vendor dependency |
| Type conversion | `ConversionService` | Converter backend |
| Validation | `Validator` + JSR-380 | Spring-native |
| Resource access | `ResourceLoader` + `PathMatchingResourcePatternResolver` | Classpath scanning |
| Auto-configuration | `@AutoConfiguration` + `AutoConfiguration.imports` | Starter wiring |
| Configuration | `TwinMapperProperties` via `@ConfigurationProperties(prefix = "twinmapper")` | Central config |
| Null-safety | JSpecify 1.0.0 | `@NullMarked`, `@Nullable` from `org.jspecify.annotations` |

---

## Technology Stack

| Dependency | Version | Purpose |
|------------|---------|---------|
| Java | 21+ | Records, sealed interfaces, pattern matching |
| `spring-core` | 7.x | Utilities, ResourceLoader, ConversionService SPI |
| `spring-context` | 7.x | Auto-configuration contracts |
| `spring-boot` | 4.x | @ConfigurationProperties, AutoConfiguration |
| `org.jspecify:jspecify` | 1.0.0 | Null-safety annotations (`@NullMarked`, `@Nullable`) |
| `jackson-databind` | 2.18+ | JSON format support |
| `snakeyaml` | 2.x | YAML format support (bundled in Spring Boot) |
| JDK StAX | Built-in | BPMN XML parsing — no vendor dependency |

> **No mandatory runtime dependency on:** Lombok, MapStruct, Micronaut, any BPMN library, or Hibernate Validator (optional).

---

## Documentation

| Document | Description |
|----------|-------------|
| [ARCHITECTURE.md](docs/ARCHITECTURE.md) | Two-engine architecture, module layer map, Spring integration model |
| [MODULES.md](docs/MODULES.md) | Full module-by-module responsibility and tool reference |
| [FEATURES.md](docs/FEATURES.md) | Complete feature reference by category and classification |
| [FEATURE_MATRIX.md](docs/FEATURE_MATRIX.md) | Locked feature matrix: Core, Additional, Optional, Hard Non-Goals |
| [DEVELOPMENT_PHASE_DETAILS.md](docs/DEVELOPMENT_PHASE_DETAILS.md) | Ten development phases with entry/exit conditions |
| [AUTHORING_GUIDE.md](docs/AUTHORING_GUIDE.md) | YAML DSL reference for definitions and mappings |
| [CODEGEN_CONTRACT.md](docs/CODEGEN_CONTRACT.md) | Generated artifact shapes, naming conventions, guarantees |
| [CONFIGURATION.md](docs/CONFIGURATION.md) | All configuration properties, Gradle/Maven plugin config |
| [OBJECT_MAPPING_SPEC.md](docs/OBJECT_MAPPING_SPEC.md) | Object engine full specification |
| [DOCUMENT_BINDING_SPEC.md](docs/DOCUMENT_BINDING_SPEC.md) | Document engine full specification |
| [VALIDATION_SPEC.md](docs/VALIDATION_SPEC.md) | Validation model, constraints, Spring integration |
| [BPMN_SUPPORT_MATRIX.md](docs/BPMN_SUPPORT_MATRIX.md) | Supported BPMN 2.0 element vocabulary |
| [SPI_EXTENSION_GUIDE.md](docs/SPI_EXTENSION_GUIDE.md) | All SPI interfaces with registration examples |
| [TESTING_STRATEGY.md](docs/TESTING_STRATEGY.md) | Internal and consumer testing guidance |
| [SECURITY_MODEL.md](docs/SECURITY_MODEL.md) | XXE, YAML safe construction, runtime security model |
| [COMPATIBILITY.md](docs/COMPATIBILITY.md) | Platform requirements and version compatibility |
| [DECISIONS.md](docs/DECISIONS.md) | Architecture decision records |
| [ERROR_CATALOG.md](docs/ERROR_CATALOG.md) | All error codes, triggers, and resolution guidance |
| [EXAMPLES.md](docs/EXAMPLES.md) | Ten end-to-end examples |
| [GLOSSARY.md](docs/GLOSSARY.md) | All terms defined |
| [MIGRATION_GUIDE.md](docs/MIGRATION_GUIDE.md) | Schema migration, MapStruct/ModelMapper/Dozer/Orika migration paths |
| [FAQ.md](docs/FAQ.md) | Frequently asked questions |
| [CONTRIBUTING.md](docs/CONTRIBUTING.md) | Contribution guide and standards |
| [RELEASE_STRATEGY.md](docs/RELEASE_STRATEGY.md) | Versioning, release cadence, compatibility promise |

---

## Roadmap Beyond V1

- Union / discriminator-capable types
- JSON-based object-mapping DSL
- Arbitrary additional definition DSL formats
- Annotation-first architecture
