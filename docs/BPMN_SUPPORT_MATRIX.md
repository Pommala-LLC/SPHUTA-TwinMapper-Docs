# TwinMapper — BPMN Support Matrix

## Overview

TwinMapper supports BPMN 2.0 XML as a first-class input format. This document defines which BPMN 2.0 elements and attributes TwinMapper supports in its intermediate model, how they map to TwinMapper's internal definition model, and what behavior occurs for unsupported elements.

BPMN parsing uses JDK StAX with no third-party BPMN library. TwinMapper defines its own `BpmnIntermediateModel` covering the supported vocabulary only.

---

## BPMN Parsing Architecture

```
BPMN XML Input
    │
    ▼
BpmnDocumentParser (JDK StAX)
[XXE disabled: IS_SUPPORTING_EXTERNAL_ENTITIES=false, SUPPORT_DTD=false]
    │
    ▼
BpmnIntermediateModel
(TwinMapper-owned typed model)
    │
    ▼
BpmnIntermediateModelMapper
    │
    ├── Build time: → Internal DefinitionModel
    └── Runtime:    → NodeCursor → Generated Binder → DTO
```

---

## Supported BPMN Elements

### Process Container

| BPMN Element | Supported | Notes |
|---|---|---|
| `<definitions>` | Yes | Root element; `id`, `name`, `targetNamespace` parsed |
| `<process>` | Yes | `id`, `name`, `isExecutable` parsed |
| `<subProcess>` | Yes | Treated as nested process scope |
| `<callActivity>` | Yes | `calledElement` attribute parsed |
| `<transaction>` | Partial | Treated as subProcess; transaction semantics not modeled |

---

### Flow Nodes — Tasks

| BPMN Element | Supported | Notes |
|---|---|---|
| `<task>` | Yes | Generic task |
| `<userTask>` | Yes | `assignee`, `candidateGroups`, `candidateUsers` from extension elements |
| `<serviceTask>` | Yes | `implementation` attribute parsed |
| `<scriptTask>` | Yes | `scriptFormat`, script body parsed |
| `<businessRuleTask>` | Yes | `decisionRef` from extension elements |
| `<sendTask>` | Yes | `implementation` attribute parsed |
| `<receiveTask>` | Yes | `messageRef` attribute parsed |
| `<manualTask>` | Yes | Treated as userTask without assignee |

---

### Flow Nodes — Gateways

| BPMN Element | Supported | Notes |
|---|---|---|
| `<exclusiveGateway>` | Yes | `default` attribute (default sequence flow) parsed |
| `<inclusiveGateway>` | Yes | `default` attribute parsed |
| `<parallelGateway>` | Yes | No additional attributes |
| `<eventBasedGateway>` | Yes | `instantiate` attribute parsed |
| `<complexGateway>` | Partial | Parsed as gateway; complex activation condition not modeled |

---

### Flow Nodes — Events

#### Start Events

| BPMN Element | Supported | Notes |
|---|---|---|
| `<startEvent>` (none) | Yes | Plain start |
| `<startEvent>` + `<messageEventDefinition>` | Yes | `messageRef` parsed |
| `<startEvent>` + `<timerEventDefinition>` | Yes | `timeDuration`, `timeDate`, `timeCycle` parsed |
| `<startEvent>` + `<signalEventDefinition>` | Yes | `signalRef` parsed |
| `<startEvent>` + `<errorEventDefinition>` | Yes | `errorRef` parsed |
| `<startEvent>` + `<conditionalEventDefinition>` | Yes | Condition expression parsed |
| `<startEvent>` + `<escalationEventDefinition>` | Yes | `escalationRef` parsed |
| `<startEvent>` + `<compensateEventDefinition>` | No | Not in supported vocabulary |

#### End Events

| BPMN Element | Supported | Notes |
|---|---|---|
| `<endEvent>` (none) | Yes | Plain end |
| `<endEvent>` + `<messageEventDefinition>` | Yes | Sending end event |
| `<endEvent>` + `<errorEventDefinition>` | Yes | Error throw end event |
| `<endEvent>` + `<signalEventDefinition>` | Yes | Signal throw end event |
| `<endEvent>` + `<escalationEventDefinition>` | Yes | Escalation throw end event |
| `<endEvent>` + `<terminateEventDefinition>` | Yes | Process termination |
| `<endEvent>` + `<compensateEventDefinition>` | No | Not in supported vocabulary |

#### Intermediate Events (Catching)

| BPMN Element | Supported | Notes |
|---|---|---|
| Intermediate catching (none) | Yes | Link catch treated as none |
| + `<messageEventDefinition>` | Yes | `messageRef` parsed |
| + `<timerEventDefinition>` | Yes | `timeDuration`, `timeDate`, `timeCycle` parsed |
| + `<signalEventDefinition>` | Yes | `signalRef` parsed |
| + `<conditionalEventDefinition>` | Yes | Condition expression parsed |
| + `<linkEventDefinition>` | Yes | `name` used as link identifier |
| + `<compensateEventDefinition>` | No | Not in supported vocabulary |

#### Intermediate Events (Throwing)

