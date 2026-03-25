# TwinMapper — Formal Feature Matrix

This is the locked writing baseline for TwinMapper. All seven buckets are explicitly separated: Core, Additional, Optional, Open Implementation Decisions, Hard Non-Goals, External, and Roadmap Beyond V1.

---

## Core

The default TwinMapper product shape. Fourteen core modules, compile-time-first document binding and typed object mapping, Spring-native validation and conversion integration, and `TwinMapperProperties` as a foundational starter contract. `twinmapper-runtime` is explicitly limited to shared contracts, utilities, and abstractions only. Binding and object-mapping implementations live in the two runtime submodules.

| Feature | Bucket | Module / Home | Default | Activation | Notes |
|---|---|---|---|---|---|
| `twinmapper-core` | Core | foundation | Yes | Baseline | Shared foundation |
| `twinmapper-definition-model` | Core | meta-model | Yes | Baseline | Canonical internal model |
| `twinmapper-codegen` | Core | build-time | Yes | Baseline | Build-time generator |
| `twinmapper-validation` | Core | validation | Yes | Baseline | Validation engine |
| `twinmapper-runtime` | Core | shared runtime | Yes | Baseline | Contracts, common utilities, and shared abstractions only |
| `twinmapper-runtime-binding` | Core | runtime | Yes | Baseline | Document binding runtime |
| `twinmapper-runtime-objectmap` | Core | runtime | Yes | Baseline | Object mapping runtime |
| `twinmapper-format-json` | Core | format | Yes | Baseline | JSON support |
| `twinmapper-format-yaml` | Core | format | Yes | Baseline | YAML support |
| `twinmapper-format-bpmn` | Core | format | Yes | Baseline | BPMN support; JDK StAX; no third-party BPMN library |
| `twinmapper-gradle-plugin` | Core | tooling | Yes | Baseline | Gradle integration |
| `twinmapper-maven-plugin` | Core | tooling | Yes | Baseline | Maven integration |
| `twinmapper-cli` | Core | tooling | Yes | Baseline | CLI and developer tooling |
| `twinmapper-spring-boot-starter` | Core | integration | Yes | Baseline | Default Spring Boot integration layer |
| Compile-time-first design | Core | platform-wide | Yes | Baseline | Foundational architecture |
| Document binding engine | Core | binding stack | Yes | Baseline | Fixed-definition binding only |
| Typed object mapping engine | Core | objectmap stack | Yes | Baseline | Typed layer-to-layer mapping flows |
| DTO, binder, validator, registry, and mapper generation | Core | codegen | Yes | Baseline | Generated primary path |
| YAML, JSON, and BPMN support | Core | format modules | Yes | Baseline | All three formats supported |
| Spring-native validation integration | Core | validation | Yes | Baseline | `Validator`, `SmartValidator`, `Errors`, `BindingResult`, `WebDataBinder`, `LocalValidatorFactoryBean`, `MethodValidationPostProcessor`, `@Valid`, `@Validated` |
| Spring-native conversion integration | Core | objectmap / runtime | Yes | Baseline | `ConversionService`, `GenericConversionService`, `FormattingConversionService`, `DefaultConversionService`, `Converter`, `ConverterFactory`, `GenericConverter`, `BeanWrapper`, `DirectFieldAccessor` |
| `TwinMapperProperties` via `@ConfigurationProperties(prefix="twinmapper")` | Core | starter | Yes | Baseline | Foundational starter contract; covers strictness mode, definition locations, generation options, optional feature flags, and compatibility toggles |
| `@EnableConfigurationProperties(TwinMapperProperties.class)` | Core | starter auto-config | Yes | Baseline | Required starter wiring; without this, property binding does not activate |

---

## Additional

Required Spring-grade completeness items for a production-ready baseline. These are not optional product features. They are distributed across the core modules and must be present before implementation is considered complete.

