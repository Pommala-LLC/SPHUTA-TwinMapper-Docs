# TwinMapper — Frequently Asked Questions

---

## General

### What is TwinMapper?

TwinMapper is a compile-time-first, schema-driven code generation and mapping platform for Java. It generates strongly typed DTOs, enums, binders, validators, metadata, and typed object mappers from fixed YAML, JSON, and BPMN definitions. At runtime it binds actual documents into generated DTOs and executes typed layer-to-layer object mappers without reflection-heavy inference.

---

### How is TwinMapper different from MapStruct?

MapStruct generates compile-time object mappers from annotated Java interfaces. TwinMapper does that and more. The key differences:

- TwinMapper is definition-file-driven first. Annotations are optional convenience, not the primary model.
- TwinMapper also generates DTOs, enums, binders, and validators from schema definitions — MapStruct does none of this.
- TwinMapper supports runtime document binding for YAML, JSON, and BPMN XML — MapStruct has no document binding concept.
- MapStruct is the quality benchmark for TwinMapper's object engine. It is not a dependency.

---

### How is TwinMapper different from ModelMapper?

ModelMapper uses runtime reflection and convention-based heuristics to map between objects. TwinMapper is the opposite: all mapping logic is generated at build time from explicit definitions, uses plain Java method calls at runtime, and fails loudly on ambiguity rather than guessing silently. ModelMapper-style heuristic auto-mapping is explicitly excluded from TwinMapper's core design.

---

### Does TwinMapper require Spring Boot?

No. The core modules (`twinmapper-core`, `twinmapper-definition-model`, `twinmapper-codegen`, `twinmapper-runtime`) are framework-agnostic. `twinmapper-spring-boot-starter` provides the Spring Boot integration, but it is not required. In non-Boot Spring applications, use `@EnableTwinMapper`, `@EnableTwinMapperBinding`, or `@EnableTwinMapperObjectEngine` for explicit activation.

---

### What Java version is required?

Java 17 minimum. TwinMapper generates Java records for immutable DTOs and uses sealed interface patterns internally. Java 17 is also required by Spring Boot 3.x, which is TwinMapper's Spring Boot integration target.

---

### Is TwinMapper production-ready?

V1.0 targets production readiness. The test suite includes security tests (XXE, SnakeYAML safe construction), round-trip tests for all formats, and Spring Boot integration tests. See `testing-strategy.md` for the full test coverage model.

---

## Configuration

### Where do I put definition files?

By default, TwinMapper scans `classpath:/twinmapper/schemas/` for schema definitions and `classpath:/twinmapper/mappings/` for object mapping definitions. This translates to `src/main/resources/twinmapper/schemas/` and `src/main/resources/twinmapper/mappings/` in a standard Maven/Gradle project. Custom locations are configured via `twinmapper.definition-locations`.

---

### What is the difference between schema definitions and mapping definitions?

Schema definitions declare types, fields, enums, and constraints. They drive DTO, binder, validator, and metadata generation. Mapping definitions declare how Java objects are mapped between layers — entity to domain, domain to DTO, etc. They drive object mapper generation. Both use the same TwinMapper-native YAML DSL but serve distinct purposes.

---

### How do I configure TwinMapper in application.yaml?

```yaml
twinmapper:
  strictness: STRICT
  definition-locations:
    - classpath:/twinmapper/schemas/
    - classpath:/twinmapper/mappings/
  generation:
    base-package: com.example.generated
    immutable-dtos: true
    generate-validators: true
    generate-mappers: true
```

All properties are documented with IDE auto-completion via `spring-configuration-metadata.json` in the starter.

---

### How do I enable optional features?

Each optional feature has an explicit property key. All are `false` by default.

```yaml
twinmapper:
  objectmap:
    convention-mapping:
      enabled: true   # controlled convention mapping
    reflection-compat:
      enabled: true   # reflection compatibility helpers
  binding:
    bounded-discovery:
      enabled: true   # bounded runtime discovery
  annotations:
    enabled: true     # annotation-heavy usage mode
  compatibility-resolution:
    enabled: true     # alias/fallback compatibility
```

Sample-to-definition assistant is CLI-only and does not have a runtime property.

---

### Can I use TwinMapper without the Gradle or Maven plugin?

The Gradle and Maven plugins are the standard way to integrate code generation into the build lifecycle. Without a plugin, you would need to invoke `twinmapper-codegen` programmatically and manage the generated source directory manually. This is not recommended for production use. The CLI (`twinmapper-cli`) can be used for validation and dry-run inspection without the build plugins.

