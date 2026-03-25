# TwinMapper — Security Model

## Overview

TwinMapper enforces security at every point where external data enters the platform: BPMN XML parsing, YAML parsing, and runtime document binding. This document defines the security controls, their enforcement points, and the threats they address.

---

## Threat Model

TwinMapper processes structured documents from potentially untrusted sources: uploaded YAML configuration files, BPMN workflow definitions from external modelers, and JSON API payloads. The primary threats addressed are:

| Threat | Vector | Addressed By |
|---|---|---|
| XML External Entity (XXE) | BPMN XML documents | XMLInputFactory hardening |
| Arbitrary Java object instantiation | YAML type tags | SnakeYAML SafeConstructor |
| Reflection-based classpath attacks | Runtime document parsing | Generated code uses no reflection |
| Unexpected field injection | Unknown fields in input documents | STRICT mode unknown-field rejection |
| Data exfiltration via SSRF | XXE-triggered server-side requests | DTD and external entity disabling |
| Denial of service via deeply nested objects | Malformed documents | Depth limit on NodeCursor traversal |

---

## BPMN XML Security

### XXE Prevention

All BPMN XML parsing uses JDK StAX with the following mandatory settings applied before any document is parsed:

```java
XMLInputFactory factory = XMLInputFactory.newInstance();
factory.setProperty(XMLInputFactory.IS_SUPPORTING_EXTERNAL_ENTITIES, false);
factory.setProperty(XMLInputFactory.SUPPORT_DTD, false);
```

**Both properties must be set.** Omitting either is a security vulnerability.

- `IS_SUPPORTING_EXTERNAL_ENTITIES=false` prevents the parser from resolving external entity references.
- `SUPPORT_DTD=false` prevents the parser from processing DTD declarations entirely, including internal DTD subsets that could be used for entity bombing attacks.

### What XXE Prevents

An XXE attack in a BPMN document could:
- Read arbitrary files from the server filesystem (e.g., `/etc/passwd`, application configuration)
- Trigger Server-Side Request Forgery (SSRF) by referencing internal network addresses
- Cause denial-of-service via billion laughs / XML bomb attacks

TwinMapper's configuration prevents all of these by disabling external entity support and DTD processing at the factory level before any input is consumed.

### Verification

TwinMapper's test suite includes explicit XXE rejection tests. Any CI pipeline for a TwinMapper deployment should verify:

```java
@Test
void shouldRejectXxePayload() {
    String xxePayload = """
        <?xml version="1.0" encoding="UTF-8"?>
        <!DOCTYPE foo [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
        <definitions xmlns="http://www.omg.org/spec/BPMN/20100524/MODEL">
          <process id="&xxe;" />
        </definitions>
        """;

    assertThatThrownBy(() ->
        bpmnParser.parse(new ByteArrayInputStream(xxePayload.getBytes()), ParsingContext.defaults()))
        .isInstanceOf(DocumentParseException.class);
}
```

---

## YAML Security

### SafeConstructor Mandate

All YAML parsing uses `new Yaml(new SafeConstructor(new LoaderOptions()))`. The default SnakeYAML constructor is never used.

The default SnakeYAML constructor allows YAML type tags to instantiate arbitrary Java classes:

```yaml
# This MUST be rejected — arbitrary class instantiation
!!com.example.SensitiveClass
  field: value
```

With `SafeConstructor`, only safe YAML types are permitted: strings, integers, booleans, lists, and maps. Type tags are rejected with a `YAMLException`.

### What This Prevents

- Remote code execution via YAML deserialization gadget chains
- Classpath scanning attacks via YAML type instantiation
- Loading of sensitive configuration classes from the application context

### Verification

```java
@Test
void shouldRejectArbitraryClassInstantiation() {
    String maliciousYaml = "!!com.example.SensitiveClass\n  secret: value";

    assertThatThrownBy(() -> yamlParser.parse(maliciousYaml))
        .isInstanceOf(YAMLException.class);
}
```

---

## Runtime Binding Security

### No Reflection in Generated Code

Generated binders use plain Java method calls to read fields from `NodeCursor`. There is no runtime reflection in the binding path. This eliminates the following attack surfaces:

- Reflection-based classpath scanning
- `Class.forName()` injection via crafted field values
- Method handle attacks

### Unknown Field Rejection (STRICT mode)

In STRICT mode (the default), any field present in the input document that is not declared in the definition set produces a `UNKNOWN_FIELD` error and the binding fails. This prevents:

- Parameter injection attacks via unexpected fields
- Silent data acceptance that bypasses business validation

### Forbidden Field Rejection

