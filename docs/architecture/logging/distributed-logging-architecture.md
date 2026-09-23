# HM Logging — Distributed Logging Architecture

**Architecture baseline:** 22 September 2026

| Property | Value |
|---|---|
| **Status** | Architecture defined; Contracts v1 published as v1.0.0-preview.1 through NuGet and BSR |
| **Date** | 22 September 2026 |
| **Scope** | Technology-independent distributed logging architecture |
| **Relationship to domain** | Builds on HM Logging Domain without modifying its fundamental semantics |
| **Primary distributed concept** | Logging Flow |

## 1. Purpose

This document defines the distributed logging architecture built on top of the HM Logging domain. Its purpose is to establish the concepts, invariants, lifecycle and interaction model required when logging context must survive across process and network boundaries.

It is intentionally broader than any one implementation. A component may adopt or implement this architecture without this document prescribing a fixed project dependency graph. The architecture defines behavior and semantics; concrete products decide how they realize them.

## 2. Relationship to HM Logging Domain and Core

HM Logging Domain remains the source of truth for fundamental logging semantics such as LogEntry, LogContext, LogLevel, metadata, normalization intent and precedence. Distributed logging extends those semantics; it does not redefine them.

Hm.Logging is the existing Core library and public technical namespace reserved for the Core implementation. Core provides local logging and local nested scopes and remains independent from distributed transport, gRPC, Logging Flows, service state and providers.

A distributed implementation maps remote representations into the domain/Core semantics without forcing transport concerns back into Core.

## 3. Why Distributed Logging Needs an Additional Model

A local scope can rely on in-process lifetime and execution context. A remote call cannot. Independent unary requests may arrive at different times, on different threads, through different connections or even different service instances. Therefore a remote scope cannot be represented by a local IDisposable lifetime or by assuming a persistent connection.

The architecture introduces an explicit remotely addressable state container: the Logging Flow.

## 4. Logging Flow

### 4.1 Definition

A Logging Flow is a logical remote container that owns an ordered stack of logging contexts across multiple independent calls. It has an explicit identity and a lifecycle independent from the current depth of its scope stack.

### 4.2 FlowId

FlowId is an opaque handle that identifies one Logging Flow. Consumers retain it and send it with later operations that must address the same Flow. Its internal format is not part of the conceptual architecture.

- FlowId is not TraceId.
- FlowId is not CorrelationId.
- FlowId is not a business transaction identifier.
- Two or more Flows may share TraceId, CorrelationId, Source or metadata and still remain fully independent.

### 4.3 Multiple Flows

A single client process may create and use multiple Flows simultaneously. Each Flow has independent scope state, activity and lifecycle. An operation targeting one Flow must not mutate or refresh another Flow merely because both originate from the same process or share contextual values.

## 5. Explicit Flow Lifecycle

### 5.1 CreateFlow

Flow creation is explicit. CreateFlow creates a new empty Logging Flow and returns its FlowId. Creation is not hidden inside PushScope.

```text
CreateFlow()
 -> FlowId = XYZ
 -> Flow XYZ stack = []
```

### 5.2 CloseFlow

Flow closure is explicit. CloseFlow terminates the addressed Flow. After closure, the Flow is no longer an active target for scope operations. Flow-aware logging behavior for inactive Flows and the exact transport response/result model are defined by `docs/architecture/logging/contracts-architecture.md`.

```text
CloseFlow(XYZ)
 -> Flow XYZ becomes closed/removed
```

### 5.3 Empty stack does not mean closed Flow

Flow lifetime and scope-stack depth are separate. Removing the last scope leaves a valid Flow with an empty stack. The Flow ends only through explicit closure or implementation-owned expiration/cleanup.

```text
Flow XYZ = [A]
PopScope(XYZ)
Flow XYZ = [] // still active
Log(XYZ, entry) // valid
CloseFlow(XYZ) // lifecycle ends
```

## 6. Remote Scope Stack

### 6.1 PushScope

PushScope adds one LogContext layer to an existing Flow. It requires the target FlowId. It does not create a Flow.

### 6.2 PopScope

