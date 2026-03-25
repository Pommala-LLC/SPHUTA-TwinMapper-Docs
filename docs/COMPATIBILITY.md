# TwinMapper — Compatibility

## Platform Requirements

| Requirement | Minimum Version | Recommended Version |
|---|---|---|
| Java | 21 | 21 (LTS) |
| Spring Boot | 4.0 | 4.0.4 |
| Spring Framework | 7.0 | 7.0.6 |
| Jakarta EE | 11 | 11 |
| Gradle | 8.14 | 9.x |
| Maven | 3.9 | 3.9.x |
| Jackson | 3.0 | 3.1.x |
| SnakeYAML | 2.0 | 2.x (bundled in Spring Boot 4.x) |
| GraalVM (native image) | 25 | 25 |

**Java 21** is the required minimum. TwinMapper generates Java records by default, uses sealed interface patterns for binding result types, and targets Java 21 as the LTS baseline. Java 17 is no longer supported. Spring Boot 4.x raises the Java baseline and Java 21 is the current LTS.

**Spring Boot 4.x** is required. TwinMapper uses `AutoConfiguration.imports` (Spring Boot 3.x+), `ConfigDataLoader`/`ConfigDataLocationResolver` (Boot 2.4+ required in 4.x form), and Spring Framework 7 APIs throughout. Spring Boot 3.x is not supported.

**Jackson 3.x** is required. Spring Boot 4.0 migrated to Jackson 3 as its preferred JSON library. Jackson 3 uses new group IDs and package names — `com.fasterxml.jackson` became `tools.jackson`. TwinMapper targets Jackson 3 throughout. Jackson 2 compatibility bridges exist in Spring Boot 4.x for gradual migration but are not the TwinMapper target.

---

## Spring Boot Version Compatibility

| Spring Boot | TwinMapper | Notes |
|---|---|---|
| 4.0.x | 1.x | Supported; minimum supported line |
| 4.0.4 | 1.x | Latest stable as of March 2026; recommended baseline |
| 4.1.x | 1.x | Supported once GA; milestone releases in progress as of March 2026 |
| 3.x | Not supported | Spring Boot 3.5 EOL: June 2026; use Spring Boot 4.x |
| 2.x | Not supported | EOL; uses legacy `spring.factories` model |

Spring Boot 3.x is not supported. `spring.factories` was deprecated in Spring Boot 2.7 and removed in 3.0. TwinMapper uses `AutoConfiguration.imports` exclusively.

---

## Spring Framework Version Compatibility

| Spring Framework | Spring Boot | TwinMapper |
|---|---|---|
| 7.0.x | 4.0.x | Supported |
| 6.x | 3.x | Not supported |

Spring Framework 7.0 introduces Jakarta EE 11, JSpecify null safety annotations, built-in resilience (retry, concurrency limit), and first-class REST API versioning. TwinMapper targets the Spring Framework 7 API throughout.

---

## Jakarta EE Requirements

TwinMapper uses `jakarta.*` namespaces throughout. It is not compatible with `javax.*` (pre-Jakarta) environments.

| Namespace | Status |
|---|---|
| `jakarta.validation` | Required — Jakarta EE 11 |
| `jakarta.annotation` | Required — Jakarta EE 11 |
| `jakarta.servlet` | Required — Servlet 6.1 baseline |
| `javax.*` | Not supported |

Spring Boot 4.0 is based on Jakarta EE 11 and requires a Servlet 6.1 baseline.

---

## Jackson Version Compatibility

TwinMapper targets Jackson 3.x, which is Spring Boot 4.0's default JSON library.

| Jackson Version | Compatible | Notes |
|---|---|---|
| 3.1.x | Yes | Recommended; Spring Boot 4.1 target |
| 3.0.x | Yes | Minimum supported Jackson 3 line |
| 2.x | Not supported | Jackson 2 is in deprecated bridge mode in Spring Boot 4.x |

**Important:** Jackson 3 changed the package namespace from `com.fasterxml.jackson` to `tools.jackson`. TwinMapper's format module, generated DTO annotations, and `Jackson2ObjectMapperBuilderCustomizer` equivalent all use Jackson 3 APIs exclusively.

