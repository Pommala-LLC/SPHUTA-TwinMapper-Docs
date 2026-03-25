# TwinMapper — Architecture Decision Record

This document records every significant architecture and design decision made for TwinMapper, the rationale behind each decision, and the alternatives that were considered and rejected.

---

## ADR-001 — Compile-Time-First Architecture

**Status:** Accepted

**Decision:** TwinMapper generates all DTOs, binders, validators, and mappers at build time from fixed definitions. Nothing is inferred from runtime payloads.

**Rationale:** Runtime inference creates non-deterministic behavior, increases startup time, makes debugging harder, and cannot be verified at compile time. Compile-time generation produces explicit, readable, type-safe code that can be inspected and tested like any other handwritten code.

**Alternatives considered:**
- Runtime reflection-based mapping (ModelMapper style) — rejected because it is non-deterministic, slow to debug, and cannot enforce type safety at compile time.
- Hybrid approach with runtime schema discovery — rejected because it undermines the determinism guarantee and creates behavior that depends on classpath state at runtime.

---

## ADR-002 — TwinMapper-Native YAML DSL

**Status:** Accepted

**Decision:** TwinMapper uses its own YAML DSL rather than JSON Schema or any other existing schema language.

**Rationale:** JSON Schema carries versioning, semantic, and extensibility constraints that conflict with TwinMapper's code generation and object-mapping needs. A native DSL gives full control over syntax evolution, allows object-mapping constructs (modes, profiles, converters) to sit naturally alongside type definitions, and avoids impedance mismatch between the schema language's concepts and TwinMapper's internal meta-model.

**Alternatives considered:**
- JSON Schema — rejected due to versioning complexity, missing object-mapping concepts, and unwanted semantic baggage.
- OpenAPI schema section — rejected because it is API-specification-first rather than code-generation-first.
- Avro / Protobuf IDL — rejected because these are wire-format oriented, not Java-code-generation oriented.

---

## ADR-003 — No Camunda Dependency for BPMN Parsing

**Status:** Accepted

**Decision:** `twinmapper-format-bpmn` uses JDK StAX (`XMLInputFactory`) for BPMN XML parsing and defines its own `BpmnIntermediateModel`. No third-party BPMN library is used.

**Rationale:** Camunda's BPMN Model API, while well-structured, introduces vendor versioning, licensing trajectory dependency, and transitive dependency burden. It also defines the BPMN intermediate model in Camunda's class hierarchy, which would create an impedance mismatch when mapping into TwinMapper's own internal meta-model. Using JDK StAX gives zero vendor dependency and full vocabulary control.

**Alternatives considered:**
- Camunda BPMN Model API — initially evaluated, then rejected after the vendor dependency concern was raised.
- Flowable BPMN parser — rejected for the same reasons as Camunda; also more engine-centered in its public API.
- Apache ODE — rejected as unmaintained.

---

## ADR-004 — Spring ConversionService as Converter Backend

**Status:** Accepted

**Decision:** In Spring applications, TwinMapper's object engine integrates with Spring's `ConversionService` as the converter registry backend.

**Rationale:** Spring applications already maintain a `ConversionService` populated with application-registered converters. Integrating with it avoids duplicating the converter registry and allows converters registered for other Spring purposes to be available in TwinMapper without double-registration.

**Alternatives considered:**
- Separate TwinMapper-owned converter registry — rejected because it would require maintaining two separate registries in Spring applications and creates inconsistency.
- Jackson-based conversion — rejected because Jackson's type conversion is JSON-serialization-oriented, not domain-object-transformation-oriented.

---

## ADR-005 — Spring Validator as the Validation Contract

**Status:** Accepted

**Decision:** Generated validators implement Spring's `Validator` and `SmartValidator` interfaces.

**Rationale:** Spring's `Validator` is the standard validation contract across Spring MVC, Spring Batch, and Spring WebFlux. Implementing it means generated validators are usable in any Spring validation context without additional adapters. It also integrates naturally with `BindingResult`, `WebDataBinder`, and `MethodValidationPostProcessor`.

**Alternatives considered:**
- Custom TwinMapper validator interface — rejected because it would require a custom adapter layer for every Spring validation integration point.
- JSR-380 only — rejected because JSR-380 alone does not integrate cleanly with Spring MVC's `BindingResult` model without `LocalValidatorFactoryBean` wrapping.

---

## ADR-006 — Annotations as Optional Convenience Layer

**Status:** Accepted

**Decision:** TwinMapper annotations (`@TwinMap`, `@TwinField`, `@TwinIgnore`, etc.) are optional and never required. Definition files are the primary configuration mechanism. Annotations can supplement but never replace definition files.

**Rationale:** Annotation-first architectures scatter configuration across many source files, make cross-cutting concerns harder to manage, and create tight coupling between the definition and the generated artifact. Definition files are explicit, centralized, version-controllable, and readable by non-Java authors.

**Alternatives considered:**
- Annotation-only model (MapStruct style) — considered but rejected as the primary model. MapStruct is kept only as the quality benchmark for the object engine, not as an architectural model.
- Pure programmatic registration — valid but verbose for large definition sets. Kept as the second-priority configuration mechanism.

---

## ADR-007 — Single Spring Boot Starter

**Status:** Accepted

**Decision:** TwinMapper ships one starter (`twinmapper-spring-boot-starter`) with conditional bean loading internally, rather than splitting into `twinmapper-spring-boot-starter-binding` and `twinmapper-spring-boot-starter-objectmap`.

**Rationale:** The split starter model adds dependency management complexity for consumers without a clear benefit at initial release. Internal conditional bean registration (`@ConditionalOnProperty`, `@ConditionalOnClass`) achieves the same granularity without requiring consumers to choose between multiple starters. If adoption patterns later justify a split, it can be done without breaking changes.

