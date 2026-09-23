# HM Logging Contracts Architecture

**Component:** `Hm.Logging.Contracts`\
**Protocol:** `hm.logging.contracts.v1`\
**NuGet package:** `HDev.Hm.Logging.Contracts`\
**Status:** Contracts v1 architecture implemented and publicly released as `v1.0.0-preview.1`\
**Date:** 22 September 2026

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
-   Define which wire representations are valid without coupling those
    representations to a particular runtime type system.
-   Provide generated .NET gRPC/protobuf types as a language-specific
    distribution of the canonical schemas.
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

From the first publicly distributed release of protocol v1, including a
preview or prerelease:

-   Existing field numbers must not be reused.
-   Existing enum numeric values must not be renumbered.
-   Removed fields should be declared `reserved`.
-   Backward-compatible additions may evolve within v1.
-   An incompatible protocol change requires protocol v2 and package
    major 2. Contracts v1 does not permit breaking-change overrides within
    the existing protobuf API version.

During preview releases, the schema MAY evolve through
backward-compatible changes within the current protobuf API version.

The first publicly distributed preview or prerelease establishes the
compatibility baseline for that API version.

Breaking wire-contract changes MUST NOT be introduced within the same
API version solely because the distribution is still in preview.

An incompatible change requires a new protobuf API version (for example,
`v2`). Contracts v1 does not provide a compatibility override or bypass for
a failing breaking-change validation.

### 4.5 .NET target and schema distribution

The initial `Hm.Logging.Contracts` implementation targets `net10.0`,
aligned with the HM .NET ecosystem baseline. Contracts remains
language-neutral at the protocol level even though the official package
initially targets .NET 10.

HM-owned `.proto` schemas are the canonical protocol source and must be
included in the `HDev.Hm.Logging.Contracts` NuGet package alongside the
compiled .NET protobuf/gRPC types. The schemas distributed with a
package release must correspond exactly to that package version.

The canonical `google.type.Decimal` definition remains owned by Google
Common Protos. HM Logging must not fork or redefine that schema as an
HM-owned type. The .NET implementation uses the canonical Google Common
Protos support needed for `Google.Type.Decimal`, while protobuf
compilation resolves the canonical `google/type/decimal.proto` import
through the package/source import mechanism.

Official non-.NET SDKs are outside the current Contracts scope. Non-.NET
consumers may use the published language-neutral schemas to generate and
maintain clients in their own language/toolchain.

### 4.6 NuGet package layout and consumer behavior

`HDev.Hm.Logging.Contracts` is a compiled .NET distribution of the
canonical Contracts v1 schemas. The package must contain the compiled
assembly and its public XML documentation together with the HM-owned
`.proto` schemas that correspond exactly to the package release.

HM-owned Contracts schemas are distributed under:

``` text
content/protos/hm/logging/contracts/v1/
```

preserving their canonical relative hierarchy.

The schemas are included for discovery, interoperability, inspection,
and consumer-controlled generation. The package must not inject those
schemas into a consuming project through `build`, `buildTransitive`, or
equivalent automatic compilation behavior. Consumers that choose to
compile the schemas own that integration explicitly.

External schemas, including Google Common Protos, remain externally
owned dependencies and must not be repackaged or presented as HM-owned
schemas.

The public package should include the repository README as package
documentation and the approved HM package icon before the first public
release.

### 4.7 .NET source mapping and symbols

The public NuGet distribution should provide Source Link-compatible
source mapping, portable PDB information, and a symbol package
(`.snupkg`) when supported by the publication target.

Symbols and source mapping must correspond to the exact validated source
state and compiled binaries represented by the `.nupkg`. They are
distribution and debugging aids; they do not become a source of protocol
semantics.

### 4.8 Assembly and package version identity

`PackageVersion` is derived from the release SemVer tag and is the
public distribution version. A manually maintained static package
version must not become the release source of truth.

