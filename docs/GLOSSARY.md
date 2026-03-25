# TwinMapper — Glossary

This glossary defines all terms used in TwinMapper documentation, DSL files, and generated code.

---

## A

**Alias**
An alternative source field name that maps to a canonical target field during document binding. Aliases are declared in the definition DSL under the `aliases` key. Alias resolution is transparent — the generated binder handles it without consumer code changes.

**AliasResolutionPolicy**
A shared runtime contract in `twinmapper-runtime` that defines how alias and fallback field name resolution is performed. Used by both the binding and object mapping runtimes. Part of the optional alias/fallback compatibility resolution feature.

**AnnotationProcessor** (`twinmapper-annotation-processor`)
Optional APT-based annotation processor that reads TwinMapper annotations and bridges them into the internal definition model. Feeds the same codegen pipeline as YAML definitions. Activated via `twinmapper.annotations.enabled=true`.

**AutoConfiguration**
Spring Boot's mechanism for automatically configuring beans when specific conditions are met. TwinMapper's starter uses `@AutoConfiguration` with `AutoConfiguration.imports`. All TwinMapper beans are conditionally registered.

---

## B

**basePackage**
The Java package under which all generated sources are placed. Configured via `twinmapper.generation.base-package` or the Gradle/Maven plugin. Must be unique to avoid Java module system split-package issues.

**BeanWrapper**
A Spring Framework class used in TwinMapper's UPDATE and PATCH mapper implementations. Handles nested property path resolution, type conversion, and JPA-loaded proxy target transparency for Spring-managed bean targets.

**BindingContext**
A runtime object carrying the strictness mode, alias resolution policy, default application policy, and error accumulator for a single document binding operation.

**BindingResult**
The result of a document binding operation. Contains either a successfully populated DTO or a list of accumulated `BindingError` instances with field paths and error codes.

**BPMN (Business Process Model and Notation)**
An XML-based standard for modeling business processes. TwinMapper supports BPMN as a first-class input format for both build-time definitions and runtime document binding. BPMN parsing uses JDK StAX with no third-party BPMN library.

**BpmnDocumentParser**
The StAX-based BPMN XML reader inside `twinmapper-format-bpmn`. Produces a `BpmnIntermediateModel` from an input stream. XXE protection is applied at parser factory instantiation.

**BpmnIntermediateModel**
The TwinMapper-owned typed intermediate representation of a parsed BPMN XML document. Sits between the raw XML and the `NodeCursor` abstraction. Covers the supported BPMN vocabulary only.

---

## C

**CodegenCustomizer**
An SPI that allows post-processing of generated `TypeSpecBuilder` instances before Java source is written. Used for advanced annotation injection or method signature modification.

**COMPATIBLE mode**
A TwinMapper strictness mode that allows deprecated aliases and certain backward-compatible interpretations while still reporting warnings. Intended for migration periods.

**ConditionalConstraintDefinition**
A meta-model type representing a constraint that is only active when a discriminator field equals a specific value. Used for conditional required rules.

**ConditionalGenericConverter**
A Spring SPI interface for converters that need to inspect source and target type descriptors before deciding whether to convert. Used in TwinMapper's object engine for value-object converters where conversion validity depends on generic type parameters.

**ConstraintDefinition**
A meta-model type representing a schema constraint on a field. Covers min, max, minLength, maxLength, pattern/regex, one-of, and mutually exclusive rules.

**ConventionMappingResolver**
An optional SPI active only when `twinmapper.objectmap.convention-mapping.enabled=true`. Resolves target field names from source field names by naming convention. Ambiguity always causes failure — never silent resolution.

**ConversionService**
Spring's central type-conversion API. TwinMapper's object engine uses `ConversionService` as its converter backend in Spring applications. Converters implementing Spring's `Converter`, `ConverterFactory`, or `GenericConverter` SPI contracts registered in the application context are automatically available.

**ConverterDefinition**
A meta-model type declaring a custom converter by name, source type, and target type. Referenced from field mapping definitions in the object mapping DSL.

---

## D

**DefinitionReader**
An SPI that reads definition sources into `DefinitionSet` instances. The built-in readers cover YAML, JSON, and BPMN support definitions. Custom readers can be registered to add support for other formats.

