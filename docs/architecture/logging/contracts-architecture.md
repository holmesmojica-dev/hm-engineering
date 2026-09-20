# HM Logging Contracts Architecture

**Component:** `Hm.Logging.Contracts`\
**Protocol:** `hm.logging.contracts.v1`\
**NuGet package:** `HDev.Hm.Logging.Contracts`\
**Status:** Architecture baseline for Contracts v1\
**Date:** 19 September 2026

------------------------------------------------------------------------

## 1. Purpose

This document defines the architecture and wire-contract decisions for
`Hm.Logging.Contracts`.

It is the authoritative architecture specification for how the HM
Logging domain and the distributed logging architecture are expressed
through Protocol Buffers and gRPC.

This document does not redefine the HM Logging domain and does not
define the internal implementation of `Hm.Logging.Service`.

### 1.1 Source-of-truth hierarchy

Contracts implementations must interpret the engineering documentation
in the following order:

1.  `docs/domains/logging/Logging.md` --- canonical domain semantics.
2.  `docs/architecture/logging/distributed-logging-architecture.md` ---
    distributed logging and Logging Flow architecture.
3.  `docs/architecture/logging/contracts-architecture.md` ---
    protobuf/gRPC representation defined by this document.
4.  ADRs and engineering standards applicable to the implementation.

If an implementation concern appears ambiguous or conflicts with these
sources, the implementation must not silently introduce a new
architectural decision. The ambiguity must be reported and resolved at
the architecture level first.

------------------------------------------------------------------------

## 2. Architectural Role

`Hm.Logging.Contracts` is the protocol-contract component of the HM
Logging ecosystem.

Its responsibilities are:

-   Define the language-neutral protobuf schema for the distributed
    logging API.
-   Define the gRPC service operations and their request/response
    messages.
-   Represent HM Logging domain concepts without coupling the protocol
    to .NET.
-   Preserve protobuf field presence where domain semantics require
    distinguishing absence from an explicit value.
-   Define typed transport representations for supported metadata
    values.
-   Encapsulate transport-specific conversion and validation when the
    wire representation itself requires it.
-   Provide generated .NET gRPC/protobuf types and Contracts-owned
    adapters needed by .NET consumers.
-   Preserve compatibility rules for protocol v1.

It does **not** own Logging Flow state or domain behavior.

------------------------------------------------------------------------

## 3. Dependency and Responsibility Boundaries

### 3.1 Independence

`Hm.Logging.Contracts` must not depend on:

-   `Hm.Logging`
-   `Hm.Logging.Service`
-   `Hm.Logging.Providers.*`

The `.proto` definitions are the protocol source of truth and must
remain usable by future Java, Python, Go, JavaScript, or other clients.

The initial NuGet package is a .NET distribution mechanism, not the
conceptual definition of the protocol.

### 3.2 Relationship to Hm.Logging

`Hm.Logging` is the independent Core logging library. It owns local
logging behavior and implements the HM Logging domain for .NET.

Contracts may represent equivalent domain concepts, but it must not
force Core to adopt transport-specific behavior.

In particular:

-   Core does not know about gRPC.
-   Core does not know about protobuf.
-   Core does not know about FlowId.
-   Core does not know about remote Logging Flows.

### 3.3 Relationship to Hm.Logging.Service

`Hm.Logging.Service` will implement the gRPC contract and is responsible
for runtime concerns such as:

-   Flow storage and lifecycle.
-   Flow stack management.
-   Per-Flow concurrency and atomicity.
-   Flow expiration.
-   Effective-context calculation.
-   Defensive ingress validation.
-   Mapping valid Contracts representations into Core/domain semantics.
-   Logging pipeline execution.

Contracts defines what can be communicated. Service defines how
distributed runtime state is implemented.

### 3.4 Relationship to Providers

Providers are independent implementations of Core provider abstractions.
Contracts does not depend on them and does not prescribe persistence or
output behavior.

------------------------------------------------------------------------

## 4. Naming and Versioning

### 4.1 Names

The approved names are:

  Concern                    Name
  -------------------------- -----------------------------
  .NET project / namespace   `Hm.Logging.Contracts`
  NuGet package              `HDev.Hm.Logging.Contracts`
  Protobuf package           `hm.logging.contracts.v1`

Generated C# types must use:

``` proto
option csharp_namespace = "Hm.Logging.Contracts";
```

### 4.2 Protocol and package major versions

The NuGet package major version must equal the protobuf protocol major
version.

Therefore:

-   `hm.logging.contracts.v1` corresponds to package `1.x.x`.
-   `hm.logging.contracts.v2` corresponds to package `2.x.x`.