For Contracts v1, `AssemblyVersion` remains stable at `1.0.0.0`
throughout the package major version 1 line unless a later architecture
decision establishes a concrete compatibility reason to change it.
Package prerelease, patch, and minor evolution is expressed through
`PackageVersion`, not by continuously changing assembly identity.

`FileVersion` is derived from the numeric release version as
`major.minor.patch.0`. Prerelease identifiers are not represented in
`FileVersion`. For example, `1.0.0-preview.1` produces `1.0.0.0`.

`InformationalVersion` preserves the complete SemVer release identity,
including any prerelease identifier, and MAY additionally include source
revision metadata when produced deterministically by the build. For
example, a release may identify itself as `1.0.0-preview.1` while
retaining traceability to the exact release commit.

`PackageVersion`, `FileVersion`, and `InformationalVersion` must be
derived from the same release identity rather than maintained as
independent manual version values.

Release-specific notes belong to the corresponding GitHub Release or
equivalent release record. Static package metadata must not preserve
release notes in a form that becomes stale across later package
versions.

### 4.9 BSR distribution and compatibility baseline

The public language-neutral protobuf distribution is published as:

``` text
buf.build/hdev-hm/logging
```

The repository directory `proto/` is the protobuf module root. BSR must
therefore preserve the canonical relative schema hierarchy exactly as:

``` text
hm/logging/contracts/v1/
```

The BSR module publishes only HM-owned schemas. External schemas such as
Google Common Protos remain external BSR dependencies and must be resolved
through the declared Buf dependency graph and committed `buf.lock`.

Each public release uses the same release identity across channels:

- Git release tag: `v<SemVer>`;
- NuGet package version: `<SemVer>`;
- BSR release label: `v<SemVer>`;
- exact Git source commit associated with both distributions.

Release labels are explicit. Branch names must not be published as official
release labels.

Breaking-change validation uses the immutable BSR commit from the last
successful BSR publication as its baseline. The repository-level GitHub
Actions variable `LAST_BSR_COMMIT_ID` stores that operational state.

`LAST_BSR_COMMIT_ID` has exactly three valid states:

- `INITIAL`: intentionally establish a new compatibility baseline and skip
  `buf breaking` for that publication;
- a valid immutable commit ID belonging to `buf.build/hdev-hm/logging`:
  `buf breaking` is mandatory against that snapshot;
- missing, empty, malformed, or unresolvable value: release validation fails.

`INITIAL` is an explicit administrative compatibility decision, not an
automatic fallback for missing state and not a bypass for an ordinary
breaking change within Contracts v1.

After a successful `buf push`, the resulting immutable BSR commit ID becomes
the new `LAST_BSR_COMMIT_ID`. The variable must not change when BSR
publication fails.

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

Contracts defines the interoperable wire representation and preserves
presence and type information required by the protocol. It does not own
runtime conversion into language-specific native types.

The canonical `.proto` schemas may define what constitutes a valid wire
representation, but language-specific parsing, conversion,
normalization, and mapping belong to the implementation consuming the
protocol.

`Hm.Logging.Service`, as the HM .NET implementation, performs defensive
ingress validation and maps valid Contracts representations into the
domain/Core model before executing the logging pipeline.

Other language implementations are responsible for equivalent validation
and mapping appropriate to their own runtime while preserving the public
contract semantics.

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

-   Level absent -\> Contracts preserves absence and assigns no wire
    default.
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

-   absent -\> Contracts preserves absence.
-   present -\> preserve the supplied value.

Contracts itself does not apply a default or prescribe how every
implementation must materialize an omitted value. An implementation that
maps the request into the HM Logging domain must satisfy the domain
semantics at that boundary; other consumers of the language-neutral
schema own their own runtime policy.

### 7.3 Timestamp

`timestamp` uses `google.protobuf.Timestamp`.

It is required in the final semantic LogEntry but may be omitted on the
wire.

-   supplied -\> explicit absolute event time.
-   omitted -\> Contracts preserves absence.

