# TwinMapper — Architecture

## Overview

TwinMapper is a compile-time-first, schema-driven code generation and mapping platform. Its architecture is split into two primary engines — the Document Engine and the Object Engine — supported by a shared foundation, three format modules, build tooling, and a Spring Boot integration layer.

The central architectural rule is: all primary behavior is generated at build time from fixed definitions. Runtime behavior executes the generated code. Nothing is inferred at runtime from live payloads.

---

## Architectural Principles

- **Compile-time first** — DTOs, binders, validators, and mappers are generated at build time. No runtime inference.
- **Definition first** — The internal meta-model is the single source of truth. All generated artifacts derive from it.
- **Strongly typed** — No reflection-heavy runtime inference. No heuristic field guessing.
- **Deterministic** — Identical inputs always produce identical outputs across builds.
- **Separation of concerns** — Document binding and object mapping are independent engines sharing a common foundation.
- **Spring-native** — Integration with Spring's validation, conversion, resource-loading, and auto-configuration infrastructure is first-class, not an afterthought.
- **Generic core** — No domain-specific semantics in the core platform. All domain knowledge lives in consuming applications or external extension packs.

---

## High-Level Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        Build Time                               │
│                                                                 │
│  YAML / JSON / BPMN       Object Mapping                        │
│  Definition Files         DSL Files                             │
│         │                      │                                │
│         ▼                      ▼                                │
│  ┌─────────────────────────────────────────┐                    │
│  │        Format Readers                   │                    │
│  │  format-yaml  format-json  format-bpmn  │                    │
│  └─────────────────────────────────────────┘                    │
│                      │                                          │
│                      ▼                                          │
│  ┌─────────────────────────────────────────┐                    │
│  │        twinmapper-definition-model      │                    │
│  │        (Internal Meta-Model)            │                    │
│  └─────────────────────────────────────────┘                    │
│                      │                                          │
│                      ▼                                          │
│  ┌─────────────────────────────────────────┐                    │
│  │        twinmapper-codegen               │                    │
│  │  DTOs / Enums / Binders / Validators /  │                    │
│  │  Registries / Mappers / Metadata        │                    │
│  └─────────────────────────────────────────┘                    │
│                      │                                          │
│                      ▼                                          │
│            Generated Java Sources                               │
│         build/generated/sources/twinmapper                      │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                        Runtime                                  │
│                                                                 │
│   YAML / JSON / BPMN        Java Source Object                  │
│   Runtime Documents              │                              │
│          │                       │                              │
│          ▼                       ▼                              │
│  ┌──────────────────┐   ┌──────────────────────┐               │
│  │ runtime-binding  │   │  runtime-objectmap   │               │
│  │                  │   │                      │               │
│  │ NodeCursor       │   │ TwinObjectMapper     │               │
│  │ Binder Dispatch  │   │ TwinUpdateMapper     │               │
│  │ BindingResult    │   │ TwinPatchMapper      │               │
│  └────────┬─────────┘   └──────────┬───────────┘               │
│           │                        │                            │
│           └──────────┬─────────────┘                            │
│                      ▼                                          │
│  ┌─────────────────────────────────────────┐                    │
│  │        twinmapper-validation            │                    │
│  │        ValidationReport                 │                    │
│  └─────────────────────────────────────────┘                    │
│                      │                                          │
│                      ▼                                          │
│             Consuming Application                               │
└─────────────────────────────────────────────────────────────────┘
```

---

## Module Layer Map

```
Layer               Modules
──────────────────────────────────────────────────────────────────
Foundation          twinmapper-core
                    twinmapper-definition-model

Code Generation     twinmapper-codegen

Validation          twinmapper-validation

Runtime Base        twinmapper-runtime

Runtime Engines     twinmapper-runtime-binding
                    twinmapper-runtime-objectmap

Format Support      twinmapper-format-json
                    twinmapper-format-yaml
                    twinmapper-format-bpmn

Build Tooling       twinmapper-gradle-plugin
                    twinmapper-maven-plugin

Developer Tooling   twinmapper-cli