The first preview release of Contracts is therefore expected to be:

``` text
1.0.0-preview.1
```

and not `0.1.0-preview.1`.

### 4.3 Preview maturity

Preview maturity is expressed only through SemVer prerelease
identifiers.

The protobuf package remains:

``` text
hm.logging.contracts.v1
```

from the first v1 preview through stable v1.

A package such as `hm.logging.contracts.v1.preview` must not be
introduced because changing it later would change fully qualified
protobuf and gRPC identities.

### 4.4 Compatibility

Once protocol v1 becomes stable:

-   Existing field numbers must not be reused.
-   Existing enum numeric values must not be renumbered.
-   Removed fields should be declared `reserved`.
-   Backward-compatible additions may evolve within v1.
-   An incompatible protocol change requires protocol v2 and package
    major 2.

During preview, the schema may evolve before stable release, but
protocol/package major coherence must still be preserved.

### 4.5 .NET target and schema distribution

The initial `Hm.Logging.Contracts` implementation targets `net10.0`, aligned
with the HM .NET ecosystem baseline. Contracts remains language-neutral at the
protocol level even though the official package initially targets .NET 10.

HM-owned `.proto` schemas are the canonical protocol source and must be
included in the `HDev.Hm.Logging.Contracts` NuGet package alongside the
compiled .NET protobuf/gRPC types. The schemas distributed with a package
release must correspond exactly to that package version.

The canonical `google.type.Decimal` definition remains owned by Google Common
Protos. HM Logging must not fork or redefine that schema as an HM-owned type.
The .NET implementation uses the canonical Google Common Protos support needed
for `Google.Type.Decimal`, while protobuf compilation resolves the canonical
`google/type/decimal.proto` import through the package/source import mechanism.

Official non-.NET SDKs are outside the current Contracts scope. Non-.NET
consumers may use the published language-neutral schemas to generate and
maintain clients in their own language/toolchain.

------------------------------------------------------------------------

## 5. Domain Representation Principles

### 5.1 Presence

A domain value that may be absent must preserve absence in Contracts
when protobuf scalar defaults would otherwise erase that distinction.

The general rule is:

> Optional domain scalar values use explicit protobuf presence when
> absence has semantic meaning.

Message-valued fields already provide presence.

### 5.2 Normalization boundaries

Contracts transports semantic data. It must not duplicate all domain
normalization rules.

Domain-oriented validation and normalization are generally performed at
the Service ingress or subsequent domain mapping layer.

However, validation or conversion that exists specifically because of a
transport representation belongs in Contracts-owned adapters where
appropriate.

The Service must remain defensive because wire input is untrusted.

------------------------------------------------------------------------

## 6. LogLevel

The wire enum preserves the same severity ordering and numeric values
used by the HM Logging domain and Core:

``` proto
enum LogLevel {
  LOG_LEVEL_TRACE = 0;
  LOG_LEVEL_DEBUG = 1;
  LOG_LEVEL_INFORMATION = 2;
  LOG_LEVEL_WARNING = 3;
  LOG_LEVEL_ERROR = 4;
  LOG_LEVEL_CRITICAL = 5;
}
```

`Trace = 0` is a valid explicit value.

Therefore the `LogEntry` level field must preserve presence:

``` proto
optional LogLevel level = ...;
```

Semantics:

-   Level absent -\> later semantic default is `Information`.
-   Level explicitly `TRACE` -\> Trace.
-   Unknown/untrusted enum numeric values must be rejected defensively
    before domain conversion.
-   Contracts must not shift the enum merely to create an
    `UNSPECIFIED = 0` value.

This enum is a domain value enum, not an RPC-result enum.

------------------------------------------------------------------------

## 7. LogEntry

Contracts represents the canonical Log Entry semantics.

Conceptually:

``` proto
message LogEntry {
  string message = 1;
  optional LogLevel level = 2;
  google.protobuf.Timestamp timestamp = 3;
  optional string source = 4;
  optional string trace_id = 5;
  optional string correlation_id = 6;
  optional string exception = 7;
  map<string, MetadataValue> metadata = 8;
}
```

The field numbers follow the canonical LogEntry property order and are
compatibility-sensitive protocol identifiers.

### 7.1 Message

`message` is semantically required and has no valid absent state.

A plain protobuf `string` is sufficient because both
missing/default-empty and explicitly empty input are invalid.

At Service ingress:

-   absent/default empty -\> `INVALID_ARGUMENT`
-   empty -\> `INVALID_ARGUMENT`
-   whitespace-only -\> `INVALID_ARGUMENT`

The log operation must not continue with an invalid Message.