---

## Document Binding

### Which document formats are supported?

YAML, JSON, and BPMN 2.0 XML. See `bpmn-support-matrix.md` for the supported BPMN vocabulary.

---

### What happens if the input document has unknown fields?

In STRICT mode (the default), unknown fields cause a `UNKNOWN_FIELD` binding error and binding fails. In COMPATIBLE mode, unknown fields produce a warning and are ignored. In LENIENT mode, unknown fields are silently ignored. LENIENT is for migration tooling only.

---

### How do aliases work?

Aliases are alternative source field names that map to the canonical target field. Declare them in the definition DSL:

```yaml
fields:
  name:
    type: string
    required: true
    aliases: [productName, product_name]
```

When binding, if `name` is absent, TwinMapper tries `productName`, then `product_name`. The first match wins. Alias resolution is transparent — no consumer code changes are needed.

---

### How are defaults applied?

Defaults are applied at bind time by the generated binder. If an optional field is absent from the input document, the declared default value is used to populate the DTO field. Defaults are not applied by the validator — they are already in place by the time validation runs.

---

### Can I bind the same document into multiple DTO types?

Yes. The `GeneratedBinderRegistry` provides binders for all generated DTO types. Call `runtime.readYaml(input, TypeA.class)` and `runtime.readYaml(input, TypeB.class)` on the same input if needed. Note that the input stream is consumed on the first read — buffer it or re-open the stream for multiple bindings.

---

### Does TwinMapper support streaming or large document processing?

TwinMapper's runtime binding loads the full document into a `NodeCursor` tree before binding begins. It does not support streaming binding of very large documents. For documents larger than a few megabytes, consider pre-processing or chunking at the application level.

---

### Can I use TwinMapper for JSON REST API request binding?

Yes. The generated binder can bind JSON from any `InputStream`. In Spring MVC, use `runtime.readJsonSafe(request.getInputStream(), MyRequestDto.class)` in your controller. The generated validator integrates with Spring's `BindingResult`. See `examples.md` Example 2 for a full Spring MVC example.

---

## Object Mapping

### What mapping flows are supported?

All thirteen flows are first-class supported: DTO↔DTO, Entity↔Domain, Domain↔Entity, Domain↔DTO, DTO↔Domain, Entity↔DTO, DTO↔Entity, Entity↔Projection, Projection↔DTO, Request/Command↔Domain, Domain↔Response/View/Event, Patch DTO↔Entity, Patch DTO↔Domain.

---

### What is the difference between UPDATE and PATCH mapping?

UPDATE applies all source field values onto an existing target. Null source fields leave the target field unchanged (IGNORE_NULLS default). PATCH applies only non-null source fields onto an existing target. The null policy for PATCH is always IGNORE_NULLS and cannot be changed. Use PATCH for HTTP PATCH semantics where the consumer provides only the fields they want to change.

---

### How do I map a nested source field to a flat target field?

Use dotted path notation in the mapping definition:

```yaml
fields:
  - source: address.city
    target: city
  - source: address.postcode
    target: postalCode
```

The generated mapper reads `source.address().city()` and assigns it to `target.setCity(...)`.

---

### How do I convert between value objects?

Declare a named converter in the mapping definition and register its implementation:

```yaml
fields:
  - source: id
    target: id
    converter: longToCustomerId
```

```java
@Configuration
public class ConverterConfig implements TwinMapperRuntimeConfigurer {
    @Override
    public void addConverters(ConverterRegistry registry) {
        registry.addConverter(Long.class, CustomerId.class,
            id -> id != null ? new CustomerId(id) : null);
    }
}
```

---

### Can I generate inverse mappers automatically?

Yes. Declare `inverseOf` in the mapping DSL:

```yaml
- name: CustomerDomainToEntity
  inverseOf: CustomerEntityToDomain
  mode: UPDATE
```

TwinMapper generates the reverse mapper if the mapping is unambiguous and safe. Flattening, value object conversions without a declared reverse converter, or one-to-many relationships cause a build-time `INVERSE_MAPPING_UNSAFE` error. In those cases, write the inverse mapping explicitly.

---

### Does the object engine work with JPA entities?