**DefinitionSet**
The top-level container in TwinMapper's internal meta-model. Represents all type definitions, mapping definitions, constraints, profiles, and generation options loaded from one or more definition files with the same name and version.

**DefinitionSource**
The resource reference passed to a `DefinitionReader`. Carries the file location, format hint, and loading context.

**DeprecationDefinition**
A meta-model type recording that a field or alias is deprecated. Carries the deprecation message, replacement field name, and the version from which the deprecation applies.

**DirectFieldAccessor**
A Spring Framework class used in TwinMapper's UPDATE and PATCH mapper implementations for target objects without setter methods, such as value objects with final fields.

**Document Engine**
One of the two primary TwinMapper engines. Responsible for compile-time DTO generation and runtime binding of YAML, JSON, and BPMN documents into generated DTOs.

**DSL (Domain-Specific Language)**
TwinMapper's native YAML-based authoring language for defining types, constraints, mappings, profiles, and converters. Purpose-built for code generation and object mapping. Not JSON Schema.

---

## E

**Enum generation**
The process of generating a Java enum class from an `EnumTypeDefinition` in the internal meta-model. Generated enums include code-to-enum and enum-to-string conversion methods.

**EnumTypeDefinition**
A meta-model type representing an enumeration. Contains a list of `EnumValueDefinition` entries each carrying a name, code, description, and optional deprecation marker.

**Extension pack**
A separate library built on TwinMapper's SPI and extension model. Not part of TwinMapper itself. Examples include domain-specific packs for finance, insurance, workflow, and healthcare. Extension packs are built and owned by customers or partners.

---

## F

**FieldDefinition**
A meta-model type representing a single field within an `ObjectTypeDefinition`. Carries the source name, target name, type reference, required/optional status, aliases, deprecated aliases, default value, and constraints.

**ForbiddenFieldDefinition**
A meta-model type representing a field that must never appear in a source document. Produces a binding error with a migration hint if present regardless of strictness mode.

**Format module**
Any of `twinmapper-format-json`, `twinmapper-format-yaml`, or `twinmapper-format-bpmn`. Each format module provides a definition reader for build-time and a `NodeCursor`-producing parser for runtime.

---

## G

**GeneratedBinderRegistry**
A generated class providing type-keyed lookup of binder instances. Built at code generation time. All registered binders are pre-wired — no runtime class scanning.

**GenerationOptionsDefinition**
A meta-model type carrying per-definition-set code generation options including immutable DTO preference, Spring bean mode, and selective generation toggles.

---

## H

**Hard non-goal**
A feature explicitly excluded from TwinMapper in all forms: uncontrolled live schema learning, silent hidden guessing, runtime-first architecture, and domain-specific semantics in the core platform.

---

## I

**Internal meta-model**
TwinMapper's canonical format-neutral representation of all definition inputs. The single source of truth for all code generation. Defined in `twinmapper-definition-model`.

**Inverse mapping**
A generated mapper that performs the reverse of a declared mapping. Declared via `inverseOf` in the mapping DSL. Build-time failure if inverse generation is ambiguous or unsafe.

---

## L

**LENIENT mode**
A TwinMapper strictness mode intended for migration tooling only. Never for production use. Ignores unknown fields and applies best-effort binding.

---

## M

**MappingContext**
A runtime object carrying the null policy, converter registry reference, mapper registry reference, and error accumulator for a single object mapping operation.

**Metadata descriptor**
A generated class for each object type containing field name constants, alias lists, required field sets, default values, forbidden fields, and definition set metadata.

**Mode (mapping mode)**
The behavioral mode of a generated object mapper. CREATE builds a new target. UPDATE applies source values onto an existing target. PATCH applies only non-null source values onto an existing target.

---

## N

**NodeCursor**
TwinMapper's format-neutral document traversal abstraction. Generated binders operate only on `NodeCursor` — they have no direct dependency on Jackson, SnakeYAML, or StAX. Each format module provides its own `NodeCursor` implementation.

**Null policy**
The behavior applied when a source field value is null during object mapping. `IGNORE_NULLS` skips null source values. `SET_NULLS` propagates null to the target. `FAIL_ON_NULL` throws a `MappingError`.

---