PopScope removes only the most recently pushed context layer. The previous effective context is restored. PopScope is therefore LIFO by architectural semantics.

### 6.3 Effective context

The effective context is obtained from the ordered scope stack using the HM Logging precedence semantics. Inner/more specific context overrides inherited values where the domain permits override. A LogEntry's explicit values remain authoritative over inherited context.

```text
Flow stack: [ A, B, C ]
Effective context: A + B + C

PopScope()

Flow stack: [ A, B ]
Effective context: A + B
```

## 7. Logging Operations

### 7.1 Flow-bound log

A log operation may include a FlowId. The distributed implementation resolves the Flow's effective context and applies it to the incoming LogEntry according to HM Logging semantics before the event reaches its final logging pipeline.

### 7.2 Independent log

A Flow is optional. A client may send an independent Log without FlowId when no cross-call contextual lifetime is needed. Distributed logging must not make every log stateful.

```text
Log(entry) -> independent event
Log(flowId, entry) -> event participates in Flow context
```

## 8. Complete Conceptual Interaction

```text
CreateFlow() -> FlowId XYZ
PushScope(XYZ, A) -> [A]
Log(XYZ, entry1) -> effective A

PushScope(XYZ, B) -> [A, B]
Log(XYZ, entry2) -> effective A + B

PushScope(XYZ, C) -> [A, B, C]
Log(XYZ, entry3) -> effective A + B + C

PopScope(XYZ) -> [A, B]
Log(XYZ, entry4) -> effective A + B

PopScope(XYZ) -> [A]
PopScope(XYZ) -> []
Log(XYZ, entry5) -> valid; no scoped context

CloseFlow(XYZ) -> Flow ends
```

## 9. Flow Activity, Inactivity and Expiration

Remote clients can crash, disconnect or fail to close a Flow. A distributed implementation therefore needs an abandonment strategy. Expiration is evaluated independently per Flow.

Activity in Flow B must never keep Flow A alive. This remains true when A and B belong to the same client process or share tracing/correlation information.

The concrete inactivity duration, refresh rules, storage, cleanup algorithm and scheduling mechanism are implementation policy rather than fundamental protocol semantics. Contracts may expose stable outcomes needed by consumers, but should not embed an operational timeout policy into the domain model.

## 10. Concurrency

Each Logging Flow maintains a strictly ordered LIFO stack. Operations that mutate Flow state (PushScope, PopScope and CloseFlow) must be applied atomically and serialized per Flow. The Service is responsible for preventing concurrent calls against the same FlowId from corrupting the stack or producing an internally inconsistent state.

The client is responsible for semantic ordering. If multiple tasks within the same client use the same FlowId, the client must coordinate operations whenever a particular logical order is required. The Service guarantees integrity and a well-defined processing order; it does not infer the client's intended business ordering. Independent Flows may be processed concurrently. The concrete mechanism (for example, a per-Flow lock, actor or serialized queue) is an Hm.Logging.Service implementation concern, not part of the wire contract.

### 10.1 Consecutive identical scope rule

A Logging Flow must not contain two consecutive scopes whose normalized LogContext values are identical. PushScope compares the normalized incoming context with the current top scope. If they are identical, the operation does not add another stack level and still completes successfully. This rule removes a semantically redundant state: repeating an identical top context does not change the effective logging context. The same context may appear again later in the Flow when one or more different scopes exist between occurrences.

```text
[A, B] + PushScope(B) -> [A, B]          // redundant, success
[A, B, C] + PushScope(B) -> [A, B, C, B] // valid
```

### 10.2 Retry and idempotency semantics

Retry behavior is defined per operation. The protocol does not require `isRetry`, `ScopeId` or `OperationId` for the currently established model. PushScope derives safe retry behavior from the consecutive-identical-scope rule. PopScope offers an expected-context form for safe retry when the client needs resilience.