Contracts must not generate a timestamp merely because the field was
omitted and does not prescribe a universal runtime default for schema
consumers.

When supplied, the producer must provide an unambiguous absolute instant
compatible with `google.protobuf.Timestamp`. A producer that starts from
a local or otherwise ambiguous date/time is responsible for resolving
its time zone before transmission. The protocol must not infer a time
zone from the receiving Service or silently reinterpret an ambiguous
local time.

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

Metadata is intentionally constrained to scalar semantic categories that
can be represented consistently across languages, providers, persistence
technologies, and output formats.

Contracts v1 supports:

-   text/string;
-   boolean;
-   signed integer;
-   unsigned integer;
-   single-precision floating-point;
-   double-precision floating-point;
-   decimal;
-   date/time;
-   duration.

Null is not a retained metadata value and therefore has no `oneof`
branch. Raw bytes, arbitrary objects, arrays, nested collections, and
object graphs are not part of Contracts v1 metadata.

The conceptual structure is:

``` proto
message MetadataValue {
  oneof value {
    string string_value = 1;
    bool boolean_value = 2;
    int64 signed_integer_value = 3;
    uint64 unsigned_integer_value = 4;
    float float_value = 5;
    double double_value = 6;
    google.type.Decimal decimal_value = 7;
    DateTimeValue date_time_value = 8;
    google.protobuf.Duration duration_value = 9;
  }
}

message DateTimeValue {
  google.protobuf.Timestamp timestamp = 1;
  google.protobuf.Duration utc_offset = 2;
}
```

The exact field numbers become compatibility-sensitive once the first
public schema is published. The implementation must preserve the
semantic ordering above when assigning the initial v1 field numbers.

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

### 8.2 Text and language-specific scalar values

`string_value` is the interoperable textual representation. Contracts
does not attempt to infer additional runtime semantics from its
contents.

Language-specific values that do not require a distinct wire category
may be represented as text by the producer. Examples include:

-   character values;
-   UUID/GUID values;
-   enum names or textual enum representations;
-   other producer-defined textual identifiers.

Contracts does not validate a `string_value` as UUID, enum, character,
or another originating runtime type. The producer owns any such
semantics.

When an application needs to log a complex object or collection, it must
adapt the value before transmission. Valid approaches include flattening
relevant properties into multiple scalar metadata entries, mapping the
object into scalar key-value metadata, or serializing it into a textual
representation such as JSON. Contracts does not reconstruct the original
object model.

### 8.3 Integers

Signed integer metadata uses protobuf `int64`. Native signed integer
types with a smaller range can be represented without loss by widening
to `int64`.

Unsigned integer metadata uses protobuf `uint64`. Native unsigned
integer types with a smaller range can be represented without loss by
widening to `uint64`.

Contracts must not force unsigned values through `int64`, because values
above the signed 64-bit maximum would be lost or rejected unnecessarily.

Each language implementation is responsible for checking that conversion
between its native integer types and the selected wire representation is
lossless.

### 8.4 Floating-point

Single-precision metadata uses protobuf `float`.

Double-precision metadata uses protobuf `double`.

The distinction is preserved rather than widening every floating-point
value to `double`, so the wire representation retains the semantic
precision category supplied by the producer.

Both branches preserve the value space defined by protobuf for their
respective scalar type, including `NaN`, positive infinity, and negative
infinity.

If a later Provider or output format cannot represent one of these
values natively, safe adaptation to that destination is the
responsibility of that Provider. Contracts must not discard or
reinterpret the value for that reason.

### 8.5 Decimal

Decimal values require a precision-preserving, language-neutral
representation.

Contracts v1 uses `google.type.Decimal`, the canonical Google Common
Protos decimal representation. Contracts must not replace it with
`double`, an HM-owned decimal wire type, or another lossy
representation.

The canonical schema defines the wire representation. Validation,
parsing, canonicalization, range checking, and conversion into a native
decimal type belong to the implementation consuming the contract.

