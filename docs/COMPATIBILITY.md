# TwinMapper — Compatibility

## Platform Requirements

| Requirement | Minimum Version |
|---|---|
| Java | 17 |
| Spring Boot | 3.0 |
| Spring Framework | 6.0 |
| Gradle | 7.6 |
| Maven | 3.8 |
| SnakeYAML | 2.0 (bundled in Spring Boot 3.x) |
| Jackson | 2.14 (bundled in Spring Boot 3.x) |

Java 17 is required because TwinMapper generates Java records by default, and the runtime uses sealed interface patterns for binding result types.

Spring Boot 3.x is required because TwinMapper uses `AutoConfiguration.imports` (the Spring Boot 3.x auto-configuration registration mechanism) and `ConfigDataLoader`/`ConfigDataLocationResolver` (introduced in Spring Boot 2.4, required in 3.x form).

---

## Spring Boot Version Compatibility

| Spring Boot | TwinMapper | Notes |
|---|---|---|
| 3.0.x | 1.x | Supported |
| 3.1.x | 1.x | Supported |
| 3.2.x | 1.x | Supported; recommended baseline |
| 3.3.x | 1.x | Supported |
| 2.x | Not supported | Uses legacy `spring.factories` model |

Spring Boot 2.x is not supported because TwinMapper uses `AutoConfiguration.imports` which is the Spring Boot 3.x auto-configuration registration mechanism. `spring.factories` support was deprecated in Spring Boot 2.7 and removed in 3.0.

---

## Jakarta EE vs javax

TwinMapper uses `jakarta.*` namespaces throughout. It is not compatible with `javax.*` (pre-Jakarta) environments.

| Namespace | Status |
|---|---|
| `jakarta.validation` | Required |
| `jakarta.annotation` | Required |
| `javax.*` | Not supported |

---

## GraalVM Native Image Compatibility

TwinMapper is designed with GraalVM native image compatibility in mind.

- All classpath resource access uses Spring's `ResourceLoader` — direct classloader access is not used.
- Generated binders and mappers use plain Java method calls with no runtime reflection.
- The `twinmapper-runtime` module avoids dynamic class loading.
- Optional reflection-based compatibility helpers (`twinmapper.objectmap.reflection-compat.enabled=true`) are not compatible with native image. They must not be enabled in applications compiled to native image.
- The BPMN parser uses JDK StAX which is supported in native image with standard GraalVM reflection configuration.

Native image configuration hints are provided by the starter for the generated binder registry and mapper registry.

---

## Module System (JPMS) Compatibility

TwinMapper-generated code is placed under the `basePackage` configured in generation options. To avoid Java module system split-package issues:

- The `basePackage` must not overlap with any package that exists in a named module on the module path.
- The generated source directory is added as a module-aware source root by both the Gradle and Maven plugins.
- All TwinMapper modules declare proper `module-info.java` descriptors with explicit exports.

---

## Jackson Version Compatibility

TwinMapper uses Jackson's `JsonMapper.builder()` API introduced in Jackson 2.12. The minimum supported Jackson version is 2.14 (bundled in Spring Boot 3.0+).

| Jackson Feature | Minimum Version |
|---|---|
| `JsonMapper.builder()` | 2.12 |
| `@JsonNaming` on records | 2.14 |
| `PropertyNamingStrategies` | 2.12 |

---

## SnakeYAML Version Compatibility

TwinMapper mandates `new Yaml(new SafeConstructor(new LoaderOptions()))`. The `LoaderOptions` class was introduced in SnakeYAML 1.26 and the API stabilized in SnakeYAML 2.0. Spring Boot 3.x bundles SnakeYAML 2.x.

| SnakeYAML | Compatible |
|---|---|
| 2.0+ | Yes |
| 1.x | No — `LoaderOptions` API differs |

---

## BPMN Version Compatibility

TwinMapper's BPMN support covers BPMN 2.0 XML. BPMN 1.x is not supported.

Supported BPMN 2.0 elements are defined in the `twinmapper-format-bpmn` supported vocabulary. BPMN elements not in the supported vocabulary are ignored at runtime and produce a diagnostic at definition-scan time.

| BPMN Version | Status |
|---|---|
| BPMN 2.0 | Supported (supported vocabulary only) |
| BPMN 1.x | Not supported |

---

## OSGi Compatibility

TwinMapper uses Spring's `ResourceLoader` for all resource access, which is OSGi-safe. Direct classloader access is avoided throughout. TwinMapper modules declare OSGi bundle metadata in their manifests.

---

## Compatibility Mode Behavior

TwinMapper's COMPATIBLE strictness mode provides the following backward-compatibility behaviors:

| Behavior | STRICT | COMPATIBLE | LENIENT |
|---|---|---|---|
| Unknown fields | Rejected (error) | Ignored (warning) | Ignored (no warning) |
| Deprecated aliases | Rejected (error) | Accepted (warning) | Accepted (no warning) |
| Forbidden fields | Rejected (error) | Rejected (error) | Rejected (error) |
| Missing required fields | Rejected (error) | Rejected (error) | Rejected (error) |
| Invalid enum values | Rejected (error) | Rejected (error) | Rejected (error) |

Forbidden fields and missing required fields are always errors regardless of strictness mode.

---

## Definition Versioning and Backward Compatibility

When a definition set is versioned, TwinMapper tracks compatibility across versions.

### Non-breaking changes (safe to update without migration)
- Adding optional fields
- Adding new aliases
- Adding new enum values
- Relaxing constraints (increasing max, decreasing min)
- Adding new types to a definition set

### Breaking changes (require migration or version bump)
- Removing required fields
- Renaming fields without adding an alias
- Making optional fields required
- Removing enum values
- Tightening constraints
- Changing field types

### Migration bridge mappers
When `migration.generateBridgeMapper: true` is declared in the definition set, TwinMapper generates bridge mappers between `VersionN` and `VersionN+1` DTOs to support gradual migration.

---

## Integration Compatibility Summary

| Integration | Support Level |
|---|---|
| Spring MVC | Full — `Validator`, `BindingResult`, `WebDataBinder`, `HandlerMethodArgumentResolver` |
| Spring WebFlux | Partial — `Validator` and programmatic binding supported; reactive binding is not a built-in feature |
| Spring Batch | Full — generated validators work with `ItemProcessor` validation flows |
| Spring Data JPA | Full — UPDATE and PATCH mappers handle JPA proxy targets via `BeanWrapper` |
| Spring Security | Compatible — no conflict; TwinMapper does not intercept Spring Security flows |
| Spring Boot Actuator | Full — `InfoContributor` and `HealthIndicator` provided |
| GraalVM Native | Core supported; optional reflection compat mode not supported |
| OSGi | Supported via `ResourceLoader` |
| JPMS | Supported with correct `basePackage` configuration |
