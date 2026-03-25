# TwinMapper — Error Catalog

## Overview

Every error produced by TwinMapper carries a structured payload including an error code, field path, source file reference, and contextual message. This document catalogs all error codes, their triggers, their severity, and resolution guidance.

---

## Error Structure

Every TwinMapper error includes the following fields:

| Field | Type | Description |
|---|---|---|
| `errorCode` | `String` | Machine-readable error identifier |
| `severity` | `ErrorSeverity` | ERROR, WARNING, or INFO |
| `fieldPath` | `String` | Dot-separated path to the offending field, e.g. `workflow.states[2].trigger.type` |
| `sourceFile` | `String` | Definition or input file name where the error originated |
| `line` | `int` | Line number (0 if not available from parser) |
| `column` | `int` | Column number (0 if not available from parser) |
| `targetType` | `String` | Fully qualified Java type being generated or bound into |
| `ruleViolated` | `String` | Human-readable description of the constraint that failed |
| `migrationHint` | `String` | Optional guidance for forbidden or deprecated field errors |
| `expectedValue` | `String` | Expected type or value when a type mismatch occurs |
| `actualValue` | `String` | Actual type or value received |

---

## Error Severity

| Severity | Behavior |
|---|---|
| `ERROR` | Binding, mapping, or generation fails. No result is produced. |
| `WARNING` | Processing continues but the issue is reported. Result is produced. |
| `INFO` | Informational only. No action required. |

---

## Build-Time Errors (Code Generation)

These errors occur during the code generation phase in the Gradle or Maven build.

### DEFINITION_PARSE_ERROR

**Severity:** ERROR
**Trigger:** Definition file contains invalid YAML or JSON syntax that cannot be parsed.
**Message pattern:** `Failed to parse definition file '{file}' at line {line}: {detail}`
**Resolution:** Fix the syntax error in the definition file. Check for incorrect indentation, missing colons, or malformed YAML constructs.

---

### DEFINITION_VALIDATION_ERROR

**Severity:** ERROR
**Trigger:** Definition file is syntactically valid but contains a semantic error, such as a field referencing an undeclared type.
**Message pattern:** `Definition validation failed in '{file}': field '{fieldPath}' references undeclared type '{typeName}'`
**Resolution:** Ensure all referenced types are declared in the same definition set or in an imported definition set.

---

### UNKNOWN_TYPE_REFERENCE

**Severity:** ERROR
**Trigger:** A field references a type name that has not been declared in any loaded definition set.
**Message pattern:** `Unknown type '{typeName}' referenced at '{fieldPath}' in '{file}'`
**Resolution:** Declare the missing type or add the definition set that contains it to the definition scan locations.

---

### CIRCULAR_TYPE_REFERENCE

**Severity:** ERROR
**Trigger:** Two or more types reference each other in a cycle with no nullable break.
**Message pattern:** `Circular type reference detected: {typeA} → {typeB} → {typeA}`
**Resolution:** Introduce a nullable optional field or restructure the type hierarchy to break the cycle.

---

### INVERSE_MAPPING_UNSAFE

**Severity:** ERROR
**Trigger:** An inverse mapping is declared but the reverse direction is ambiguous or would produce incorrect behavior due to lossy field mappings (e.g., flattening or value object conversion with no declared reverse converter).
**Message pattern:** `Inverse mapping '{mappingName}' is unsafe: field '{fieldPath}' has no reverse converter declared`
**Resolution:** Declare an explicit reverse converter, or write the inverse mapping explicitly rather than using `inverseOf`.

---

### AMBIGUOUS_CONVENTION_MAPPING

**Severity:** ERROR
**Trigger:** Convention-based mapping is enabled and multiple source fields match a target field with equal confidence.
**Message pattern:** `Ambiguous convention mapping: multiple source fields match target '{fieldPath}' in '{mappingName}'`
**Resolution:** Declare the field mapping explicitly in the mapping DSL file. TwinMapper never resolves ambiguity silently.

