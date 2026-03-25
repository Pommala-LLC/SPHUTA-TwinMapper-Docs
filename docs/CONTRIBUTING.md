# TwinMapper — Contributing Guide

## Overview

This guide covers how to contribute to TwinMapper. It defines the contribution process, code standards, testing requirements, module boundaries, and the review expectations for all changes.

---

## Before Contributing

Before opening a pull request, confirm the following:

- The change is within TwinMapper's defined scope. Review `feature-matrix.md` for the locked Core, Additional, and Optional feature classifications.
- The change does not introduce a Hard Non-Goal behavior. Review the Hard Non-Goals section in `feature-matrix.md`.
- The two Open Implementation Decisions in `decisions.md` are noted. Do not make choices in those areas without prior discussion.
- Breaking changes to generated code contracts require a discussion issue before implementation.

---

## Module Contribution Rules

### Core modules

Changes to `twinmapper-core` and `twinmapper-definition-model` have the highest blast radius. All other modules depend on them. Requirements:

- No Spring container dependency may be introduced in `twinmapper-core`.
- `twinmapper-definition-model` must remain format-neutral. No YAML, JSON, or BPMN parsing belongs here.
- Any change to the internal meta-model types requires updating the codegen engine, all format readers, and documentation.

### twinmapper-runtime

`twinmapper-runtime` must contain only contracts, common utilities, and shared abstractions. No binding logic and no object-mapping implementations may be introduced here. The locked sentence from the architecture decision must be preserved:

> `twinmapper-runtime` is a shared-base runtime module containing contracts, common utilities, and shared abstractions only; it contains no format-specific binding logic and no object-mapping implementations. Those live in `twinmapper-runtime-binding` and `twinmapper-runtime-objectmap`.

### Format modules

Each format module must provide two things and nothing more: a build-time definition reader and a runtime `NodeCursor`-producing parser. Business logic, domain semantics, and cross-format concerns do not belong in format modules.

### Security-sensitive modules

Changes to `twinmapper-format-bpmn` and `twinmapper-format-yaml` that touch parsing code require explicit security review:

- Any change to `XMLInputFactory` configuration must preserve both `IS_SUPPORTING_EXTERNAL_ENTITIES=false` and `SUPPORT_DTD=false`.
- Any change to SnakeYAML usage must preserve `SafeConstructor` with `LoaderOptions`.
- New tests must be added for any changed parsing path verifying XXE rejection and safe construction.

---

## Coding Standards

### Java version

Write Java 17. Use records for immutable value types. Use sealed interfaces for algebraic types where appropriate.

### Generated code

Do not write Lombok annotations on generated types. Generated DTOs must be Lombok-free. Generated binders and mappers must use plain Java method calls with no reflection.

### Spring usage

Use Spring utilities from `spring-core` (`Assert`, `StringUtils`, `ObjectUtils`, `ClassUtils`, `AnnotationUtils`, `ReflectionUtils`) rather than writing custom equivalents. All resource access must use `ResourceLoader` or `PathMatchingResourcePatternResolver`.

### Naming

Follow the naming conventions in `codegen-contract.md`. Do not deviate from the established generated artifact naming patterns without a discussion and update to the contract document.

### Null handling

Annotate all nullable return values and parameters with `@Nullable` from `org.springframework.lang`. Use `@NonNull` for values that must not be null. Constructor validation via `Objects.requireNonNull` is required for required fields on generated records.

---

## Testing Requirements

Every pull request must include tests for the changed behavior. The test requirements by change type are:

| Change Type | Required Tests |
|---|---|
| New definition DSL construct | DSL parsing test, codegen output test, binder behavior test |
| New validation constraint | Constraint positive test, constraint negative test, error message test |
| New object mapping feature | Create, update, and patch mode tests, null policy tests |
| Format module change | Format-specific parsing test, round-trip test |
| Security-sensitive change | XXE rejection test or SafeConstructor enforcement test |
| SPI change | SPI ordering test with multiple registrations |
| Starter/auto-config change | `@SpringBootTest` with `@AutoConfigureTwinMapper` integration test |
| New optional feature | Feature disabled by default test, feature enabled by property test |

### Test coverage requirements

- Unit tests for all new logic paths.
- Integration tests for all Spring context interactions.
- At least one round-trip test for any new binding or mapping capability.
- Security tests for any parser changes.

---

## Pull Request Process

### Branch naming

- `feature/short-description` for new features
- `fix/short-description` for bug fixes
- `docs/short-description` for documentation changes
- `refactor/short-description` for internal restructuring

### Commit messages

Use the following format:

```
[module-name] Short imperative description

Longer explanation if the change is non-obvious. Reference to issue number.
Explain what changed and why, not just what the code does.
```

Examples:
```
[twinmapper-format-bpmn] Disable DTD processing in XMLInputFactory
[twinmapper-codegen] Generate @Qualifier on beans with multiple source/target pairs
[twinmapper-validation] Add minLength/maxLength constraint to generated validators
```

### Pull request description

Every pull request must include:

1. What changed and why
2. Which feature-matrix bucket the change belongs to (Core / Additional / Optional)
3. Test coverage summary
4. Breaking change indicator (yes/no). If yes, describe the migration path.

---

## Documentation Requirements

If the pull request introduces a new feature or changes existing behavior, documentation must be updated. The affected documents depend on the change:

| Change | Documents to Update |
|---|---|
| New DSL construct | `authoring-guide.md`, `features.md`, `glossary.md` |
| New generated artifact type | `codegen-contract.md`, `features.md` |
| New validation constraint | `validation-spec.md`, `error-catalog.md` |
| New object mapping capability | `object-mapping-spec.md`, `features.md` |
| New BPMN element support | `bpmn-support-matrix.md` |
| New optional feature | `feature-matrix.md`, `configuration.md`, `features.md` |
| Security change | `security-model.md` |
| New Spring integration | `configuration.md`, `architecture.md` |
| SPI change | `spi-extension-guide.md` |

---

## What Not to Contribute

The following will not be accepted:

- Reflection-heavy runtime inference as a default behavior
- Silent field guessing in any mode
- Heuristic auto-mapping without explicit configuration
- Domain-specific semantics in `twinmapper-core` or `twinmapper-definition-model`
- Customer-specific extension packs — these belong in separate libraries
- `spring.factories`-based auto-configuration — use `AutoConfiguration.imports`
- Direct `ClassLoader.getResourceAsStream()` calls — use `ResourceLoader`
- Default SnakeYAML constructor usage — use `SafeConstructor`
- XMLInputFactory without XXE disabling

---

## Versioning Policy

TwinMapper follows semantic versioning.

- **Patch release** (x.y.Z): Bug fixes, security patches, documentation corrections. No API changes.
- **Minor release** (x.Y.z): New features, new optional capabilities, new Additional completeness items. Backward compatible.
- **Major release** (X.y.z): Breaking changes to generated code contracts, DSL changes, module API changes.

Changes to the generated code contract (DTO shapes, binder interfaces, mapper interfaces) are always breaking changes and require a major version bump.

---

## Review Criteria

Reviewers will assess pull requests against these criteria:

1. Does the change align with the locked feature classification in `feature-matrix.md`?
2. Does the change preserve the security properties defined in `security-model.md`?
3. Does the generated code remain Lombok-free and reflection-free on the primary path?
4. Is the module boundary respected? (Especially `twinmapper-runtime`'s locked role.)
5. Are tests present and sufficient?
6. Is documentation updated?
7. Does the change preserve determinism? (Same inputs, same outputs.)