For `Hm.Logging.Service`, an explicitly supplied decimal representation
must be validated before any log is written or Flow state is mutated.
Conversion to the .NET native decimal type must be lossless; malformed,
out-of-range, or lossy values are invalid input.

### 8.6 Date/time and UTC offset

A metadata date/time always contains an absolute instant represented by
`google.protobuf.Timestamp`.

The producer is responsible for resolving any local or ambiguous
date/time into an unambiguous absolute instant before transmission.
Contracts and Service must not infer a time zone from the receiver.

`utc_offset` is optional. When present, it preserves the UTC offset
associated with the producer's original date/time representation. It
does not alter the absolute instant stored in `timestamp`.

For example, `13:30 -05:00` and `18:30 UTC` identify the same absolute
instant. The timestamp carries that instant; the optional offset
preserves that the originating representation used `-05:00`.

An offset is not a time-zone identifier and must not be documented as
one.

### 8.7 Duration

Duration metadata uses `google.protobuf.Duration`.

The wire value must satisfy the protobuf Duration representation rules.
Conversion to any language-specific duration type is the responsibility
of the consuming implementation.

### 8.8 Empty MetadataValue

A `MetadataValue` whose `oneof value` has no branch set represents no
retained metadata value. The corresponding metadata entry is discarded
during normalization rather than causing the containing Log or
LogContext operation to fail.

The same resilience principle applies to metadata entries whose key or
textual value becomes empty during normalization: incomplete entries are
removed and are not persisted or added to the normalized context.

This rule does not apply to an explicitly supplied value whose
representation is invalid for its declared branch. Such a value is a
validation failure, not an absent value.

### 8.9 Invalid typed representations

A value explicitly supplied through a typed Contracts representation
must be valid for that representation. Implementations must not silently
repair, reinterpret, or discard an explicitly malformed typed value.

This rule applies, among others, to:

-   invalid `google.protobuf.Timestamp` values;
-   invalid `google.protobuf.Duration` values;
-   malformed, out-of-range, or lossy decimal representations;
-   integer conversions that cannot be represented losslessly in the
    target implementation.

When such invalid data is received by `Hm.Logging.Service`, the
operation fails with gRPC `INVALID_ARGUMENT`. No log is written and no
Flow mutation is performed.

Absence and normalizable empty data remain distinct from malformed
explicit data: absence/emptiness follows the established resilience and
normalization rules, while an explicitly invalid typed representation is
rejected.

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

Contracts/protobuf structurally constrains the wire shape and documents
the valid semantics of each representation. Contracts does not own
language-specific runtime conversion.

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
    Timestamp, UTC offset, Duration, integer ranges, and decimal values;
-   removal of metadata entries that contain no retained value after
    normalization;
-   mapping into domain/Core semantics.

Explicitly supplied data whose representation is invalid for its
declared type causes `INVALID_ARGUMENT` and must not produce a log or
mutate Flow state. Absent or normalizable empty metadata remains subject
to the established resilience rules and may be discarded during
normalization.

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
-   tests for HM-authored contract helpers or generated-code integration
    where applicable;
-   automated protobuf compatibility / breaking-change validation;
-   publication gated by CI/CD.

Buf is the protobuf governance tool for formatting, linting, build, dependency
resolution, and compatibility validation. `buf.lock` is committed and CI must
verify that it remains consistent with the dependencies declared by the
project before release publication.

### 22.1 Validation gates by phase

Validation is layered so that each phase gates the next one:

1. **Local / pre-push** provides fast deterministic feedback: formatting,
   Release build with zero warnings, automated tests, and applicable local Buf
   validation. Expensive remote analysis such as SonarCloud is not required
   locally.
2. **Pull Request** is the authoritative integration gate. It runs the full
   applicable quality suite, including formatting, build/static analysis,
   tests and coverage, SonarCloud analysis with blocking Quality Gate,
   dependency/security review, Buf validation, and package validation where
   applicable. A mandatory failure blocks merge.