---

### CODEGEN_SPLIT_PACKAGE

**Severity:** ERROR
**Trigger:** The configured `basePackage` conflicts with an existing package in the module system, creating a split-package condition.
**Message pattern:** `Generated package '{package}' conflicts with an existing module package. Configure a unique basePackage.`
**Resolution:** Change `twinmapper.generation.base-package` to a package that does not exist in any module on the module path.

---

## Runtime Binding Errors

These errors occur during runtime document binding.

### MISSING_REQUIRED_FIELD

**Severity:** ERROR
**Trigger:** A field declared as `required: true` is absent from the input document.
**Message pattern:** `Required field '{fieldPath}' is missing in document '{sourceFile}'`
**Resolution:** Ensure the input document includes a value for the required field. Check for typos in field names and review the alias list for the field.

---

### UNKNOWN_FIELD

**Severity:** ERROR (STRICT mode) / WARNING (COMPATIBLE mode)
**Trigger:** The input document contains a field that is not declared in the definition and is not a known alias.
**Message pattern:** `Unknown field '{fieldPath}' in document '{sourceFile}'. Not declared in definition '{definitionName}'.`
**Resolution (STRICT):** Remove the unknown field from the input document or declare it in the definition.
**Resolution (COMPATIBLE):** The field is ignored. Consider declaring it if it carries meaningful data.

---

### FORBIDDEN_FIELD

**Severity:** ERROR (all modes)
**Trigger:** The input document contains a field that has been explicitly declared as forbidden — typically a renamed or removed legacy field.
**Message pattern:** `Forbidden field '{fieldPath}' found in document '{sourceFile}'. {migrationHint}`
**Resolution:** Replace the forbidden field with the canonical field name. The `migrationHint` provides the correct replacement.

---

### DEPRECATED_FIELD

**Severity:** ERROR (STRICT mode) / WARNING (COMPATIBLE mode)
**Trigger:** The input document uses a field name listed in `deprecatedAliases`.
**Message pattern:** `Deprecated field name '{fieldPath}' used in '{sourceFile}'. Use '{canonicalName}' instead.`
**Resolution:** Update the input document to use the canonical field name.

---

### INVALID_ENUM

**Severity:** ERROR
**Trigger:** An enum field contains a value that is not a member of the declared enum set.
**Message pattern:** `Invalid enum value '{actualValue}' for field '{fieldPath}'. Valid values are: {validValues}`
**Resolution:** Correct the enum value in the input document. Use the `code` value for string-to-enum binding.

---

### TYPE_MISMATCH

**Severity:** ERROR
**Trigger:** A field value cannot be converted to the declared target Java type.
**Message pattern:** `Type mismatch at '{fieldPath}': expected {expectedType}, got {actualType}`
**Resolution:** Correct the value in the input document, or declare a custom converter for this type combination.

---

### NESTED_BINDING_ERROR

**Severity:** ERROR
**Trigger:** A nested object binding fails. The parent error path is prefixed with the parent field's path.
**Message pattern:** `Binding error in nested object at '{fieldPath}': {nestedError}`
**Resolution:** Address the nested error reported inside the message. Path-aware reporting shows exactly which nested field caused the failure.

---

### CONDITIONAL_CONSTRAINT_VIOLATION

**Severity:** ERROR
**Trigger:** A conditionally required field is absent when its activating condition is met.
**Message pattern:** `Field '{fieldPath}' is required when '{conditionField}' is '{conditionValue}' but was absent`
**Resolution:** Include the required field when the discriminator field is set to the activating value.

---

### ONE_OF_VIOLATION

**Severity:** ERROR
**Trigger:** A one-of constraint requires exactly one field from a group to be present, but zero or more than one field is present.
**Message pattern:** `One-of constraint violated: exactly one of {fieldGroup} must be present, but {count} were present`
**Resolution:** Ensure exactly one field from the declared group is present in the input document.

---

### MUTUALLY_EXCLUSIVE_VIOLATION