**Alternatives considered:**
- Split starters per engine — deferred to a future release if adoption patterns justify it.

---

## ADR-008 — Lombok-Free Generated DTOs

**Status:** Accepted

**Decision:** Generated DTOs do not use Lombok annotations. Java records are used for immutable DTOs. Explicit constructors and getters are generated for mutable DTOs.

**Rationale:** Lombok in generated code forces consumers to also have Lombok on their annotation processor path. It also makes generated code less readable and harder to debug. Java records are a standard, Lombok-free way to express immutable data. For mutable classes, explicit code is more transparent.

**Alternatives considered:**
- Lombok @Value or @Data on generated classes — rejected because it introduces a consumer dependency and reduces generated code transparency.
- Immutables library — rejected for the same reasons; it also uses annotation processing which creates circular dependency issues in generated code.

---

## ADR-009 — Three Strictness Modes

**Status:** Accepted

**Decision:** TwinMapper operates in three modes: STRICT (default), COMPATIBLE, and LENIENT. STRICT is always the default.

**Rationale:** Enterprise applications need predictable, explicit behavior. Defaulting to strict mode prevents silent field guessing, unknown field acceptance, and deprecated alias tolerance from becoming the norm. COMPATIBLE mode supports migration periods. LENIENT is for migration tooling only and should never be used in production.

**Alternatives considered:**
- Single strict mode only — rejected because migration periods legitimately require relaxed behavior.
- Permissive default — rejected because it normalizes silent data loss and unexpected field behavior.

---

## ADR-010 — SnakeYAML SafeConstructor Mandate

**Status:** Accepted

**Decision:** All YAML parsing in TwinMapper uses `new Yaml(new SafeConstructor(new LoaderOptions()))`. The default SnakeYAML constructor is never used.

**Rationale:** The default SnakeYAML constructor allows arbitrary Java class instantiation via YAML type tags (`!!com.example.SomeClass`). This is a well-documented remote code execution vector when YAML originates from untrusted sources. Even for trusted sources, the safe constructor prevents accidental exposure.

**Alternatives considered:**
- Jackson YAML backend — evaluated, but SnakeYAML is already bundled in Spring Boot with no additional dependency cost. Jackson YAML adds a separate dependency.

---

## ADR-011 — XXE-Safe BPMN Parsing

**Status:** Accepted

**Decision:** `XMLInputFactory.IS_SUPPORTING_EXTERNAL_ENTITIES` and `XMLInputFactory.SUPPORT_DTD` are both set to `false` before parsing any BPMN document. Both properties must be set — omitting either is a security vulnerability.

**Rationale:** XML External Entity (XXE) injection is one of the OWASP Top 10 vulnerabilities. BPMN XML is a common attack vector because it is often imported from external sources such as modelers, CI pipelines, and partner systems. Disabling external entities and DTD processing at the parser factory level is the correct defense.

---

## ADR-012 — ResourceLoader for All Classpath Access

**Status:** Accepted

**Decision:** All classpath resource access in TwinMapper (definition files, BPMN files, configuration files) uses Spring's `ResourceLoader` and `PathMatchingResourcePatternResolver`, never `ClassLoader.getResourceAsStream()` directly.

**Rationale:** Direct classloader access does not work reliably in OSGi environments, GraalVM native images, and some application server deployments. Spring's `ResourceLoader` is the standard abstraction that handles these environments correctly.

---

## ADR-013 — twinmapper-runtime as Shared-Base Only

**Status:** Accepted

**Decision:** `twinmapper-runtime` contains only contracts, common utilities, and shared abstractions. It contains no format-specific binding logic and no object-mapping implementations.

**Rationale:** Keeping the shared base free of implementation prevents circular dependencies, makes each engine independently testable, and keeps the module dependency graph clean. Consumers who only need one engine do not pull in the other engine's implementation.

---

## ADR-014 — Proxy-Safe Reflection Mandate

**Status:** Accepted

**Decision:** Any reflection-based operation on objects in a Spring application context must use `AopUtils.getTargetClass()`, `AopProxyUtils.ultimateTargetClass()`, or `ProxyUtils.getUserClass()` to unwrap proxies before type inspection.

**Rationale:** Spring extensively uses JDK dynamic proxies and CGLIB proxies for AOP, transaction management, security, and lazy loading. Reflection operations on proxied objects fail silently or return wrong results if performed on the proxy class rather than the target class. This is a correctness requirement, not a performance optimization.

---

## ADR-015 — YAML-Only Object Mapping DSL in V1

**Status:** Accepted

**Decision:** V1 supports only YAML as the mapping DSL format. JSON mapping DSL is deferred.

**Rationale:** Introducing JSON parity for mapping definitions in V1 increases the surface area without adding material capability for most users. YAML is already the primary format for Spring Boot configuration and is the format TwinMapper's schema DSL uses. JSON parity can be added after the YAML DSL is stable.

---

## Open Implementation Decisions

These are not architecture decisions — they are unresolved implementation choices that must be resolved before the relevant modules are implemented.

**OD-001 — Source Writer Tool for twinmapper-codegen**

Must be Spring-neutral and actively maintained. Candidates include Freemarker or Mustache template-based generation using Spring's templating support. The original Square JavaPoet is unmaintained. Micronaut SourceGen's fork has the required maintenance but introduces a Micronaut dependency. Decision pending.

**OD-002 — DefinitionSet Runtime Identity Mechanism**

How each `DefinitionSet` loaded at build time is identified and looked up at runtime must be decided. Candidates: a registry bean populated at startup, a classpath metadata file written by the build plugin, or a Spring `Environment` property. Decision pending.