Fields declared as `forbiddenFields` in the definition produce a `FORBIDDEN_FIELD` error regardless of strictness mode. This is an unconditional rejection that cannot be suppressed.

### Depth Limit

`NodeCursor` traversal enforces a configurable depth limit (default: 64 levels). Documents with excessive nesting beyond this limit produce a `BINDING_DEPTH_EXCEEDED` error. This prevents stack overflow attacks via deeply nested JSON or YAML.

---

## Object Mapping Security

### Proxy-Safe Reflection

Optional reflection-based compatibility helpers (`twinmapper.objectmap.reflection-compat.enabled=true`) use proxy-safe type inspection:

```java
Class<?> targetClass = AopUtils.getTargetClass(target);
```

Direct `target.getClass()` is never used on Spring-managed beans. This prevents reflection from operating on proxy classes, which could expose proxy internals or bypass Spring AOP security interceptors.

### Converter Isolation

Converters run within the mapping operation's `MappingContext`. They cannot access the Spring application context directly. Converters that require Spring dependencies must be registered as Spring beans and injected via `TwinMapperRuntimeConfigurer`, not obtained reflectively.

---

## Resource Loading Security

### No Direct ClassLoader Access

All resource loading in TwinMapper uses Spring's `ResourceLoader` and `PathMatchingResourcePatternResolver`. Direct `ClassLoader.getResourceAsStream()` calls are not used. This is a defense-in-depth measure that prevents path traversal attacks in environments where the classloader's resource base is restricted.

### ResourceLoader Location Validation

Definition file locations configured via `twinmapper.definition-locations` are resolved using Spring's `PathMatchingResourcePatternResolver`. Only `classpath:`, `classpath*:`, and `file:` URL schemes are accepted. HTTP URLs are not supported for definition file loading and will produce a `DEFINITION_LOAD_ERROR`.

---

## Generated Code Security Properties

Generated DTOs, binders, validators, and mappers uphold these security properties:

| Property | Guarantee |
|---|---|
| No reflection | Generated code uses plain Java method calls only |
| No dynamic class loading | Generated code does not call `Class.forName()` or similar |
| No eval-style expression execution | Generated code does not evaluate dynamic expressions at runtime |
| Immutable DTOs | Records cannot be mutated after construction (no setters) |
| Null safety | Required fields produce `NullPointerException` if null is passed to the constructor |

---

## Optional Feature Security Considerations

### Convention-Based Mapping (`twinmapper.objectmap.convention-mapping.enabled=true`)

Convention mapping resolves field names by naming convention. Ambiguous resolutions always fail — they never silently map to the wrong field. This prevents incorrect data routing that could lead to authorization bypass or data leakage between domains.

### Reflection-Based Compatibility Helpers (`twinmapper.objectmap.reflection-compat.enabled=true`)

Reflection-based helpers are proxy-safe but do involve runtime reflection. Consider the following when enabling:

- Never enable in GraalVM native image builds
- Ensure the reflecting code path is not reachable from untrusted input
- Use `twinmapper-runtime-compat` as a separate module so the reflection surface is isolated

### Alias/Fallback Resolution (`twinmapper.compatibility-resolution.enabled=true`)

Alias resolution is policy-controlled and visible. It does not perform hidden field name guessing. All alias resolutions are declared explicitly in the definition DSL. There is no risk of accepting unintended field names through alias resolution.

---

## Dependency Security

TwinMapper follows these rules for its own dependencies:

- No third-party BPMN library is used — JDK StAX only, no vendor dependency.
- SnakeYAML is used in its safe mode only.
- Jackson is used without enabling polymorphic type deserialization by default.
- No eval-capable expression libraries (SpEL, OGNL, MVEL) are used in the core runtime path.

Consumers should regularly audit TwinMapper's transitive dependencies using their dependency management toolchain.

---

## Security Checklist for Deployment

Before deploying an application that uses TwinMapper, verify the following:

- [ ] BPMN parsing: `IS_SUPPORTING_EXTERNAL_ENTITIES` is `false`
- [ ] BPMN parsing: `SUPPORT_DTD` is `false`
- [ ] YAML parsing: `SafeConstructor` is used — not default constructor
- [ ] Strictness mode is `STRICT` in production
- [ ] Reflection compat mode is disabled unless explicitly required
- [ ] Definition file locations do not include user-supplied HTTP URLs
- [ ] NodeCursor depth limit is configured appropriately for the expected document depth
- [ ] Optional features are audited: only those actively needed are enabled
- [ ] Generated code does not include `@SuppressWarnings("unchecked")` on security-sensitive paths