| Feature | Bucket | Module / Home | Default | Activation | Notes |
|---|---|---|---|---|---|
| `AnnotationUtils` | Additional | `twinmapper-core` | Yes | Built-in | Shared reflection helper for optional compat modules |
| `ReflectionUtils` | Additional | `twinmapper-core` | Yes | Built-in | Shared reflection helper |
| `ResourceUtils` | Additional | `twinmapper-core` | Yes | Built-in | Shared resource-loading abstraction |
| `PathMatchingResourcePatternResolver` | Additional | `twinmapper-core` / binding | Yes | Built-in | Ant-style pattern scanning for definition directories |
| `OrderComparator` | Additional | `twinmapper-core` | Yes | Built-in | Deterministic SPI ordering |
| `PriorityOrdered` | Additional | `twinmapper-core` | Yes | Built-in | Deterministic SPI ordering |
| `@Order` | Additional | `twinmapper-core` | Yes | Built-in | Deterministic SPI ordering |
| `Ordered` | Additional | `twinmapper-core` | Yes | Built-in | Deterministic SPI ordering |
| Deterministic `DefinitionSet` runtime identity support | Additional | `twinmapper-definition-model` | Yes | Required | Must be resolved before implementation begins |
| Generated bean disambiguation | Additional | `twinmapper-codegen` | Yes | Generated when needed | `@Qualifier`, `@Primary`, `@Conditional` on generated beans |
| `basePackage` configuration option | Additional | `twinmapper-codegen` | Yes | Config-driven | Aligns generated package structure with consuming application; avoids Java module system split-package issues |
| `ConstraintValidatorFactory` | Additional | `twinmapper-validation` | Yes | Built-in | Enables Spring-injected dependencies in custom JSR-380 constraint validators |
| `HandlerMethodArgumentResolver` | Additional | `twinmapper-validation` / web | Yes | Built-in | Spring MVC method parameter binding integration point |
| `ResponseEntityExceptionHandler` alignment | Additional | `twinmapper-validation` / web | Yes | Built-in | Clean translation of `MethodArgumentNotValidException` and `ConstraintViolationException` |
| `TwinMapperRuntimeConfigurer` | Additional | `twinmapper-runtime` | Yes | SPI | Spring-idiomatic programmatic customization interface following `WebMvcConfigurer` pattern |
| Property keys for optional features | Additional | `TwinMapperProperties` | Yes | Config | All six optional feature keys defined so `@ConditionalOnProperty` has concrete references |
| `Environment`-backed resolution | Additional | `twinmapper-runtime-binding` | Yes | Built-in | Property-backed definition path and placeholder resolution |
| `@ConditionalOnProperty` wiring | Additional | starter / binding / objectmap | Yes | Auto-config | References named optional feature keys in `TwinMapperProperties` |
| `ConditionalGenericConverter` | Additional | `twinmapper-runtime-objectmap` | Yes | Built-in | Required for converters that must inspect generic type parameters before converting |
| Proxy-safe reflection | Additional | `twinmapper-runtime-objectmap` | Yes | Built-in | `AopUtils.getTargetClass()`, `AopProxyUtils.ultimateTargetClass()`, `ProxyUtils.getUserClass()`; mandatory for correctness on Spring-managed proxied beans |
| Clarified `BeanWrapper` usage | Additional | `twinmapper-runtime-objectmap` | Yes | Built-in | Used in UPDATE and PATCH mode; handles nested property paths, type conversion, and JPA-loaded proxy targets |
| Clarified `DirectFieldAccessor` usage | Additional | `twinmapper-runtime-objectmap` | Yes | Built-in | Used when target has no setters, such as value objects with final fields |
| `Jackson2ObjectMapperBuilderCustomizer` | Additional | `twinmapper-format-json` | Yes | Starter wiring | Ensures TwinMapper-generated DTOs and Boot auto-configured `ObjectMapper` share consistent field-naming assumptions |
| `@JsonNaming` | Additional | `twinmapper-format-json` | Yes | Generated when needed | JSON naming alignment for kebab-case to camelCase |
| `@JsonAlias` | Additional | `twinmapper-format-json` | Yes | Generated when needed | JSON alias compatibility on generated DTOs |
| `PropertyNamingStrategies` | Additional | `twinmapper-format-json` | Yes | Built-in | Naming policy alignment |
| `MappingJackson2HttpMessageConverter` | Additional | `twinmapper-format-json` | Yes | MVC integration | Integration point when TwinMapper DTOs are used as Spring MVC REST request or response bodies |
| `JsonMapper.builder()` | Additional | `twinmapper-format-json` | Yes | Built-in | Modern Jackson 2.12+ builder API; preferred over raw `ObjectMapper` construction |
| Secure SnakeYAML construction | Additional | `twinmapper-format-yaml` | Yes | Built-in | `new Yaml(new SafeConstructor(new LoaderOptions()))` mandatory; default constructor allows arbitrary Java object deserialization and is prohibited |
| `YamlPropertiesFactoryBean` | Additional | `twinmapper-format-yaml` | Yes | Reference alignment | Spring Boot YAML loading alignment |
| `YamlMapFactoryBean` | Additional | `twinmapper-format-yaml` | Yes | Reference alignment | Spring Boot YAML loading alignment |
| `ConfigDataLoader` | Additional | `twinmapper-format-yaml` | Yes | Boot integration | Enables `spring.config.import=twinmapper:classpath:/definitions/` in Spring Boot 2.4+ |
| `ConfigDataLocationResolver` | Additional | `twinmapper-format-yaml` | Yes | Boot integration | Paired with `ConfigDataLoader` for `spring.config.import` support |
| XXE-safe `XMLInputFactory` settings | Additional | `twinmapper-format-bpmn` | Yes | Built-in | `XMLInputFactory.IS_SUPPORTING_EXTERNAL_ENTITIES=false` and `XMLInputFactory.SUPPORT_DTD=false`; both required; omitting either is a security vulnerability |
| Mandatory `ResourceLoader` for BPMN | Additional | `twinmapper-format-bpmn` / binding | Yes | Built-in | All BPMN classpath file resolution via `ResourceLoader`; required for OSGi, GraalVM native image, and application server correctness |
| `@ConditionalOnClass` | Additional | starter | Yes | Auto-config | Standard Spring Boot auto-configuration condition |
| `@ConditionalOnMissingBean` | Additional | starter | Yes | Auto-config | References `TwinMapperRuntime` interface type so any custom implementation suppresses the auto-configured one |
| `AutoConfiguration.imports` | Additional | starter | Yes | Boot integration | Mandatory for Spring Boot 3.x and 4.x; `spring.factories` is legacy |
| `spring-configuration-metadata.json` | Additional | starter | Yes | IDE metadata | Under `META-INF/`; provides IDE auto-completion for all `twinmapper.*` properties with descriptions, types, and default values |
| Named TwinMapper application events | Additional | starter / runtime | Yes | Event publication | `TwinMapperBindingSuccessEvent`, `TwinMapperBindingFailureEvent`, `TwinMapperMappingSuccessEvent`, `TwinMapperMappingFailureEvent` |
| Actuator `InfoContributor` | Additional | starter | Yes | Actuator | Exposes registered mapper counts, supported formats, and strictness mode |
| Actuator `HealthIndicator` | Additional | starter | Yes | Actuator | Reports whether all required definition files loaded successfully at startup |
| `@ImportAutoConfiguration(TwinMapperAutoConfiguration.class)` | Additional | starter / test | Yes | Test support | Enables TwinMapper inclusion in Spring Boot test slices |
| `@AutoConfigureTwinMapper` | Additional | starter / test | Yes | Test support | Test-slice opt-in annotation following the `@AutoConfigureMockMvc` pattern |
| `spring-boot-test-autoconfigure` test-scoped dependency | Additional | starter / test | Yes | Test-scoped dep | Required for `@AutoConfigureTwinMapper` to function |
| `@EnableTwinMapper` | Additional | starter | Yes | Explicit activation | For non-Boot or highly controlled Spring contexts where auto-configuration is not used |
| `@EnableTwinMapperBinding` | Additional | starter | Yes | Explicit activation | Selective binding activation in non-Boot contexts |
| `@EnableTwinMapperObjectEngine` | Additional | starter | Yes | Explicit activation | Selective object engine activation in non-Boot contexts |
| CLI boot hardening with `WebApplicationType.NONE` | Additional | `twinmapper-cli` | Yes | Startup config | Prevents embedded servlet container startup during CLI execution |
| `ApplicationRunner` | Additional | `twinmapper-cli` | Yes | CLI runtime | Spring Boot CLI application contract |
| `CommandLineRunner` | Additional | `twinmapper-cli` | Yes | CLI runtime | Spring Boot CLI application contract |
| `ExitCodeGenerator` | Additional | `twinmapper-cli` | Yes | CLI runtime | Process exit code handling |
| `SpringApplication.exit()` | Additional | `twinmapper-cli` | Yes | CLI runtime | Controlled application exit |
| Gradle generated-source registration | Additional | `twinmapper-gradle-plugin` | Yes | Build integration | `sourceSets.main.java.srcDir(generatedDir)` for IDE and compilation |
| Maven generated-source registration | Additional | `twinmapper-maven-plugin` | Yes | Build integration | `project.addCompileSourceRoot(generatedDir)` for IDE and build lifecycle |