3. **Main** revalidates the integrated commit, including SonarCloud Quality
   Gate, so a release tag is created only from an integrated state that has
   passed the required main-branch validations.
4. **Release** validates release identity and the exact tagged commit rather
   than using publication as a substitute for earlier quality gates. It
   verifies canonical SemVer/tag rules, source-commit identity, package
   identity, `buf.lock`, BSR compatibility baseline, and all publication
   prerequisites before any public channel is mutated.

Normal development changes must not advance to the next phase when the current
phase has a mandatory validation failure.

### 22.2 Release publication order and recovery

Official publication is triggered only by an authorized HM SemVer release tag
on a validated commit in the protected `main` history. Release executions for
Contracts are serialized so that two publication runs cannot concurrently
modify BSR release state.

All deterministic BSR checks must complete before NuGet publication. Public
channels are then updated in this order:

1. publish or verify the NuGet release;
2. publish or verify the corresponding BSR release;
3. persist the successful BSR commit as `LAST_BSR_COMMIT_ID`;
4. perform post-publication records such as the GitHub Release.

NuGet and BSR together constitute successful contract distribution. Failure of
a later GitHub Release step does not make an already successful NuGet + BSR
distribution unpublished.

`buf push` uses at most two attempts. Each attempt has a 60-second timeout and
the second attempt follows a 10-second backoff. If both attempts fail, the
release run fails and `LAST_BSR_COMMIT_ID` remains unchanged.

Updating `LAST_BSR_COMMIT_ID` uses at most two attempts, each with a 15-second
timeout and a 10-second backoff. The update uses a dedicated least-privilege
GitHub credential scoped only to repository Variables read/write access.

Release reruns must be idempotent. If NuGet or BSR already contains the
release, automation must verify that the existing publication corresponds to
the exact tagged Git source state before skipping publication and continuing
recovery. NuGet `RepositoryCommit` records the exact source commit used for
this verification. BSR publication metadata must preserve equivalent
traceability to the same source commit.

If the same release identity exists in NuGet and BSR but resolves to different
source states, automation must stop. It must not rewrite either published
release automatically; remediation requires investigation and a new corrective
version when appropriate.

A failure during publication is recovered by retrying/resuming the same
release. A defect discovered after successful publication is corrected by a
new higher version, not by rewriting or rolling back an existing SemVer
release identity.

### 22.3 Publication credentials and artifacts

BSR publication uses `BUF_TOKEN` stored as a GitHub Secret. The current BSR
credential is a dedicated personal token with Full access because the available
limited-token configuration did not expose the push/write permissions required
for this publication workflow. This is an explicitly accepted security debt, not
the desired steady-state permission model; a narrower CI credential or service
identity SHOULD replace it when BSR supports one that can publish the module.
`LAST_BSR_COMMIT_ID` is non-sensitive operational state and is stored as a
GitHub Actions Variable.

PR artifacts are generated only as needed for validation and do not need
long-term retention. Main and release diagnostic artifacts may use short
retention (currently seven days) when useful for troubleshooting. GitHub
workflow logs follow repository log-retention policy and are not duplicated as
separate artifacts solely for archival purposes.

The release build must produce the NuGet package and its corresponding symbol
package from the same validated source state. Source mapping, portable symbols,
XML documentation, package documentation, package icon, and distributed
HM-owned schemas must describe that same release.

Release-specific notes are maintained by the GitHub Release or equivalent
release record rather than as stale static package metadata.

Publication occurs from release tags according to the repository release
process. Public release artifacts include the provenance/attestation required
by the HM delivery standard and must remain traceable to the exact validated
source state.

------------------------------------------------------------------------

## 23. Documentation Requirements

Public protocol documentation must be written in English.

Canonical `.proto` comments and protocol documentation must describe
semantics in language-neutral terms. They must not make a .NET type or
behavior a requirement unless that concern is intrinsically part of the
wire contract.