## O

**Object Engine**
One of the two primary TwinMapper engines. Responsible for generating and executing typed Java-to-Java mappers for layer-crossing flows.

**ObjectMappingDefinition**
A meta-model type representing a declared mapping between two Java types. Contains the mapping name, source type, target type, mode, profile reference, field mappings, and inverse declaration.

**ObjectTypeDefinition**
A meta-model type representing a DTO or domain object. Contains field definitions, forbidden fields, and type-level constraints.

---

## P

**PATCH mapper**
A generated mapper that applies only non-null source field values onto an existing target object. Always uses `IGNORE_NULLS` null policy.

**PathMatchingResourcePatternResolver**
Spring's resource resolver that supports Ant-style patterns for scanning definition directories. Used by `twinmapper-runtime-binding` for all classpath resource access.

**ProfileDefinition**
A meta-model type representing a reusable set of mapping defaults. Carries null policy, unmapped target policy, shared converters, and shared ignore rules.

**Proxy-safe reflection**
The practice of using `AopUtils.getTargetClass()`, `AopProxyUtils.ultimateTargetClass()`, or `ProxyUtils.getUserClass()` to unwrap Spring-managed proxies before type inspection. Mandatory for all reflection-based operations in TwinMapper.

---

## R

**ResourceLoader**
Spring's abstraction for loading resources from classpath, filesystem, or URL paths. Mandatory for all resource access in TwinMapper — direct classloader access is not used.

**Round-trip test**
A test that verifies a known valid document produces the expected DTO, and optionally that the DTO can be mapped back to a source object, preserving all field values.

---

## S

**SafeConstructor**
A SnakeYAML constructor that restricts YAML parsing to safe types only. TwinMapper mandates `new Yaml(new SafeConstructor(new LoaderOptions()))` for all YAML parsing to prevent arbitrary Java object instantiation.

**SPI (Service Provider Interface)**
A defined interface that consuming applications or library authors can implement to extend TwinMapper behavior. TwinMapper provides SPIs for definition reading, document parsing, validation extension, value conversion, and code generation customization.

**Spring bean mode**
An optional code generation mode controlled by `twinmapper.generation.spring-bean-mode`. When enabled, `@Component` is emitted on generated mapper and validator beans for Spring application context registration.

**STRICT mode**
TwinMapper's default strictness mode. Rejects unknown fields, invalid enums, missing required fields, ambiguous mappings, and incompatible types without a declared converter.

---

## T

**TwinMapperProperties**
The Spring `@ConfigurationProperties` class bound to the `twinmapper.*` namespace. The central configuration contract for the TwinMapper starter. Contains all global settings and optional feature flags.

**TwinMapperRuntime**
The main runtime facade interface. Provides `readJson`, `readYaml`, `readBpmn`, `map`, `update`, and `patch` methods. Auto-configured as a Spring bean by the starter.

**TwinMapperRuntimeConfigurer**
A Spring-idiomatic programmatic customization interface. Implementing classes can register converters, definition readers, validation extensions, and document parsers. Follows the `WebMvcConfigurer` pattern.

---

## U

**UPDATE mapper**
A generated mapper that applies source field values onto an existing target object. Preserves identity and audit fields unless explicitly mapped. Default null policy: `IGNORE_NULLS`.

---

## V

**ValidationExtension**
An SPI for adding domain-specific validation logic that runs after generated schema validators. Used for cross-field or business-rule validation that cannot be expressed in the schema DSL.

**ValidationReport**
The output of the TwinMapper validation engine. Contains all accumulated validation errors with field paths, error codes, source file references, and rule descriptions.

**VersionDefinition**
A meta-model type carrying definition set version metadata, migration rules, and backward compatibility metadata.

---

## W

**WebApplicationType.NONE**
A Spring Boot setting that prevents the embedded servlet container from starting. Mandatory for TwinMapper CLI applications to avoid starting a web server during command execution.

**XXE (XML External Entity)**
An XML injection attack that exploits DTD external entity processing. TwinMapper prevents XXE by setting `XMLInputFactory.IS_SUPPORTING_EXTERNAL_ENTITIES=false` and `XMLInputFactory.SUPPORT_DTD=false` before parsing any BPMN document.