---

## Optional

All six optional capabilities are explicit opt-in, property-gated, default `false` where applicable, and must never replace generated binders or mappers as the primary path. Each runtime-gated option has a concrete property key.

| Feature | Bucket | Module / Home | Default | Activation | Notes |
|---|---|---|---|---|---|
| Controlled convention-based mapping mode | Optional | `twinmapper-runtime-objectmap` | No | `twinmapper.objectmap.convention-mapping.enabled=false` | Deterministic opt-in only; ambiguity causes failure, never silent resolution |
| Annotation-heavy usage mode | Optional | `twinmapper-annotations`, `twinmapper-annotation-processor`, starter | No | `twinmapper.annotations.enabled=false` | Convenience mode only; definition files remain primary source of truth |
| Reflection-based compatibility helpers | Optional | `twinmapper-runtime-objectmap` | No | `twinmapper.objectmap.reflection-compat.enabled=false` | For migration and legacy onboarding only; proxy-aware via `AopUtils`/`AopProxyUtils`/`ProxyUtils`; never the default runtime path |
| Bounded runtime discovery helpers | Optional | `twinmapper-runtime-binding` | No | `twinmapper.binding.bounded-discovery.enabled=false` | Adapter-specific; bounded and auditable only; never replaces fixed-definition mode |
| Alias/fallback compatibility resolution | Optional | `twinmapper-runtime-binding`, `twinmapper-runtime-objectmap` | No | `twinmapper.compatibility-resolution.enabled=false` | Policy-controlled compatibility bridge; never hidden guessing |
| Sample-to-definition assistant tooling | Optional | `twinmapper-cli` | No | CLI flag only — not a runtime property | Developer tooling only; requires human review; never automatic live schema learning |

