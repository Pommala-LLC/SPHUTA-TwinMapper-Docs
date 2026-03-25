# TwinMapper — Configuration Reference

## Overview

TwinMapper is configured through `TwinMapperProperties`, bound via `@ConfigurationProperties(prefix="twinmapper")`. All configuration properties live under the `twinmapper.*` namespace. IDE auto-completion is provided via `spring-configuration-metadata.json` in the starter's `META-INF/` directory.

---

## Configuration Precedence

TwinMapper resolves configuration in this order. Higher entries take precedence.

1. Definition files (YAML/JSON mapping DSL files)
2. Programmatic registration via `TwinMapperRuntimeConfigurer`
3. `application.properties` / `application.yaml` via `TwinMapperProperties`
4. Optional annotations (`@TwinMap`, `@TwinField`, etc.) when annotation mode is enabled
5. Convention-based matching when explicitly enabled

---

## TwinMapperProperties

```yaml
twinmapper:
  strictness: STRICT                          # STRICT | COMPATIBLE | LENIENT
  definition-locations:
    - classpath:/twinmapper/schemas/
    - classpath:/twinmapper/mappings/
  generation:
    base-package: com.example.generated
    immutable-dtos: true
    generate-binders: true
    generate-validators: true
    generate-mappers: true
    generate-metadata: true
    spring-bean-mode: false                   # Emit @Component on generated beans
  objectmap:
    convention-mapping:
      enabled: false                          # Controlled convention mapping opt-in
    reflection-compat:
      enabled: false                          # Reflection-based compatibility helpers
  binding:
    bounded-discovery:
      enabled: false                          # Bounded runtime discovery helpers
  annotations:
    enabled: false                            # Annotation-heavy usage mode
  compatibility-resolution:
    enabled: false                            # Alias/fallback compatibility resolution
```

---

## Property Reference

### Core Properties

| Property | Type | Default | Description |
|---|---|---|---|
| `twinmapper.strictness` | `CompatibilityMode` | `STRICT` | Global strictness mode. STRICT rejects unknown fields, invalid enums, and missing required fields. COMPATIBLE allows deprecated aliases. LENIENT is migration-only. |
| `twinmapper.definition-locations` | `List<String>` | `classpath:/twinmapper/` | Locations to scan for definition and mapping DSL files. Supports Ant-style patterns via `PathMatchingResourcePatternResolver`. |

### Generation Properties

| Property | Type | Default | Description |
|---|---|---|---|
| `twinmapper.generation.base-package` | `String` | (required) | Base Java package for all generated sources. Must align with consuming application package to avoid split-package issues. |
| `twinmapper.generation.immutable-dtos` | `boolean` | `true` | Generate immutable DTOs using Java records by default. Set to `false` for mutable class generation. |
| `twinmapper.generation.generate-binders` | `boolean` | `true` | Generate document binder classes. |
| `twinmapper.generation.generate-validators` | `boolean` | `true` | Generate schema validator classes. |
| `twinmapper.generation.generate-mappers` | `boolean` | `true` | Generate typed object mapper classes. |
| `twinmapper.generation.generate-metadata` | `boolean` | `true` | Generate metadata descriptor classes. |
| `twinmapper.generation.spring-bean-mode` | `boolean` | `false` | When true, emits `@Component` on generated mapper and validator beans for Spring context registration. |

### Optional Feature Properties

All optional features are disabled by default. Enabling any optional feature does not affect the behavior of the generated primary path.

| Property | Type | Default | Description |
|---|---|---|---|
| `twinmapper.objectmap.convention-mapping.enabled` | `boolean` | `false` | Enables controlled convention-based mapping mode. Deterministic only. Ambiguity causes failure — never silent resolution. |
| `twinmapper.objectmap.reflection-compat.enabled` | `boolean` | `false` | Enables reflection-based compatibility helpers for migration and legacy onboarding use cases. Proxy-aware. Never the default runtime path. |
| `twinmapper.binding.bounded-discovery.enabled` | `boolean` | `false` | Enables bounded runtime discovery helpers for adapter-specific use cases. Never replaces fixed-definition mode. |
| `twinmapper.annotations.enabled` | `boolean` | `false` | Enables annotation-heavy usage mode. Definition files remain the primary source of truth. |
| `twinmapper.compatibility-resolution.enabled` | `boolean` | `false` | Enables alias/fallback compatibility resolution for legacy field names and deprecated aliases. Policy-controlled. Never hidden guessing. |

---

## Gradle Plugin Configuration

```groovy
// build.gradle
plugins {
    id 'com.twinmapper.codegen' version '1.0.0'
}

twinmapper {
    schemaDir = 'src/main/twinmapper/schemas'
    mappingDir = 'src/main/twinmapper/mappings'
    outputPackage = 'com.example.generated'
    strictness = 'STRICT'
    immutableRecords = true
    generateMetadata = true
    generateValidators = true
    generateMappers = true
}
```