| BPMN Element | Supported | Notes |
|---|---|---|
| Intermediate throwing (none) | Yes | |
| + `<messageEventDefinition>` | Yes | |
| + `<signalEventDefinition>` | Yes | |
| + `<escalationEventDefinition>` | Yes | |
| + `<linkEventDefinition>` | Yes | |
| + `<compensateEventDefinition>` | No | Not in supported vocabulary |

#### Boundary Events

| BPMN Element | Supported | Notes |
|---|---|---|
| Boundary + `<timerEventDefinition>` | Yes | `cancelActivity` attribute parsed |
| Boundary + `<messageEventDefinition>` | Yes | |
| Boundary + `<errorEventDefinition>` | Yes | |
| Boundary + `<signalEventDefinition>` | Yes | |
| Boundary + `<escalationEventDefinition>` | Yes | |
| Boundary + `<conditionalEventDefinition>` | Yes | |
| Boundary + `<compensateEventDefinition>` | No | Not in supported vocabulary |

---

### Sequence Flow

| BPMN Element | Supported | Notes |
|---|---|---|
| `<sequenceFlow>` | Yes | `id`, `name`, `sourceRef`, `targetRef` parsed |
| `<conditionExpression>` | Yes | Expression body parsed as string |
| Default flow (via `default` attribute) | Yes | Identified in intermediate model |

---

### Data and Variables

| BPMN Element | Supported | Notes |
|---|---|---|
| `<dataObject>` | No | Not in supported vocabulary |
| `<dataStore>` | No | Not in supported vocabulary |
| `<property>` | No | Not in supported vocabulary |
| Input/Output sets | No | Not in supported vocabulary |

---

### Participants and Collaboration

| BPMN Element | Supported | Notes |
|---|---|---|
| `<participant>` | No | Not in supported vocabulary |
| `<messageFlow>` | No | Not in supported vocabulary |
| `<collaboration>` | No | Not in supported vocabulary |

---

### Lanes

| BPMN Element | Supported | Notes |
|---|---|---|
| `<laneSet>` | Partial | Parsed for metadata/labeling purposes only; not mapped to DTO fields |
| `<lane>` | Partial | `name` attribute parsed; `flowNodeRef` elements read for lane membership |

---

### Extension Elements

| Extension Element | Support Model |
|---|---|
| `<extensionElements>` | Yes — container parsed |
| Custom extension child elements | Parsed as key-value pairs into `ExtensionBlock` in the intermediate model |
| Camunda extension elements (`camunda:*`) | Parsed as generic extension elements; not interpreted semantically |
| Flowable extension elements (`flowable:*`) | Parsed as generic extension elements; not interpreted semantically |

Extension elements are available in the intermediate model's `ExtensionBlock` and can be mapped to DTO fields via definition configuration.

---

## Unsupported Element Behavior

BPMN elements not in the supported vocabulary are handled as follows:

| Context | Behavior |
|---|---|
| Build-time definition scan | Warning diagnostic emitted; element skipped |
| Runtime document binding | Element silently skipped; no error |
| Compensation constructs | Silently skipped; no compensate DTO fields generated |
| Choreography tasks | Silently skipped |
| Data objects/stores | Silently skipped |
| Collaboration diagrams | Silently skipped |

No unsupported element causes binding failure. Only supported elements contribute to the generated DTO.

---

## Attribute Support

### Common Attributes (all flow elements)

| Attribute | Supported |
|---|---|
| `id` | Yes |
| `name` | Yes |
| `documentation` | Yes — parsed as description |
| `incoming` | Yes — resolved to SequenceFlow references |
| `outgoing` | Yes — resolved to SequenceFlow references |

### Task-Specific Attributes

| Attribute | Supported |
|---|---|
| `implementation` | Yes |
| `default` | Yes |
| `isForCompensation` | No |
| `startQuantity` | No |
| `completionQuantity` | No |

### Event Attributes

| Attribute | Supported |
|---|---|
| `isInterrupting` | Yes — on start events |
| `cancelActivity` | Yes — on boundary events |
| `parallelMultiple` | No |

---

## Security Requirements

All BPMN parsing enforces these settings before any document is processed. Both must be set. Omitting either is a security vulnerability.

```java
XMLInputFactory factory = XMLInputFactory.newInstance();
factory.setProperty(XMLInputFactory.IS_SUPPORTING_EXTERNAL_ENTITIES, false);
factory.setProperty(XMLInputFactory.SUPPORT_DTD, false);
```

---

## BPMN Namespace Support

| Namespace | Status |
|---|---|
| `http://www.omg.org/spec/BPMN/20100524/MODEL` | Supported — primary BPMN 2.0 namespace |
| `http://www.omg.org/spec/BPMN/20100524/DI` | Ignored — diagram interchange, not structural |
| `http://www.omg.org/spec/DD/20100524/DC` | Ignored — diagram common |
| `http://www.omg.org/spec/DD/20100524/DI` | Ignored — diagram interchange |
| Custom namespaces in extension elements | Parsed as opaque extension data |

---

## Diagnostic Messages for Unsupported Elements

When an unsupported BPMN element is encountered at build-time scan, the following diagnostic is emitted:

```
[BPMN_UNSUPPORTED_ELEMENT] Element '<compensateEventDefinition>' at
process 'order-fulfilment', element 'compensation-boundary' is not
in the TwinMapper supported BPMN vocabulary and will be ignored.
```

This is a WARNING severity diagnostic at build time and is silently ignored at runtime.