Language-specific distributions may provide implementation examples. The
initial NuGet documentation may use C#/.NET examples, but those examples
must be clearly identified as language-specific guidance rather than
protocol requirements.

Important representation rules, including timestamp/UTC behavior,
optional field semantics, metadata categories, and invalid-input
behavior, must be discoverable from the public contract documentation.
Where generated tooling preserves schema comments, those comments should
provide useful IDE guidance without making generated code the source of
truth.

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
-   Timestamp uses protobuf Timestamp, represents an absolute instant,
    and Contracts does not generate its default or infer a receiver time
    zone.
-   TraceId remains optional and Contracts does not generate it.
-   Exception is optional opaque textual diagnostic information.
-   Metadata uses scalar typed `MetadataValue`; arbitrary objects,
    collections, arrays, and object graphs are not wire metadata values.
-   Signed integers use `int64`; unsigned integers use `uint64`.
-   Single-precision and double-precision floating-point values remain
    distinct through protobuf `float` and `double`.
-   Character, UUID/GUID, enum, and other producer-defined textual
    values use `string_value`; Contracts does not infer or validate
    their originating runtime type.
-   Decimal uses canonical `google.type.Decimal`; native parsing,
    validation, and conversion belong to the consuming implementation.
-   `Hm.Logging.Service` must reject malformed, out-of-range, or lossy
    decimal values before logging or mutating Flow state.
-   Metadata date/time uses an absolute protobuf Timestamp plus an
    optional UTC offset that preserves the originating offset without
    changing the instant.
-   The .NET Contracts implementation targets `net10.0`.
-   HM-owned `.proto` schemas are distributed with the NuGet package
    under `content/protos/hm/logging/contracts/v1/` and must exactly
    match the corresponding package release; they are not automatically
    injected into consumer builds through `build` or `buildTransitive`.
-   Google Common Protos remain externally owned canonical dependencies
    and are not redistributed as HM-owned schemas.
-   The public NuGet package includes compiled binaries, XML
    documentation, package documentation, and the approved package icon.
-   Public .NET distribution provides Source Link-compatible source
    mapping, portable symbols, and `.snupkg` publication when supported.
-   `PackageVersion` comes from the release SemVer tag; Contracts v1
    keeps `AssemblyVersion` stable at `1.0.0.0` unless a later
    compatibility decision requires otherwise.
-   Release-specific notes belong to the corresponding GitHub Release or
    equivalent release record rather than stale static package metadata.
-   Complex producer values may be flattened into scalar metadata or
    serialized to text before transmission.
-   Floating-point metadata accepts and preserves the protobuf special
    values supported by the selected `float` or `double` branch.
-   Metadata entries with no retained value after normalization are
    discarded rather than failing the containing operation.
-   Explicitly supplied malformed typed representations are rejected
    with `INVALID_ARGUMENT` and produce no log or Flow mutation.
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
-   BSR module identity is `buf.build/hdev-hm/logging`, with repository
    `proto/` as the module root and canonical paths preserved below it.
-   BSR release labels use the same `v<SemVer>` identity as Git release tags;
    branch-derived labels are not official release identities.
-   `LAST_BSR_COMMIT_ID` stores the immutable BSR commit from the last
    successful BSR publication; `INITIAL` explicitly establishes a new
    baseline, while missing or invalid state blocks release.
-   Contracts v1 has no breaking-change override: a failing compatibility
    check blocks release and an incompatible protocol change requires v2.
-   NuGet and BSR publication are recoverable/idempotent for the same release;
    conflicting source identities across channels block automation.
-   Service runtime expiration policy remains outside Contracts.

This baseline is implemented and publicly distributed as `v1.0.0-preview.1`.
It is now the compatibility baseline for Contracts v1. Future Contracts work is
maintenance or intentional compatible evolution unless an explicit new protocol
version is approved.