### 7.2 Level

`level` uses explicit presence.

-   absent -\> semantic default `Information` is applied later.
-   present -\> preserve the supplied value.

Contracts itself does not apply the default.

### 7.3 Timestamp

`timestamp` uses `google.protobuf.Timestamp`.

It is required in the final semantic LogEntry but may be omitted on the
wire.

-   supplied -\> explicit event time.
-   omitted -\> Contracts preserves absence.
-   a later layer applies the domain default.

Contracts must not generate a timestamp merely because the field was
omitted.

### 7.4 Source

`source` is optional:

``` proto
optional string source = ...;
```

Contracts preserves presence. Semantic normalization is performed later.

### 7.5 TraceId

`trace_id` is optional:

``` proto
optional string trace_id = ...;
```

Contracts must not generate, infer, or replace a missing TraceId.

Core trace enrichment behavior is not a wire default.

### 7.6 CorrelationId

`correlation_id` is optional:

``` proto
optional string correlation_id = ...;
```

Contracts preserves presence. Semantic normalization occurs later.

### 7.7 Exception

`exception` is optional textual diagnostic information:

``` proto
optional string exception = ...;
```

The value is opaque textual observability data. It may contain plain
text, JSON, multiline output, or another useful textual representation
from the originating language/platform.

Contracts must not:

-   require JSON;
-   parse an exception schema;
-   reconstruct a native `.NET Exception`;
-   require preservation of the original exception runtime type.

Empty or whitespace-only exception text is normalized to absence by the
appropriate semantic mapping layer rather than causing the log to fail.

### 7.8 Metadata

Metadata is represented as:

``` proto
map<string, MetadataValue> metadata = ...;
```

A protobuf map does not preserve a distinction between absent and empty.

Therefore:

-   field not sent -\> no metadata;
-   empty map -\> no metadata;
-   map with entries -\> metadata supplied.

This is compatible with the domain because no semantic distinction
between absent and empty Metadata is required.

------------------------------------------------------------------------

## 8. MetadataValue

Metadata is constrained to semantic categories that can be represented
consistently across languages, providers, persistence technologies, and
output formats.

The supported categories are:

-   text/string
-   boolean
-   integer
-   floating-point
-   decimal
-   GUID/UUID
-   date/time
-   duration
-   enum

Null is not a retained metadata value and therefore has no `oneof`
branch.

Raw bytes are not part of Contracts v1 metadata.

The conceptual structure is:

``` proto
message MetadataValue {
  oneof value {
    string string_value = 1;
    bool boolean_value = 2;
    int64 integer_value = 3;
    double floating_point_value = 4;
    google.type.Decimal decimal_value = 5;
    string uuid_value = 6;
    google.protobuf.Timestamp timestamp_value = 7;
    google.protobuf.Duration duration_value = 8;
    string enum_value = 9;
  }
}
```

Map order has no semantic meaning.

### 8.1 Reserved metadata keys

Metadata must not redefine structural LogEntry properties.

The reserved keys are, case-insensitively:

-   `Message`
-   `Level`
-   `Timestamp`
-   `Source`
-   `TraceId`
-   `CorrelationId`
-   `Exception`
-   `Metadata`

Reserved-key validation is a semantic ingress/domain concern rather than
a reason to complicate the raw protobuf map.

### 8.2 Decimal

Decimal values require a precision-preserving, language-neutral
representation.

Contracts v1 uses `google.type.Decimal`, a canonical string-backed decimal
representation that preserves decimal precision across language boundaries.

Transport-specific decimal validation, parsing, canonicalization, and
conversion belong in Contracts-owned adaptation code so normal .NET
consumers and the Service can work with semantic decimal values rather
than arbitrary strings.

The protobuf schema must import `google/type/decimal.proto` and use
`google.type.Decimal` for the `decimal_value` branch. Contracts-owned
adaptation code is responsible for representation-specific validation,
parsing, canonicalization, and conversion.

Codex must not replace decimal with `double`, an HM-owned decimal wire
type, or another lossy representation.

For .NET semantic conversion, an inbound `decimal_value` is valid only when it
can be converted to `System.Decimal` without loss of value. Contracts-owned
conversion must use the canonical Google decimal conversion behavior rather
than implementing an independent decimal parser. Malformed values, values
outside the `System.Decimal` range, and values whose conversion would lose the
supplied decimal value are invalid representations and must be rejected by the
operation before any log is written or Flow state is mutated.

### 8.3 UUID

`uuid_value` preserves the UUID semantic category independently of any
language-specific UUID/GUID type.

A supplied UUID must use the canonical hyphenated 36-character representation:

``` text
xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

Hexadecimal characters are accepted case-insensitively and the normalized
representation is lowercase. Contracts does not impose a UUID version or
variant restriction beyond validity of the UUID representation.

### 8.4 Enum

`enum_value` preserves that the originating metadata value belonged to the
enum semantic category even though its native enum type is not transported.
The wire representation is textual.

Contracts does not attempt to reconstruct or validate the originating enum
type, its declared members, numeric backing type, assembly/module identity, or
flags definition. The textual representation is treated as opaque enum data.
This allows ordinary symbolic names, flags representations, and other textual
representations produced by the originating enum implementation without
coupling the protocol to a programming-language type system.

### 8.5 Floating-point special values

`floating_point_value` preserves the complete protobuf `double` value space.
`NaN`, positive infinity, and negative infinity are valid metadata values and
must not be rejected or transformed by Contracts.

If a later Provider or output format cannot represent one of these values
natively, adaptation to that destination is the responsibility of that
Provider. Contracts must not discard or reinterpret the value for that reason.

### 8.6 Empty MetadataValue

A `MetadataValue` whose `oneof value` has no branch set represents no retained
metadata value. The corresponding metadata entry is discarded during
normalization rather than causing the containing Log or LogContext operation
to fail.

The same resilience principle applies to metadata entries whose key or textual
value becomes empty during normalization: incomplete entries are removed and
are not persisted or added to the normalized context.

This rule does not apply to an explicitly supplied value whose representation
is invalid for its declared branch. Such a value is a validation failure, not
an absent value.

### 8.7 Invalid typed representations

A value explicitly supplied through a typed Contracts representation must be
valid for that representation. Contracts and Service must not silently repair,
reinterpret, or discard an explicitly supplied malformed typed value.

This rule applies, among others, to:

- invalid `google.protobuf.Timestamp` values;
- invalid `google.protobuf.Duration` values;
- malformed UUID representations;
- malformed, out-of-range, or lossy decimal representations.

When such invalid data is received by an RPC, the operation fails with gRPC
`INVALID_ARGUMENT`. No log is written and no Flow mutation is performed.

Absence and normalizable empty data remain distinct from malformed explicit
data: absence/emptiness follows the established resilience and normalization
rules, while an explicitly invalid typed representation is rejected.

------------------------------------------------------------------------

## 9. LogContext

Contracts v1 represents only contextual defaults supported by the
current HM Logging context model:

``` proto
message LogContext {
  optional string source = 1;
  optional string trace_id = 2;
  optional string correlation_id = 3;
  map<string, MetadataValue> metadata = 4;
}
```

It must not contain:

-   Message
-   Level
-   Timestamp
-   Exception
-   FlowId

Flow identity is protocol/runtime state and is not part of LogContext.

### 9.1 Empty contexts

For distributed scope operations, a LogContext that becomes completely
empty after normalization is invalid because pushing or comparing such a
context has no semantic purpose.

A meaningful normalized LogContext contains at least one of:

-   Source
-   TraceId
-   CorrelationId
-   Metadata entry

For `PushScope`:

``` text
INVALID_ARGUMENT
"The log context must contain at least one value."
```

For a present `PopScope.expected_context` that normalizes to empty:

``` text
INVALID_ARGUMENT
"The expected log context must contain at least one value."
```

Absence of `expected_context` itself remains valid and means an
unconditional Pop.

------------------------------------------------------------------------

## 10. LogContext Equality

Two LogContexts are equivalent when, after domain normalization:

-   all contextual properties contain the same values; and
-   Metadata contains exactly the same keys and values.

Metadata map order does not affect equality.

This definition is used by:

-   the consecutive-identical-scope rule in `PushScope`;
-   `PopScope.expected_context`.

------------------------------------------------------------------------

## 11. Logging Flow Contract

A Logging Flow is server-managed remote contextual state identified by
an opaque FlowId.

Contracts exposes Flow operations but does not implement Flow state.

FlowId is:

-   opaque;
-   not a TraceId;
-   not a CorrelationId;
-   not a business identifier;
-   explicitly created through `CreateFlow`.

`PushScope` must never create a missing Flow implicitly.

A Flow may remain active with an empty scope stack. Removing the final
scope does not close it.

Closure is explicit through `CloseFlow`, or occurs through Service-owned
expiration policy.

------------------------------------------------------------------------

## 12. gRPC Service Operations

Contracts v1 exposes five conceptual RPC operations:

``` proto
service LoggingService {
  rpc CreateFlow(CreateFlowRequest) returns (CreateFlowResponse);
  rpc CloseFlow(CloseFlowRequest) returns (CloseFlowResponse);
  rpc PushScope(PushScopeRequest) returns (PushScopeResponse);
  rpc PopScope(PopScopeRequest) returns (PopScopeResponse);
  rpc Log(LogRequest) returns (LogResponse);
}
```

The protobuf service identifier is `LoggingService` under the approved
`hm.logging.contracts.v1` package. Proto file organization must preserve
this architecture and naming hierarchy and must not introduce a broader
ecosystem namespace that obscures `Hm.Logging.Contracts`.

------------------------------------------------------------------------

## 13. gRPC Status and Semantic Results

Contracts distinguishes transport/operation failure from valid semantic
outcomes.

### 13.1 Rule

-   gRPC status answers: **Could the requested operation execute
    validly?**
-   response result answers: **What valid outcome occurred?**

HTTP numeric status codes must not be embedded in protobuf messages.

Typical invalid/error states use gRPC statuses such as:

-   `INVALID_ARGUMENT`
-   `NOT_FOUND`
-   `FAILED_PRECONDITION`
-   `INTERNAL`

Valid no-op or resilience outcomes use gRPC `OK` plus a typed semantic
result when multiple valid outcomes exist.

### 13.2 Result enums

RPC result enums use an `UNSPECIFIED = 0` defensive/default value.

Example:

``` proto
enum PopScopeResult {
  POP_SCOPE_RESULT_UNSPECIFIED = 0;
  ...
}
```

`UNSPECIFIED` is not a functional result and the Service must not
intentionally return it.

This convention applies to RPC-result enums, not to domain enums such as
`LogLevel`.

------------------------------------------------------------------------

## 14. CreateFlow

### 14.1 Contract

``` proto
message CreateFlowRequest {}