| Jackson 3 Feature | Minimum Version |
|---|---|
| `JsonMapper.builder()` | 3.0 |
| `@JsonNaming` on records | 3.0 |
| `PropertyNamingStrategies` | 3.0 |
| `tools.jackson` package namespace | 3.0 |

---

## SnakeYAML Version Compatibility

TwinMapper mandates `new Yaml(new SafeConstructor(new LoaderOptions()))`. Spring Boot 4.x bundles SnakeYAML 2.x.

| SnakeYAML | Compatible |
|---|---|
| 2.0+ | Yes |
| 1.x | No — `LoaderOptions` API differs; not bundled in Spring Boot 4.x |

---

## Gradle Version Compatibility

Spring Boot 4.0 supports Gradle 9, with a minimum of Gradle 8.14 on the 8 line.

| Gradle | Compatible | Notes |
|---|---|---|
| 9.x | Yes | Recommended |
| 8.14+ | Yes | Minimum supported on the 8 line |
| 8.x (below 8.14) | No | Not supported by Spring Boot 4.0 |
| 7.x | No | Not supported |

Gradle 9 removed the convention API that was deprecated since Gradle 7. TwinMapper's Gradle plugin targets the Gradle 9 extension-based API. Legacy convention API calls are not used.

---

## Maven Version Compatibility

| Maven | Compatible | Notes |
|---|---|---|
| 3.9.x | Yes | Recommended |
| 3.8.x | Yes | Minimum supported |
| 3.6.x or earlier | No | Not supported |

---

## GraalVM Native Image Compatibility

TwinMapper is designed with GraalVM native image compatibility in mind.

- All classpath resource access uses Spring's `ResourceLoader` — direct classloader access is not used.
- Generated binders and mappers use plain Java method calls with no runtime reflection.
- The `twinmapper-runtime` module avoids dynamic class loading.
- Optional reflection-based compatibility helpers (`twinmapper.objectmap.reflection-compat.enabled=true`) are not compatible with native image. They must not be enabled in applications compiled to native image.
- The BPMN parser uses JDK StAX which is supported in native image with standard GraalVM reflection configuration.
- GraalVM native image requires v25 or later when building Spring Boot 4.x applications.

Native image configuration hints are provided by the starter for the generated binder registry and mapper registry.

| GraalVM | Compatible |
|---|---|
| 25+ | Yes |
| Earlier | No — Spring Boot 4.x requires GraalVM 25+ for native image |

---

## Module System (JPMS) Compatibility

TwinMapper-generated code is placed under the `basePackage` configured in generation options. To avoid Java module system split-package issues:

- The `basePackage` must not overlap with any package that exists in a named module on the module path.
- The generated source directory is added as a module-aware source root by both the Gradle and Maven plugins.
- All TwinMapper modules declare proper `module-info.java` descriptors with explicit exports.

---

## JSpecify Null Safety

Spring Boot 4.x and Spring Framework 7.x adopt JSpecify annotations for null safety throughout the Spring portfolio. TwinMapper aligns with this.

- Spring's own `@Nullable` and `@NonNull` annotations are deprecated in favour of JSpecify equivalents.
- TwinMapper-generated code uses JSpecify `@Nullable` on optional fields and return types.
- Package-level `@NullMarked` via `package-info.java` establishes null-safe zones in generated packages.

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
| Spring Security 7 | Compatible — no conflict; Spring Security 7.x includes Jackson 3 support |
| Spring Boot Actuator | Full — `InfoContributor` and `HealthIndicator` provided |
| GraalVM Native | Core supported; optional reflection compat mode not supported; GraalVM 25+ required |
| OSGi | Supported via `ResourceLoader` |
| JPMS | Supported with correct `basePackage` configuration |
| Kotlin 2.2 | Compatible — Spring Boot 4.x requires Kotlin 2.2+ if using Kotlin |

---

## Spring Boot EOL Reference

| Spring Boot | EOL Date | Action |
|---|---|---|
| 4.0.x | TBD | Current supported line |
| 3.5.x | June 2026 | Migrate to 4.x before EOL |
| 3.4.x | EOL | Not supported |
| 3.x (other) | EOL | Not supported |
| 2.x | EOL | Not supported |