```kotlin
// build.gradle.kts
plugins {
    id("com.twinmapper.codegen") version "1.0.0"
}

twinmapper {
    schemaDir.set("src/main/twinmapper/schemas")
    mappingDir.set("src/main/twinmapper/mappings")
    outputPackage.set("com.example.generated")
    strictness.set("STRICT")
    immutableRecords.set(true)
    generateMetadata.set(true)
    generateValidators.set(true)
    generateMappers.set(true)
}
```

---

## Maven Plugin Configuration

```xml
<plugin>
    <groupId>com.twinmapper</groupId>
    <artifactId>twinmapper-maven-plugin</artifactId>
    <version>1.0.0</version>
    <executions>
        <execution>
            <goals>
                <goal>generate</goal>
            </goals>
        </execution>
    </executions>
    <configuration>
        <schemaDir>src/main/twinmapper/schemas</schemaDir>
        <mappingDir>src/main/twinmapper/mappings</mappingDir>
        <outputPackage>com.example.generated</outputPackage>
        <strictness>STRICT</strictness>
        <immutableRecords>true</immutableRecords>
        <generateValidators>true</generateValidators>
        <generateMappers>true</generateMappers>
    </configuration>
</plugin>
```

---

## Spring Boot Auto-Configuration

The starter auto-configures TwinMapper when it is present on the classpath. No manual bean registration is required.

```yaml
# application.yaml — minimal configuration
twinmapper:
  strictness: STRICT
  definition-locations:
    - classpath:/twinmapper/
  generation:
    base-package: com.example.generated
```

### Conditional Bean Registration

All TwinMapper beans are registered conditionally.

- `TwinMapperRuntime` — registered via `@ConditionalOnMissingBean(TwinMapperRuntime.class)`. Providing a custom `TwinMapperRuntime` bean suppresses the auto-configured one.
- `TwinObjectMapperRegistry` — registered via `@ConditionalOnMissingBean(TwinObjectMapperRegistry.class)`.
- `GeneratedBinderRegistry` — registered via `@ConditionalOnMissingBean(GeneratedBinderRegistry.class)`.

### Explicit Activation (Non-Boot Contexts)

For non-Boot or highly controlled Spring contexts where auto-configuration is not used, activate TwinMapper explicitly:

```java
@Configuration
@EnableTwinMapper
public class TwinMapperConfig {
    // TwinMapper beans registered manually or via @EnableTwinMapperBinding / @EnableTwinMapperObjectEngine
}
```

---

## Programmatic Configuration

Implement `TwinMapperRuntimeConfigurer` to register custom converters, binders, and validators programmatically. This follows the same pattern as Spring MVC's `WebMvcConfigurer`.

```java
@Configuration
public class TwinMapperCustomConfig implements TwinMapperRuntimeConfigurer {

    @Override
    public void addConverters(ConverterRegistry registry) {
        registry.addConverter(new CustomerIdConverter());
        registry.addConverter(new CustomerStatusConverter());
    }

    @Override
    public void configureStrictness(StrictnessConfigurer configurer) {
        configurer.mode(CompatibilityMode.COMPATIBLE);
    }
}
```

---

## Convention Mapping Configuration (Optional)

Convention-based mapping is disabled by default. When enabled, it applies only when source and target types are strongly compatible. Ambiguity always causes a failure — never silent resolution.

```yaml
twinmapper:
  objectmap:
    convention-mapping:
      enabled: true
      policy: STRICT_NAMES    # STRICT_NAMES | COMPATIBLE_NAMES
      fail-on-ambiguity: true # Always true regardless of setting
```

---

## Compatibility Resolution Configuration (Optional)

```yaml
twinmapper:
  compatibility-resolution:
    enabled: true
    deprecated-field-behavior: WARN    # WARN | ERROR
    legacy-name-resolution: true
    fallback-policy: EXPLICIT_ONLY     # EXPLICIT_ONLY | CONVENTION
```

---

## Definition File Locations

By default, TwinMapper scans `classpath:/twinmapper/` for all definition and mapping files. Custom locations can be configured using Ant-style patterns.

```yaml
twinmapper:
  definition-locations:
    - classpath:/twinmapper/schemas/**/*.yaml
    - classpath:/twinmapper/mappings/**/*.yaml
    - file:config/twinmapper/
```

All resource resolution uses Spring's `PathMatchingResourcePatternResolver`. Direct classloader access is not used.

---

## Environment Variable Substitution

TwinMapper definition files may reference Spring `Environment` properties using standard `${...}` placeholders. Resolution is handled via Spring's `PropertySourcesPlaceholderConfigurer`.

```yaml
definitionSet:
  name: order-model
  version: ${app.version}
  package: ${app.base-package}.generated
```

---

## Actuator Exposure

When Spring Boot Actuator is present, TwinMapper exposes two endpoints.

`/actuator/info` — via `InfoContributor`, exposes registered mapper counts, supported formats, and active strictness mode.

`/actuator/health` — via `HealthIndicator`, reports `UP` when all required definition files loaded successfully at startup, `DOWN` otherwise.

To expose these endpoints, ensure Actuator is configured to include them:

```yaml
management:
  endpoints:
    web:
      exposure:
        include: info, health
  info:
    env:
      enabled: true
```
