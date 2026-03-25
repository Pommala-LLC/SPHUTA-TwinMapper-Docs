# TwinMapper — Release Strategy

## Versioning Model

TwinMapper follows semantic versioning: `MAJOR.MINOR.PATCH`.

| Version Segment | Triggers |
|---|---|
| PATCH (x.y.Z) | Bug fixes, security patches, documentation corrections. No API or generated code contract changes. |
| MINOR (x.Y.z) | New backward-compatible features, new Additional completeness items, new Optional capabilities, new SPI extensions. |
| MAJOR (X.y.z) | Breaking changes to generated code contracts, DSL changes, module API changes, module renames, Spring Boot major version alignment. |

---

## Release Phases

TwinMapper releases follow a phased structure aligned with the development phases in `DevelopmentPhase.md`.

### V1.0 — Core Platform

**Includes:**
- All Core modules (phases 1–9 complete)
- All Additional completeness items
- No Optional features enabled by default
- Two open implementation decisions resolved before release

**V1.0 does not include:**
- Union/discriminator-capable types
- JSON-based object-mapping DSL
- Arbitrary additional DSL formats
- Annotation-first architecture

**V1.0 release criteria:**
- All fourteen core modules compile and integrate
- Full test suite passes including security tests
- Spring Boot 3.2.x integration verified
- `spring-configuration-metadata.json` complete
- `@AutoConfigureTwinMapper` test annotation functional
- Actuator endpoints verified
- GraalVM native image baseline verified
- All two open implementation decisions resolved and documented

---

### V1.x — Optional Feature Releases

Each optional feature is released independently as a minor version once the V1.0 baseline is stable.

| Version | Optional Feature |
|---|---|
| V1.1 | Controlled convention-based mapping mode |
| V1.2 | Alias/fallback compatibility resolution |
| V1.3 | Bounded runtime discovery helpers |
| V1.4 | Annotation-heavy usage mode |
| V1.5 | Reflection-based compatibility helpers |

Sample-to-definition assistant tooling is a CLI enhancement and may ship in any minor version without breaking changes.

---

### V2.0 — Roadmap Beyond V1

V2.0 is the target for the items in the Roadmap Beyond V1 bucket. These are not committed features and will be evaluated based on V1 adoption and community feedback.

Candidates:
- Union/discriminator-capable types
- JSON-based object-mapping DSL
- Additional definition DSL formats (TOML, Protobuf descriptor)
- Annotation-first architecture as an officially supported primary mode

---

## Release Cadence

- **Patch releases:** As needed for bug fixes and security patches. No fixed cadence.
- **Minor releases:** Quarterly or when a significant optional feature is ready.
- **Major releases:** Aligned with Spring Boot major releases. No more than once per year.

---

## Pre-Release Channels

| Channel | Purpose |
|---|---|
| `-SNAPSHOT` | Development builds. Not stable. Not for production. |
| `-RC.n` | Release candidates. Feature-complete. Security-reviewed. Breaking changes frozen. |
| `-M.n` | Milestone builds. Feature preview. API may still change. |
| GA | Generally available. Production-ready. |

---

## Spring Boot Alignment Policy

TwinMapper aligns with the Spring Boot release train.

- When Spring Boot releases a new major version (e.g., 4.0), TwinMapper releases a new major version that updates all Spring dependencies and updates `AutoConfiguration.imports` format if changed.
- TwinMapper supports the current Spring Boot major version and the previous Spring Boot minor stream for 12 months.
- Spring Boot EOL dates are respected. TwinMapper will not maintain compatibility with Spring Boot versions past their EOL date.

---

## Dependency Policy

TwinMapper's core modules have minimal mandatory dependencies. This is by design.

**Mandatory transitive dependencies in the runtime path:**

| Dependency | Version | Reason |
|---|---|---|
| `spring-core` | 6.x+ | Utilities, `ResourceLoader`, `ConversionService` SPI |
| `spring-context` | 6.x+ | Auto-configuration contracts |
| `spring-boot` | 3.x+ | `@ConfigurationProperties`, `AutoConfiguration` |
| `jackson-databind` | 2.14+ | JSON format support |
| `snakeyaml` | 2.x | YAML format support (bundled in Spring Boot) |

**No mandatory runtime dependency on:**
- Any BPMN library (JDK StAX only)
- Lombok
- MapStruct
- Micronaut
- Hibernate Validator (optional; provided if JSR-380 validation is needed)

---

## Generated Code Stability Contract

Generated code is considered a public API for the purposes of versioning. The following are breaking changes that require a major version bump:

- Changing the name of a generated DTO class or field
- Changing the signature of a generated binder method
- Changing the signature of a generated mapper method
- Changing the interface a generated validator implements
- Changing the package structure of generated code
- Removing a generated registry method

The following are not breaking changes:
- Adding new generated classes for new definition constructs
- Adding new fields to generated metadata descriptors
- Adding new methods to generated registry classes
- Internal implementation changes that preserve the same external contract

---

## Security Release Policy

Security releases follow these rules:

- CVEs affecting TwinMapper's direct parsing code (BPMN, YAML) are patched within 7 days of disclosure.
- CVEs in transitive dependencies are patched within 30 days.
- Security patches are released as PATCH versions and backported to the current minor release stream.
- Security advisories are published alongside the patch release.
- No security fix will introduce a behavior change that breaks existing consumers without a documented migration path.

---

## Changelog Format

Each release publishes a changelog following this structure:

```markdown
## [1.2.0] — YYYY-MM-DD

### Added
- Alias/fallback compatibility resolution (optional, `twinmapper.compatibility-resolution.enabled`)
- `AliasResolutionPolicy` SPI for custom alias resolution strategies

### Changed
- `TwinMapperRuntimeConfigurer.addConverters` now accepts `ConditionalGenericConverter` registrations

### Fixed
- Fixed path-aware error reporting for deeply nested list items

### Security
- n/a

### Migration
- n/a

### Deprecated
- n/a

### Removed
- n/a
```

---

## Maven Central Publishing

TwinMapper modules are published to Maven Central under the `com.twinmapper` group ID.

Published artifacts per module:
- `{module}.jar` — compiled classes
- `{module}-sources.jar` — source code
- `{module}-javadoc.jar` — Javadoc
- POM with accurate dependency declarations

All published artifacts are signed with the project GPG key.

---

## Compatibility Promise

TwinMapper commits to the following compatibility promises:

1. Generated code produced by `twinmapper-codegen:1.x` is compatible with `twinmapper-runtime:1.x` at any minor version.
2. `TwinMapperProperties` property keys introduced in `1.x` will not be removed or renamed within the `1.x` series.
3. SPI interfaces introduced in `1.x` will not have methods added without a default implementation within the `1.x` series.
4. The `twinmapper-runtime`'s locked role (shared-base, no binding or mapping logic) will not change within the `1.x` series.