message CreateFlowResponse {
  string flow_id = 1;
  string message = 2;
}
```

### 14.2 Semantics

`CreateFlow` has no input configuration and creates one new empty Flow.

On success:

-   gRPC status: `OK`
-   a new opaque FlowId is returned
-   message: `"Flow created successfully."`

There is no result enum because there is only one valid success outcome.

### 14.3 Retry behavior

`CreateFlow` is **not idempotent**.

If the Service creates a Flow but the response is lost, retrying may
create a second Flow. The client uses the FlowId it successfully
receives. An unused orphan Flow is eventually handled by Service-owned
expiration.

Contracts v1 does not introduce an operation ID or retry token for
CreateFlow.

------------------------------------------------------------------------

## 15. CloseFlow

### 15.1 Request

``` proto
message CloseFlowRequest {
  string flow_id = 1;
}
```

`flow_id` is required semantically.

Absent, empty, or whitespace-only FlowId:

-   gRPC status: `INVALID_ARGUMENT`
-   message: `"The flow ID is required."`

### 15.2 Result

``` proto
enum CloseFlowResult {
  CLOSE_FLOW_RESULT_UNSPECIFIED = 0;
  CLOSE_FLOW_RESULT_CLOSED = 1;
  CLOSE_FLOW_RESULT_ALREADY_CLOSED = 2;
}

message CloseFlowResponse {
  CloseFlowResult result = 1;
  string message = 2;
}
```

Functional outcomes:

**CLOSED**

-   The Flow was active and is now closed.
-   gRPC status: `OK`
-   message: `"Flow closed successfully."`

**ALREADY_CLOSED**

-   The supplied FlowId is syntactically valid but no active Flow exists
    for it.
-   This includes a Flow that never existed, was already closed, or
    expired.
-   gRPC status: `OK`
-   message:
    `"The flow is already closed or has expired. No action was required."`

The semantic result intentionally does not require the Service to
distinguish among never-existing, previously closed, and expired
identifiers.

### 15.3 Idempotency

`CloseFlow` is idempotent by postcondition.

After any successful CloseFlow call, the requested Flow is not active.
Retrying the operation preserves that state.

No tombstone/history of closed FlowIds is required solely to support
this contract.

------------------------------------------------------------------------

## 16. PushScope

### 16.1 Request

``` proto
message PushScopeRequest {
  string flow_id = 1;
  LogContext context = 2;
}
```

Both fields are semantically required.

Invalid FlowId:

``` text
INVALID_ARGUMENT
"The flow ID is required."
```

Missing context:

``` text
INVALID_ARGUMENT
"The log context is required."
```

Context empty after normalization:

``` text
INVALID_ARGUMENT
"The log context must contain at least one value."
```

A syntactically valid FlowId that does not reference an active Flow:

``` text
NOT_FOUND
"The specified flow does not exist, has been closed, or has expired."
```

### 16.2 Result

``` proto
enum PushScopeResult {
  PUSH_SCOPE_RESULT_UNSPECIFIED = 0;
  PUSH_SCOPE_RESULT_ADDED = 1;
  PUSH_SCOPE_RESULT_ALREADY_EXISTS = 2;
}