Yes. UPDATE and PATCH mappers use `BeanWrapper` which handles JPA-loaded proxy targets transparently. Proxy-safe type inspection via `AopUtils.getTargetClass()` ensures reflection-based operations work correctly on Spring-managed proxied entities. Audit fields (createdAt, createdBy, version) can be excluded from UPDATE mappings via profile `ignoreTargets`.

---

### Can I use TwinMapper with Spring Data projections?

Yes. Entity-to-Projection mapping is a first-class supported flow. Generate a projection DTO from your schema definitions and declare an entity-to-projection mapping in the mapping DSL. The generated mapper copies the declared fields from the entity into the projection DTO.

---

## Validation

### What constraints are supported in V1?

Required, optional, default, enum validity, forbidden fields, deprecated fields, conditional required rules, one-of rules, mutually exclusive rules, min, max, minLength, maxLength, regex/pattern, unknown field checks, and collection/map type checks. See `validation-spec.md` for the complete list.

---

### Can I add custom validation logic?

Yes. Implement `ValidationExtension` and register it as a Spring bean. It runs after all generated schema validators. Use this for cross-field rules, business semantics, and domain-specific checks that cannot be expressed in the schema DSL.

---

### How do generated validators integrate with Spring MVC?

Generated validators implement `SmartValidator`. Register them with a `WebDataBinder` or use `@Valid` on controller method parameters. Error messages accumulate into `BindingResult`. `ResponseEntityExceptionHandler` handles `MethodArgumentNotValidException` and `ConstraintViolationException` translation.

---

### Does TwinMapper support JSR-380 Bean Validation annotations?

Yes. Generated validators work alongside JSR-380. `LocalValidatorFactoryBean` wraps JSR-380 for Spring integration. `ConstraintValidatorFactory` enables Spring-injected dependencies in custom constraint validators. Use `@Valid` and `@Validated` as normal — TwinMapper validators and JSR-380 validators compose naturally.

---

## BPMN

### Which BPMN version is supported?

BPMN 2.0 only. BPMN 1.x is not supported.

---

### Does TwinMapper depend on Camunda or Flowable for BPMN parsing?

No. TwinMapper uses JDK StAX only. There is no Camunda or Flowable dependency anywhere in the BPMN format module. TwinMapper defines its own `BpmnIntermediateModel` covering the supported vocabulary.

---

### What BPMN elements are supported?

Process, SubProcess, CallActivity, all Task variants, all Gateway variants, Start/End/Intermediate events with their event definitions (message, timer, signal, error, conditional, escalation, link, terminate), SequenceFlow, BoundaryEvent, extension elements, and LaneSet for metadata. See `bpmn-support-matrix.md` for the full matrix.

---

### What happens when a BPMN document contains unsupported elements?

Unsupported elements are silently ignored at runtime and produce a WARNING diagnostic at build-time scan. Binding does not fail. Only supported elements contribute to the generated DTO.

---

### Is BPMN parsing XXE-safe?

Yes. `XMLInputFactory.IS_SUPPORTING_EXTERNAL_ENTITIES` and `XMLInputFactory.SUPPORT_DTD` are both set to `false` before any BPMN document is parsed. Both settings are mandatory — omitting either is a security vulnerability. TwinMapper's test suite includes explicit XXE rejection tests.

---

## Security

### Is YAML parsing safe from deserialization attacks?

Yes. All YAML parsing uses `new Yaml(new SafeConstructor(new LoaderOptions()))`. The default SnakeYAML constructor is never used. Arbitrary Java class instantiation via YAML type tags (`!!com.example.SomeClass`) is rejected.

---

### Can TwinMapper be used with GraalVM native image?

Core TwinMapper is designed with native image in mind. Generated binders and mappers use plain Java method calls — no reflection in the primary path. Spring's `ResourceLoader` is used for all resource access. The optional reflection-based compatibility helpers (`twinmapper.objectmap.reflection-compat.enabled=true`) are not compatible with native image and must not be enabled in native image builds.

---

## Spring Integration

### Does TwinMapper work without Spring Boot auto-configuration?

Yes. For non-Boot Spring applications, use `@EnableTwinMapper` (full activation), `@EnableTwinMapperBinding` (document engine only), or `@EnableTwinMapperObjectEngine` (object engine only). These annotations provide explicit Spring context activation without auto-configuration.

---

### How does TwinMapper integrate with Spring's ConversionService?

In Spring applications, TwinMapper's object engine uses Spring's `ConversionService` as the converter backend. Converters implementing `Converter<S,T>`, `ConverterFactory`, or `GenericConverter` registered in the Spring application context are automatically available to TwinMapper without additional registration. This avoids maintaining two separate converter registries.