| Operation | Idempotency / retry semantics | Established behavior |
|---|---|---|
| CreateFlow | Not idempotent | A retry may create a second Flow if the first response was lost. The client uses the FlowId it receives; an unused orphan Flow is eventually removed by inactivity cleanup. |
| CloseFlow | Idempotent by postcondition | Closing an already closed or absent Flow leaves the requested final state: the Flow is not active. Both cases are successful. |
| Log | Not idempotent | If the Service processed the log but the response was lost, retrying may produce an additional log entry. This exceptional duplicate is accepted and documented. |
| PushScope | Idempotent under the consecutive-identical-scope rule | If the normalized incoming context equals the top scope, no new level is added and the call succeeds. Otherwise it is pushed and succeeds. The same behavior applies to first attempts and retries. |
| PopScope(FlowId) | Not retry-safe | Removes the current top without checking its identity. A blind retry could remove the next scope and therefore must not be treated as safe retry behavior. |
| PopScope(FlowId, ExpectedContext) | Idempotent under the consecutive-identical-scope rule | Removes the top only when it equals ExpectedContext. If it no longer matches, nothing is removed and the call still succeeds; this safely covers a lost response after a successful first Pop. |

For PopScope with ExpectedContext, if the first request never reached the Service, the expected context remains at the top and is removed by the retry. If the first request succeeded but its response was lost, the next top differs and the retry removes nothing. Because identical contexts cannot be consecutive, a successful Pop cannot expose an immediately identical scope and accidentally remove it on retry.

## 11. Failure and Invalid-State Semantics

The architecture recognizes that operations may target unknown, closed, expired or otherwise unavailable FlowIds. It also recognizes invalid state transitions such as PopScope against an empty stack. Exact gRPC status codes, response payloads and retry semantics are defined by `docs/architecture/logging/contracts-architecture.md`.

A distributed implementation must not silently reinterpret one Flow as another or recreate an unknown Flow implicitly. Explicit Flow identity and lifecycle are architectural invariants.

## 12. Transport Independence

Logging Flows solve a distributed state problem; they are not inherently tied to gRPC. Hm.Logging.Contracts expresses this architecture through protobuf/gRPC, but the architecture itself is transport-independent and may be implemented by other distributed components in the future.

For the planned gRPC protocol, protobuf definitions must remain interoperable across languages. .NET-specific generated types or NuGet packaging are conveniences for .NET consumers, not the conceptual definition of the protocol.

## 13. Conceptual Operations

| Operation | Purpose | Flow requirement |
|---|---|---|
| CreateFlow | Create a new empty Flow and return its identity | No existing Flow |
| CloseFlow | Explicitly end a Flow lifecycle | Existing FlowId |
| PushScope | Add one contextual layer | Existing FlowId |
| PopScope | Remove the top contextual layer | Existing FlowId |
| Log | Submit a structured log entry | FlowId optional |

## 14. State Machine

```text
NO FLOW
   |
   | CreateFlow
   v
ACTIVE FLOW []
   |
   | PushScope(A)
   v
ACTIVE FLOW [A]
   |
   | PushScope(B)
   v
ACTIVE FLOW [A,B]
   |
   | PopScope
   v
ACTIVE FLOW [A]
   |
   | PopScope
   v
ACTIVE FLOW []
   |
   | CloseFlow
   v
CLOSED / REMOVED

At any ACTIVE state:
  Log(flowId, entry) -> use current effective context

Without a Flow:
  Log(entry) -> independent event

Abandoned ACTIVE Flow:
  per-Flow inactivity policy -> EXPIRED
```

## 15. Architectural Invariants

- Remote scope state is addressed explicitly; it never depends on a persistent connection, request object, thread or local execution context.
- CreateFlow and CloseFlow define an explicit Flow lifecycle.
- PushScope never implicitly creates a Flow.
- An empty scope stack does not close a Flow.
- PopScope is LIFO and removes only the top scope.
- Logs can be Flow-bound or independent.
- FlowId is opaque and semantically distinct from tracing/correlation identifiers.
- Multiple Flows from one client remain independent.
- Inactivity/expiration is evaluated per Flow.
- Distributed concepts must not force transport-specific concerns into HM Logging Domain or Hm.Logging Core.
- The architecture is transport-independent even when expressed through gRPC in Hm.Logging.Contracts.