message PushScopeResponse {
  PushScopeResult result = 1;
  string message = 2;
}
```

**ADDED**

-   The normalized context differs from the current top scope.
-   A new stack level is added.
-   gRPC status: `OK`
-   message: `"Scope added to the flow."`

**ALREADY_EXISTS**

-   An equivalent normalized context is already at the top.
-   No scope is added.
-   gRPC status: `OK`
-   message:
    `"An equivalent scope is already at the top of the flow. No scope was added."`

### 16.3 Consecutive-identical-scope rule

A Flow must not contain two consecutive equivalent normalized contexts.

Example:

``` text
[A, B] + PushScope(B) -> [A, B]           // ALREADY_EXISTS
[A, B, C] + PushScope(B) -> [A, B, C, B] // ADDED
```

### 16.4 Retry behavior

`PushScope` is retry-safe under this rule.

If the first request succeeds but its response is lost, retrying the
same normalized context sees that context at the top and returns
`ALREADY_EXISTS` without creating a duplicate scope.

No retry flag, operation ID, or ScopeId is required in v1.

------------------------------------------------------------------------

## 17. PopScope

### 17.1 Request

``` proto
message PopScopeRequest {
  string flow_id = 1;
  optional LogContext expected_context = 2;
}
```

`flow_id` is semantically required.

Absent, empty, or whitespace-only FlowId:

``` text
INVALID_ARGUMENT
"The flow ID is required."
```

A syntactically valid FlowId that does not reference an active Flow:

``` text
NOT_FOUND
"The specified flow does not exist, has been closed, or has expired."
```

If `expected_context` is present but becomes empty after normalization:

``` text
INVALID_ARGUMENT
"The expected log context must contain at least one value."
```

### 17.2 Result

``` proto
enum PopScopeResult {
  POP_SCOPE_RESULT_UNSPECIFIED = 0;
  POP_SCOPE_RESULT_REMOVED = 1;
  POP_SCOPE_RESULT_NO_SCOPES = 2;
  POP_SCOPE_RESULT_CONTEXT_MISMATCH = 3;
}