Integration         twinmapper-spring-boot-starter
```

---

## Document Engine Architecture

### Build-Time Flow

1. The build plugin scans the configured definition directory for YAML, JSON, and BPMN definition files.
2. The appropriate format reader parses each file into the internal `DefinitionSet` model.
3. The codegen engine consumes the `DefinitionSet` and generates Java sources.
4. Generated sources are written to `build/generated/sources/twinmapper`.
5. Generated and handwritten sources compile together.

### Runtime Flow

1. The consuming application provides a YAML, JSON, or BPMN XML document as input.
2. The format parser (SnakeYAML, Jackson, or JDK StAX) produces a raw parsed structure.
3. The raw structure is adapted into a `NodeCursor` — the format-neutral traversal abstraction.
4. The generated binder is resolved from the `GeneratedBinderRegistry` by target type.
5. The binder reads fields from the `NodeCursor`, applies aliases, defaults, and required-field rules, and constructs the generated DTO.
6. The `BindingResult` carries the populated DTO and any accumulated errors.
7. The generated validator optionally validates the populated DTO.

### NodeCursor Contract

`NodeCursor` is the core runtime abstraction that decouples format parsing from binding logic. Each format module provides its own `NodeCursor` implementation. Generated binders only operate on `NodeCursor` — they have no direct dependency on Jackson, SnakeYAML, or StAX.

---

## Object Engine Architecture

### Build-Time Flow

1. The build plugin scans the configured mapping definition directory for YAML mapping DSL files.
2. The YAML format reader parses mapping files into `ObjectMappingDefinition` entries in the meta-model.
3. The codegen engine generates typed mapper classes for each declared mapping.

### Runtime Flow

1. The consuming application calls `TwinMapperRuntime.map(source, TargetType.class)`.
2. The `TwinObjectMapperRegistry` resolves the correct generated mapper by source and target type.
3. The mapper executes field assignments, converter calls, and nested mapper delegations as plain Java method calls.
4. For UPDATE mode, `BeanWrapper` handles nested property path resolution and JPA proxy transparency.
5. For PATCH mode, null fields in the source are skipped.
6. The populated target object is returned.

### Proxy Safety

All reflection-based operations in the object engine use `AopUtils.getTargetClass()`, `AopProxyUtils.ultimateTargetClass()`, and `ProxyUtils.getUserClass()` to unwrap Spring-managed proxies before inspecting types. This is mandatory — not optional.

---

## Internal Meta-Model

All format readers normalize their inputs into one canonical internal model. This model is the single source of truth for all code generation. It is format-neutral and independent of any domain language.

```
DefinitionSet
├── ObjectTypeDefinition
│   └── FieldDefinition
│       ├── ConstraintDefinition
│       ├── ConditionalConstraintDefinition
│       ├── AliasDefinition
│       ├── DefaultValueDefinition
│       └── DeprecationDefinition / ForbiddenFieldDefinition
├── EnumTypeDefinition
├── ObjectMappingDefinition
│   ├── FieldMappingDefinition
│   └── ProfileDefinition
├── ConverterDefinition
├── VersionDefinition
└── GenerationOptionsDefinition
```

---

## Spring Integration Architecture

TwinMapper integrates with Spring at three levels.

**Auto-configuration level** — `twinmapper-spring-boot-starter` provides `@AutoConfiguration` with `AutoConfiguration.imports`. All TwinMapper beans are conditionally auto-configured. `TwinMapperProperties` via `@ConfigurationProperties(prefix="twinmapper")` is the central configuration contract.

**Conversion level** — The object engine's converter registry integrates with Spring's `ConversionService`. Converters implementing `Converter`, `ConverterFactory`, or `GenericConverter` registered in the Spring application context are available to TwinMapper without additional registration.

**Validation level** — Generated validators wrap as Spring `Validator` implementations. They are compatible with `WebDataBinder`, `LocalValidatorFactoryBean`, and `MethodValidationPostProcessor`. Validation errors produce `BindingResult` that integrates naturally with Spring MVC error handling.

---

## Dependency Flow

```
twinmapper-core
    ↑
twinmapper-definition-model
    ↑
twinmapper-codegen          twinmapper-format-yaml
                            twinmapper-format-json
                            twinmapper-format-bpmn
    ↑                           ↑
twinmapper-gradle-plugin    twinmapper-runtime
twinmapper-maven-plugin         ↑
                        twinmapper-runtime-binding
                        twinmapper-runtime-objectmap
                            ↑
                        twinmapper-validation
                            ↑
                        twinmapper-spring-boot-starter
```

---

## Strictness Architecture

TwinMapper operates in three strictness modes, controlled via `twinmapper.strictness` in `TwinMapperProperties`.

| Mode | Behavior |
|---|---|
| STRICT (default) | Rejects unknown fields, invalid enums, missing required fields, ambiguous mappings, incompatible types without converter |
| COMPATIBLE | Allows deprecated aliases with warnings, permits certain backward-compatible interpretations |
| LENIENT | Migration tooling only; not for production use |

Strictness mode is resolved once at application startup and applies uniformly to all binding and mapping operations.

---

## Security Architecture

Security is enforced at three points.

**BPMN parsing** — `XMLInputFactory.IS_SUPPORTING_EXTERNAL_ENTITIES` and `XMLInputFactory.SUPPORT_DTD` are both set to `false` before any BPMN document is parsed. XXE attacks are prevented at the parser level.

**YAML parsing** — All YAML parsing uses `new Yaml(new SafeConstructor(new LoaderOptions()))`. The default SnakeYAML constructor is never used. Arbitrary Java object deserialization via YAML is prevented.

**Reflection isolation** — Reflection is used only in optional compatibility mode modules. Core binders and mappers use plain generated Java method calls with no reflection. Optional reflection-based helpers are proxy-safe via `AopUtils`/`AopProxyUtils`/`ProxyUtils`.