## 16. Responsibility Boundaries

This architecture intentionally avoids declaring a permanent list of components that are allowed to know or implement Logging Flows. Instead it defines capability boundaries:

- A contract layer may expose the operations and wire representations required to interact with Logging Flows without implementing their state.
- A distributed runtime/service may implement Flow state, stack management, lifecycle, concurrency, expiration and mapping into the logging pipeline.
- A local logging Core does not need Logging Flow concepts to remain conformant with the HM Logging domain.
- Other future components may implement or consume this architecture as long as they preserve its semantics and invariants.

## 17. Compatibility and Evolution Principles

The public distributed contract must evolve without changing the meaning of established Flow operations. Wire-level versioning, field numbering, enum representation, optionality and backward/forward compatibility for Contracts v1 are defined by `docs/architecture/logging/contracts-architecture.md`.

Because non-.NET clients are an explicit requirement, protocol decisions must be evaluated for cross-language protobuf behavior rather than only for generated C# ergonomics.

## 18. Decisions Established

- Logging Flow is an explicit distributed architecture concept.
- Flow creation uses CreateFlow and returns FlowId.
- Flow closure uses CloseFlow.
- PushScope requires an existing FlowId.
- PopScope removes the top scope and does not close the Flow when the stack becomes empty.
- A Flow may remain active with an empty stack.
- Log may include FlowId or operate independently.
- One client may maintain multiple independent Flows.
- Different Flows may share TraceId, CorrelationId and contextual values without becoming coupled.
- Flow inactivity/expiration is independent per Flow.
- Remote Flow state is not part of Hm.Logging Core.
- The architecture must be expressible through an interoperable, language-neutral protocol.
- Consecutive normalized identical scopes are not allowed at the top of a Flow; redundant PushScope calls succeed without adding another level.
- PushScope does not require retry markers or operation identifiers; first attempts and retries follow the same top-context comparison rule.
- PopScope may be unconditional or guarded by ExpectedContext; the guarded form is the retry-safe form.
- Per-Flow state mutations are atomic and serialized by the Service; semantic ordering remains the client responsibility.
- Retry/idempotency behavior is defined per operation as documented in Section 10.2.

## 19. Open Design Items

The Contracts v1 wire-level decisions previously tracked here are defined by `docs/architecture/logging/contracts-architecture.md` and are implemented and publicly distributed as `v1.0.0-preview.1`. Release versioning, BSR publication, release automation, provenance/attestation, cross-registry coherence, recovery/idempotency, and baseline persistence are completed delivery concerns and do not reopen the distributed Flow semantics defined here.

The remaining open items belong to the distributed runtime / `Hm.Logging.Service` design:

- Exact Service implementation mechanism for per-Flow serialization (lock, actor, queue, or equivalent).
- Flow ownership/authorization model.
- Which valid operations refresh inactivity and the operational expiration policy.
- Persistence/recovery behavior across distributed runtime restarts.

## 20. Terminology

| Term | Meaning |
|---|---|
| HM Logging | Conceptual name for the overall HM logging ecosystem. |
| Hm.Logging | Reserved technical namespace/name of the existing Core library. |
| Logging Flow | Remote state container with explicit identity/lifecycle and an ordered scope stack. |
| FlowId | Opaque identifier of one Logging Flow. |
| Scope | One contextual layer pushed onto a Flow stack. |
| Effective context | Merged contextual state visible at the current stack depth. |
| Independent log | Remote log submitted without a FlowId. |
| Flow-bound log | Remote log submitted with a FlowId and enriched by that Flow's effective context. |

## 21. Current Architectural Baseline

This document is the current baseline for distributed logging architecture. Contracts v1 wire-level decisions are defined by `docs/architecture/logging/contracts-architecture.md`; the approved protocol is implemented and publicly released as `v1.0.0-preview.1` through NuGet and BSR. The v1 public preview is now the compatibility baseline. Remaining open architecture items are Service/runtime concerns. Later implementation details may refine operational behavior, but changes to the established invariants above should be treated as explicit architectural decisions rather than incidental code changes.