message PopScopeResponse {
  PopScopeResult result = 1;
  string message = 2;
}
```

**REMOVED**

-   The current top scope was removed.
-   gRPC status: `OK`
-   message: `"Scope removed from the flow."`

**NO_SCOPES**

-   The Flow is active but its scope stack is empty.
-   No state is changed.
-   gRPC status: `OK`
-   message: `"The flow contains no scopes. No scope was removed."`

**CONTEXT_MISMATCH**

-   `expected_context` was supplied and does not equal the normalized
    top context.
-   No state is changed.
-   gRPC status: `OK`
-   message:
    `"The scope at the top of the flow does not match the expected context. No scope was removed."`

### 17.3 Retry behavior

`PopScope(flow_id)` without `expected_context` is not retry-safe. If a
successful response is lost, a blind retry could remove the next scope.

`PopScope(flow_id, expected_context)` is the retry-safe form.

If the first request never reached the Service, the expected scope
remains at the top and the retry removes it.

If the first request succeeded but its response was lost, the previous
expected context is no longer at the top. The retry therefore returns
`CONTEXT_MISMATCH` without removing another scope.

The consecutive-identical-scope rule prevents a successful Pop from
immediately exposing an identical scope and accidentally removing it on
retry.

------------------------------------------------------------------------

## 18. Log

### 18.1 Request

``` proto
message LogRequest {
  optional string flow_id = 1;
  LogEntry entry = 2;
}
```

`entry` is semantically required.

Missing entry:

``` text
INVALID_ARGUMENT
"The log entry is required."
```

Invalid Message causes `INVALID_ARGUMENT` and the log is not written.

### 18.2 FlowId behavior

`flow_id` is optional.

**FlowId absent**

The log is valid and is processed independently without Logging Flow
context.

**FlowId present and malformed**

Empty or whitespace-only FlowId is invalid supplied data:

-   gRPC status: `INVALID_ARGUMENT`
-   the log is not written.

**FlowId present, syntactically valid, and active**

The Service applies the Flow's effective context and processes the log.

**FlowId present, syntactically valid, but inactive**

The log must not be lost merely because the requested Flow is
unavailable.

The Service processes the log **without Flow context** and reports that
semantic outcome.

This applies when the Flow is nonexistent, closed, or expired.

### 18.3 Result

``` proto
enum LogResult {
  LOG_RESULT_UNSPECIFIED = 0;
  LOG_RESULT_ACCEPTED = 1;
  LOG_RESULT_ACCEPTED_WITHOUT_FLOW = 2;
}
```

**ACCEPTED**

Used when:

-   no FlowId was requested and the independent log was accepted; or
-   an active FlowId was supplied and the log was accepted with
    effective Flow context.

gRPC status: `OK`

Message:

``` text
"Log entry accepted successfully."
```

**ACCEPTED_WITHOUT_FLOW**

Used when a syntactically valid FlowId was supplied but no active Flow
was available.

The log is still accepted without Flow context.

gRPC status: `OK`

Message:

``` text
"The specified flow was not found. The log entry was accepted without flow context."
```

### 18.4 Retry behavior

`Log` is not idempotent.

If the Service writes a log but the response is lost, a retry may create
a duplicate log entry.

Contracts v1 intentionally does not introduce a deduplication ID.

------------------------------------------------------------------------

## 19. Logging Flow Concurrency Semantics

Contracts preserves the following architectural invariants even though
their implementation belongs to Service:

-   Each Flow has a strict LIFO scope stack.
-   `PushScope`, `PopScope`, and `CloseFlow` mutations are atomic and
    serialized per Flow.
-   Different Flows may execute concurrently.
-   The client is responsible for semantic ordering when it issues
    concurrent operations against the same Flow.
-   The Service guarantees state integrity but does not infer client
    business ordering.

The mechanism used to provide serialization --- lock, actor, queue, or
another implementation --- is not part of Contracts.

------------------------------------------------------------------------

## 20. Flow Expiration

Contracts v1 recognizes that a Flow may become inactive through
expiration.

The following are deliberately **not Contracts decisions**:

-   inactivity duration;
-   which operations refresh activity;
-   cleanup scheduling;
-   storage mechanism;
-   expiration implementation;
-   persistence/recovery across Service restarts.

These decisions belong to `Hm.Logging.Service`.

Contracts must not embed an operational TTL policy into the protocol
unless a future architecture decision explicitly requires it.

------------------------------------------------------------------------

## 21. Validation Boundary Summary

Contracts/protobuf structurally constrains the wire shape.

Contracts-owned adapters may validate and convert
representation-specific data such as canonical decimal values.

Service ingress performs defensive semantic validation of untrusted
requests, including:

-   required Message;
-   required FlowId where applicable;
-   malformed supplied optional FlowId;
-   supported/valid LogLevel values;
-   required LogContext where applicable;
-   non-empty normalized LogContext;
-   reserved metadata keys;
-   metadata/domain normalization;
-   validation of explicitly supplied typed representations, including
    Timestamp, Duration, UUID, and decimal values;
-   removal of metadata entries that contain no retained value after
    normalization;
-   mapping into domain/Core semantics.

Explicitly supplied data whose representation is invalid for its declared type
causes `INVALID_ARGUMENT` and must not produce a log or mutate Flow state.
Absent or normalizable empty metadata remains subject to the established
resilience rules and may be discarded during normalization.

Invalid input must not be silently reinterpreted into a different
operation.

------------------------------------------------------------------------

## 22. Quality and CI/CD Policy

`Hm.Logging.Contracts` must follow the same engineering rigor expected
from HM Logging components.

The project requires:

-   automated formatting validation;
-   build with zero warnings;
-   all tests passing;
-   static analysis;
-   SonarCloud Quality Gate;
-   zero new issues;
-   zero relevant known vulnerabilities;
-   zero new duplication;
-   coverage for new handwritten code where applicable;
-   generated protobuf C# excluded from coverage metrics;
-   unit tests for Contracts-owned adapters, conversion, and validation
    code;
-   automated protobuf compatibility / breaking-change validation;
-   publication gated by CI/CD.

The implementation should investigate and adopt an appropriate protobuf
compatibility tool rather than relying only on code review.

Package version must be derived from the release tag. A static manually
maintained package version must not become the release source of truth.

Publication occurs from release tags according to the repository release
process.

------------------------------------------------------------------------

## 23. Documentation Requirements

Public protocol documentation must be written in English.

Each RPC whose behavior involves retries, idempotency, resilience, or
multiple semantic outcomes must document:

-   design intent;
-   each functional result;
-   whether the outcome represents mutation, no-op, or error;
-   relationship between response results and gRPC status;
-   retry behavior;
-   examples where they materially clarify semantics.

Documentation must make the intended behavior understandable without
requiring consumers to infer semantics solely from enum names.

------------------------------------------------------------------------

## 24. Explicitly Out of Scope for Contracts v1

The following must not be designed or implemented inside Contracts
unless the architecture is explicitly revised:

-   Flow storage.
-   Flow TTL configuration.
-   Activity refresh rules.
-   Cleanup algorithms.
-   Service persistence or recovery.
-   Per-Flow locking/actor/queue implementation.
-   Provider selection.
-   Log persistence.
-   Provider failover.
-   Core local scope implementation.
-   Core trace enrichment.
-   OpenTelemetry spans, metrics, baggage, or telemetry pipeline
    semantics.
-   Authentication/authorization or Flow ownership policy unless
    separately specified.
-   Deduplication of Log requests.
-   Operation IDs, retry flags, or ScopeIds.
-   Reconstructing native exceptions from textual exception information.

------------------------------------------------------------------------

## 25. Implementation Governance

This architecture is intended to be directly consumable by
implementation agents and developers.

When implementing `Hm.Logging.Contracts`, the implementation must:

1.  Read the canonical Logging domain specification.
2.  Read the distributed logging architecture.
3.  Read this Contracts architecture.
4.  Inspect the current `Hm.Logging` implementation when mapping
    established .NET/domain concepts.
5.  Preserve the boundaries and semantics defined by those sources.
6.  Stop and report any ambiguity, contradiction, or missing
    architectural decision that would materially affect the public
    protocol.
7.  Never silently redesign the protocol for implementation convenience.

Implementation prompts are work instructions, not architectural sources
of truth.

------------------------------------------------------------------------

## 26. Established Contracts v1 Baseline

The following decisions are closed for the initial Contracts v1
implementation:

-   `Hm.Logging.Contracts` is independent from Core, Service, and
    Providers.
-   Protobuf package is `hm.logging.contracts.v1`.
-   Protobuf service identifier is `LoggingService`.
-   Generated C# namespace is `Hm.Logging.Contracts`.
-   NuGet package is `HDev.Hm.Logging.Contracts`.
-   Package major equals protocol major.
-   Initial preview versioning begins at `1.0.0-preview.1`.
-   Preview maturity is not encoded in protobuf identities.
-   `LogLevel` preserves Trace=0 through Critical=5.
-   Optional domain scalars preserve protobuf presence where required.
-   Timestamp uses protobuf Timestamp and Contracts does not generate
    its default.
-   TraceId remains optional and Contracts does not generate it.
-   Exception is optional opaque textual diagnostic information.
-   Metadata uses typed `MetadataValue` with fixed field numbers following
    the established semantic type order.
-   Decimal uses `google.type.Decimal` and Contracts owns
    representation-specific validation and conversion.
-   Decimal-to-`.NET decimal` conversion must be lossless; malformed,
    out-of-range, or lossy representations are invalid.
-   The .NET Contracts implementation targets `net10.0`.
-   HM-owned `.proto` schemas are distributed with the NuGet package and must
    exactly match the corresponding package release; Google Common Protos
    remain externally owned canonical dependencies.
-   UUID metadata uses canonical hyphenated UUID text and normalizes hexadecimal
    characters to lowercase.
-   Enum metadata preserves the enum semantic category as opaque textual data
    without reconstructing the originating enum type.
-   Floating-point metadata accepts and preserves `NaN`, positive infinity,
    and negative infinity.
-   Metadata entries with no retained value after normalization are discarded
    rather than failing the containing operation.
-   Explicitly supplied malformed typed representations are rejected with
    `INVALID_ARGUMENT` and produce no log or Flow mutation.
-   LogContext contains Source, TraceId, CorrelationId, and Metadata
    only.
-   Empty normalized distributed LogContext is invalid.
-   FlowId is opaque and external to LogContext.
-   Flows are explicitly created and explicitly closed.
-   Final Pop does not close a Flow.
-   Consecutive equivalent scopes are not duplicated.
-   `PushScope` is retry-safe through top-context equality.
-   blind `PopScope` is not retry-safe.
-   guarded `PopScope(expected_context)` is retry-safe.
-   `CloseFlow` is idempotent by postcondition.
-   valid but inactive FlowId on CloseFlow returns `ALREADY_CLOSED`.
-   valid but inactive FlowId on Log does not lose the log; it returns
    `ACCEPTED_WITHOUT_FLOW`.
-   `Log` remains non-idempotent.
-   gRPC statuses represent invalid/failed operations; typed results
    represent valid semantic outcomes.
-   RPC-result enums reserve numeric zero for `UNSPECIFIED`.
-   Service runtime expiration policy remains outside Contracts.

This baseline must be treated as stable input to the first
implementation of `Hm.Logging.Contracts`.