---

## Open Implementation Decisions

These are explicitly not features. They must stay outside Core, Additional, and Optional. Both must be resolved before implementation of their respective modules begins.

| Item | Bucket | Module / Home | Default | Activation | Notes |
|---|---|---|---|---|---|
| Source writer tool choice | Open Decision | `twinmapper-codegen` | N/A | Must decide before implementation | Must be Spring-neutral and actively maintained; candidates include Freemarker/Mustache template-based generation using Spring's templating support; must not introduce a Micronaut dependency |
| `DefinitionSet` runtime identity mechanism | Open Decision | `twinmapper-definition-model` / `twinmapper-runtime` | N/A | Must decide before implementation | Candidates: registry bean, classpath metadata file, or Spring `Environment` property |

---

## Hard Non-Goals

Explicitly excluded from all implementation. Must not appear in any phase or module.

| Item | Bucket | Module / Home | Default | Activation | Notes |
|---|---|---|---|---|---|
| Uncontrolled live schema learning | Hard Non-Goal | None | N/A | Never | Explicitly excluded |
| Silent hidden guessing | Hard Non-Goal | None | N/A | Never | Explicitly excluded |
| Runtime-first architecture | Hard Non-Goal | None | N/A | Never | Compile-time-first is locked |
| Domain-specific semantics in core | Hard Non-Goal | None | N/A | Never | Core remains generic |

---

## External

Outside TwinMapper itself. Enabled by TwinMapper's SPI and extension model but built and owned separately.

| Item | Bucket | Module / Home | Default | Activation | Notes |
|---|---|---|---|---|---|
| Customer-specific domain extension packs | External | Outside TwinMapper | N/A | Separate artifacts | Finance, insurance, workflow, healthcare, and similar packs built by customers or partners using TwinMapper SPIs |

---

## Roadmap Beyond V1

Not part of the current implementation baseline. May be addressed in future releases.

| Item | Bucket | Module / Home | Default | Activation | Notes |
|---|---|---|---|---|---|
| Union/discriminator-capable types | Roadmap Beyond V1 | Future | N/A | Later phase | Deferred until base DSL, codegen, and binder runtime are stable |
| JSON-based object-mapping DSL | Roadmap Beyond V1 | Future | N/A | Later phase | YAML DSL is the only mapping DSL in V1 |
| Arbitrary additional definition DSL formats | Roadmap Beyond V1 | Future | N/A | Later phase | New formats addable via SPI in future releases |
| Annotation-first architecture | Roadmap Beyond V1 | Future | N/A | Later phase | Annotations remain optional convenience only in V1 |

---

## Final Bucket Summary

| Bucket | Meaning |
|---|---|
| Core | Baseline product; present and active by default |
| Additional | Required Spring-grade completeness; must be present before implementation is production-ready |
| Optional | Explicit opt-in; property-gated; default false; must never replace the generated primary path |
| Open Decision | Unresolved implementation choices; not features; must be resolved before the relevant module is implemented |
| Hard Non-Goal | Explicitly excluded from all implementation in any form |
| External | Outside TwinMapper; enabled by SPIs but owned and built separately |
| Roadmap Beyond V1 | Not in scope for current release; may be addressed in future phases |
