# TwinMapper — Grouped Modules for Development

**Overall Progress:** 🟢🟢🟢⚪⚪⚪⚪⚪⚪⚪ 3 of 10 phases complete (30%)

---

## Build-Time Pipeline

### Phase 1 — Foundation ✅ Complete

- `twinmapper-core`
- `twinmapper-definition-model`

### Phase 2 — Definition Readers ✅ Complete

- `twinmapper-format-yaml`
- `twinmapper-format-json`
- `twinmapper-format-bpmn`

### Phase 3 — Code Generation ✅ Complete

- `twinmapper-codegen`

---

## Runtime Platform

### Phase 4 — Runtime Base ⬚ Pending

- `twinmapper-runtime`

### Phase 5 — Runtime Engines ⬚ Pending

- `twinmapper-runtime-binding`
- `twinmapper-runtime-objectmap`

### Phase 6 — Validation ⬚ Pending

- `twinmapper-validation`

---

## Consumption Layer A — Build Consumption

### Phase 7 — Build Plugins ⬚ Pending

- `twinmapper-gradle-plugin`
- `twinmapper-maven-plugin`

### Phase 9 — CLI ⬚ Pending

- `twinmapper-cli`

---

## Consumption Layer B — Application Consumption

### Phase 8 — Spring Boot Starter ⬚ Pending

- `twinmapper-spring-boot-starter`

---

## Optional Extensions

### Phase 10 — Optional Modules ⬚ Pending

- `twinmapper-annotations`
- `twinmapper-annotation-processor`
- `twinmapper-runtime-compat`

---

## Dependency Spine

**Primary:**

`core → definition-model → format-* → codegen → runtime → runtime-binding → runtime-objectmap → validation → spring-boot-starter`

**Orthogonal:**

`codegen → gradle-plugin / maven-plugin / cli`

---

## Progress Summary

| Phase | Section | Modules | Status |
|-------|---------|---------|--------|
| 1 | Build-Time | core, definition-model | ✅ Done |
| 2 | Build-Time | format-yaml, format-json, format-bpmn | ✅ Done |
| 3 | Build-Time | codegen | ✅ Done |
| 4 | Runtime | runtime | ⬚ Pending |
| 5 | Runtime | runtime-binding, runtime-objectmap | ⬚ Pending |
| 6 | Runtime | validation | ⬚ Pending |
| 7 | Build Consumption | gradle-plugin, maven-plugin | ⬚ Pending |
| 8 | App Consumption | spring-boot-starter | ⬚ Pending |
| 9 | Build Consumption | cli | ⬚ Pending |
| 10 | Optional | annotations, annotation-processor, runtime-compat | ⬚ Pending |
