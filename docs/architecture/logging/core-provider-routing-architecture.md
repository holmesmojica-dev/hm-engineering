# HM Logging Core — Provider Identity and Selective Routing Architecture

## 1. Purpose and authority

This architecture defines approved generic Core provider identity, selective dispatch, and failure-observability behavior. It extends the existing local logging pipeline without changing the technology-independent HM Logging domain, distributed Logging Flow semantics, or provider-specific destination policies. **Status: approved design; implementation and publication pending (9 October 2026).**

## 2. Responsibilities and independence

- Core orchestrates `ILogProvider` instances through abstractions; it must not reference concrete Console, Files, ElasticSearch, EntityFramework, or consumer-specific provider implementations.
- Providers own destination-specific writes and their own `IServiceCollection` registration extensions (`AddLoggingXxx`); Core does not register or construct concrete providers.
- The existing Core pipeline remains responsible for filtering, scope/context merging, validation, normalization, cancellation, dispatch, isolation, and failure diagnostics.
- Selective routing is a general Core capability, not an ElasticSearch integration feature.

## 3. Provider identity

- Each registered `ILogProvider` instance exposes `Guid ProviderId { get; }` as part of the Core interface.
- `ProviderId` identifies a **registered instance**, not a provider implementation type. Distinct instances of the same implementation must be independently addressable.
- `Guid.Empty` is invalid. Two registered instances must not share the same identifier; duplicate registration identity is a configuration error.
- Provider registration extensions permit an optional caller-supplied `Guid` identifier. When omitted, the registration mechanism assigns a nonempty generated GUID to that instance. The identifier remains stable for the lifetime of the registered instance.
- Registration extensions retain ownership of their provider implementations, options, and dependencies. No Core-owned registration API for concrete providers or new builder abstraction is introduced.
- Consumers who require stable/selectable identities may retain their supplied GUIDs in named variables. Automatically generated identities do not imply stability across application lifecycles.

## 4. Dispatch API and selection semantics

- Preserve the existing `ILoggerService.LogAsync(LogEntry, ProviderFailureCallback?, CancellationToken)` overload and its all-providers behavior.
- Add an overload accepting `IEnumerable<Guid>? targetProviders` in addition to the existing failure callback and cancellation capabilities. Exact C# signature/parameter ordering must be chosen to avoid overload ambiguity and preserve existing calls.
- `targetProviders == null` or an empty enumeration means dispatch to **all** registered providers.
- Otherwise, de-duplicate the requested identifiers before dispatch. A repeated target identifier must not cause multiple writes to the same instance.
- All requested identifiers must resolve to registered providers. Unknown identifiers cause an **exception**, not `ProviderFailureCallback` invocation.
- Resolve and validate the entire selection **before invoking any provider**, so an unknown identifier cannot cause a partial dispatch.
- The same Core normalization, filtering, cancellation, and failure isolation apply to selective and default dispatch. The selection only changes which providers are invoked.
- Core must not dispatch twice to a provider because a requested identifier was repeated.

## 5. Failure observability

- `ProviderFailureContext` identifies the failing registered instance using `Guid ProviderId`, alongside the failure `Exception` and normalized `LogEntry`.
- Remove `ProviderType` from this context in the new Core contract. This is an intentional source/API compatibility break requiring coordinated consumer updates and an appropriate new prerelease.
- `ProviderFailureCallback` remains a Core-level mechanism for exceptions arising from the provider write invocation/admission. It is **not** used for unknown target identifiers or configuration errors.
- Cancellation semantics remain unchanged: cancellation-related `OperationCanceledException` is propagated rather than reported as an ordinary provider failure.
- Destination-specific deferred persistence failure events (including any future ElasticSearch observer) are **not** defined here; their contract will be designed with that provider. Do not assume `ProviderFailureContext` is suitable for deferred persistence failures.

## 6. Compatibility, rollout and acceptance

- This architecture is approved but not yet implemented or released. Existing published Core and Providers packages retain their actual public contracts until updated.
- Implement Core and test identity uniqueness, empty/unknown selections, duplicate requests, multiple instances of one implementation, unchanged default routing, failure callback identity, normalization, and cancellation.
- Update affected provider implementations and registration extensions (including Console and Files), and consumers such as Ancestral API, after the Core contract change is available. Coordinate package versions; do not imply updates have already occurred.
- Preserve independent provider package ownership and release/versioning; do not alter Contracts or distributed Logging Flow semantics.

## 7. Open implementation details

- Exact overload signature and DI implementation mechanism, including how automatically assigned identifiers are exposed when needed, are implementation details to validate against current Core/Providers source without changing the approved semantics.
- Exact exception types for invalid configuration or unknown targets are to be selected during Core implementation; the mandated behavior is to throw, not to invoke the failure callback.