**Severity:** ERROR
**Trigger:** Two fields declared as mutually exclusive are both present in the input document.
**Message pattern:** `Mutually exclusive fields '{field1}' and '{field2}' are both present at '{path}'`
**Resolution:** Remove one of the mutually exclusive fields from the input document.

---

### CONSTRAINT_VIOLATION

**Severity:** ERROR
**Trigger:** A field value violates a min, max, minLength, maxLength, or pattern constraint.
**Message pattern:** `Constraint violation at '{fieldPath}': value '{actualValue}' violates {constraintType} constraint '{constraintValue}'`
**Resolution:** Correct the field value to comply with the declared constraint.

---

## Runtime Object Mapping Errors

### NO_MAPPER_FOUND

**Severity:** ERROR
**Trigger:** No generated mapper is registered for the requested source/target type combination.
**Message pattern:** `No mapper registered for {sourceType} → {targetType}`
**Resolution:** Ensure a mapping definition exists in the YAML mapping DSL for this source/target pair and that the codegen phase completed successfully.

---

### MAPPING_ERROR

**Severity:** ERROR
**Trigger:** A mapper fails during field assignment due to a converter throwing an exception or a null policy violation.
**Message pattern:** `Mapping failed at field '{fieldPath}' in '{mappingName}': {cause}`
**Resolution:** Check the converter implementation for the affected field. For null policy violations, ensure the source provides the required non-null value.

---

### NULL_REQUIRED_MAPPING_FIELD

**Severity:** ERROR
**Trigger:** A source field mapped with `FAIL_ON_NULL` policy is null.
**Message pattern:** `Null value for required mapping field '{fieldPath}' in '{mappingName}'. Null policy is FAIL_ON_NULL.`
**Resolution:** Ensure the source object provides a non-null value for this field, or change the null policy to `IGNORE_NULLS` or `SET_NULLS`.

---

## Validation Errors

### VALIDATION_FAILED

**Severity:** ERROR
**Trigger:** One or more generated schema validators found violations.
**Message pattern:** `Validation failed for '{targetType}': {errorCount} error(s) found`
**Resolution:** Inspect the individual field errors in the `ValidationReport` for specific constraint violations.

---

## Warning Codes

### DEPRECATED_ALIAS_USED

**Severity:** WARNING
**Trigger:** An input document uses a deprecated alias in COMPATIBLE mode.
**Message pattern:** `Deprecated alias '{alias}' used for field '{canonicalName}' in '{sourceFile}'. Update to canonical name.`

---

### UNKNOWN_FIELD_IGNORED

**Severity:** WARNING
**Trigger:** An unknown field was encountered and ignored in COMPATIBLE or LENIENT mode.
**Message pattern:** `Unknown field '{fieldPath}' ignored in '{sourceFile}'. Declare it if it carries meaningful data.`

---

### CONVENTION_MAPPING_APPLIED

**Severity:** WARNING (when convention mapping is enabled)
**Trigger:** A field was resolved by convention rather than explicit mapping declaration.
**Message pattern:** `Convention mapping applied: '{sourceField}' → '{targetField}' in '{mappingName}'. Consider declaring explicitly.`

---

## Security Errors

### XXE_REJECTED

**Severity:** ERROR
**Trigger:** A BPMN document contains an XML External Entity reference that was rejected by the parser.
**Message pattern:** `XXE payload rejected in BPMN document '{sourceFile}'. External entity processing is disabled.`
**Resolution:** Remove DTD declarations and external entity references from the BPMN document.

---

### UNSAFE_YAML_CONSTRUCTION_REJECTED

**Severity:** ERROR
**Trigger:** A YAML document contains a type tag (`!!`) attempting to instantiate an arbitrary Java class.
**Message pattern:** `Unsafe YAML type construction rejected in '{sourceFile}'. Arbitrary Java object instantiation is not permitted.`
**Resolution:** Remove the type tag from the YAML document. TwinMapper's YAML binding does not support `!!` type directives.