---

### Can I override TwinMapper's auto-configured beans?

Yes. `@ConditionalOnMissingBean(TwinMapperRuntime.class)` means providing your own `TwinMapperRuntime` bean suppresses TwinMapper's auto-configured one. The same applies to `TwinObjectMapperRegistry` and `GeneratedBinderRegistry`. This allows full replacement of any auto-configured component.

---

### Does TwinMapper expose Spring Boot Actuator endpoints?

Yes. When Actuator is on the classpath, `InfoContributor` exposes registered mapper counts, supported formats, and active strictness mode at `/actuator/info`. `HealthIndicator` reports whether all required definition files loaded successfully at startup at `/actuator/health`.

---

### How do I test TwinMapper integration in a Spring Boot test slice?

Use `@AutoConfigureTwinMapper` on your test class. This activates TwinMapper's auto-configuration without requiring a full application context. It follows the same pattern as `@AutoConfigureMockMvc`.

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.NONE)
@AutoConfigureTwinMapper
class MyBindingTest {
    @Autowired
    TwinMapperRuntime runtime;
    // ...
}
```

---

## Extensibility

### Can I add support for a new document format?

Yes. Implement `DefinitionReader` for build-time definition loading and `RuntimeDocumentParser` for runtime document parsing. Register both as Spring beans or via `TwinMapperRuntimeConfigurer`. The SPI model supports any format — TOML, Protobuf descriptors, and XML are potential future additions.

---

### Can I build a domain-specific extension pack?

Yes. Extension packs are separate libraries built using TwinMapper's SPI and extension model. They register `DefinitionReader`, `ValidationExtension`, and `ValueConverter` implementations via Spring auto-configuration. TwinMapper provides the SPI contracts. Domain packs for finance, insurance, workflow, healthcare, and similar domains are built and owned outside TwinMapper.

---

### Can I customize what annotations are emitted on generated code?

Yes. Implement `CodegenCustomizer` and register it in the Gradle or Maven plugin configuration. `CodegenCustomizer` receives the `TypeSpecBuilder` before source is written and can add annotations, modify method signatures, or inject imports.

---

## Troubleshooting

### My generated code is not being picked up by the IDE.

Ensure the generated source directory is registered as a source root. For Gradle, `sourceSets.main.java.srcDir(generatedDir)` handles this. For Maven, `project.addCompileSourceRoot(generatedDir)` handles this. Both are set by the respective plugins. In IntelliJ IDEA, reimport the project or run "Reload All Gradle Projects" after the first build.

---

### I am getting UNKNOWN_FIELD errors for fields I know are declared.

Check the following in order:
1. The field name in the input document matches the declared name or a declared alias exactly (case-sensitive).
2. The definition file is in the configured scan location and was processed in the last build.
3. The generated binder was recompiled after the definition change — run a clean build.
4. The strictness mode is STRICT — if you want to allow unknown fields temporarily, switch to COMPATIBLE while investigating.

---

### My inverse mapping fails at build time with INVERSE_MAPPING_UNSAFE.

The mapping contains a field transformation that cannot be safely reversed. Common causes:
- Flattening without a declared reverse expansion: `address.city → city` has no declared reverse.
- Value object conversion without a declared reverse converter: `Long → CustomerId` needs a `CustomerId → Long` converter declared.
- Write the inverse mapping explicitly in the DSL rather than using `inverseOf`.

---

### BeanWrapper is throwing exceptions on JPA entity update.

Ensure the entity is not detached before the update mapper runs. `BeanWrapper` requires the entity to be in a managed or attached state for Hibernate proxy resolution to work correctly. Load or reattach the entity before calling the update mapper.

---

### Convention mapping is not resolving fields as expected.

Convention mapping is disabled by default. Enable it via `twinmapper.objectmap.convention-mapping.enabled=true`. When enabled, ambiguous matches always cause a failure — never silent resolution. If two source fields match one target field with equal confidence, declare the mapping explicitly in the DSL.

---

### The CLI is starting an embedded web server.

Ensure `SpringApplication.setWebApplicationType(WebApplicationType.NONE)` is set before `SpringApplication.run(...)` in the CLI entry point. The TwinMapper CLI auto-configuration applies this, but a custom entry point may override it. Alternatively, set `spring.main.web-application-type=none` in the CLI's `application.properties`.
