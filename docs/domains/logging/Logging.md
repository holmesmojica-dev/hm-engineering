# HM Logging Domain

**Version:** 1.1

**Status:** Defined

**Last Updated:** 2026-08-13

---

# Purpose

The HM Logging domain defines the canonical concepts used to represent logging events across the HM ecosystem.

Its purpose is to provide a technology-agnostic specification shared by HM Logging implementations, ensuring consistency across libraries, services, providers, and distributed systems.

This specification defines the domain itself, not the implementation details of any particular library, service, transport, or platform.

The domain must remain independent of specific programming languages and technologies. Implementations are responsible for mapping the domain concepts to their respective technologies while preserving their semantic meaning.

---

# Concepts

## Log Entry

A **Log Entry** represents a single event generated during the execution of an application.

It is the canonical representation of a logging event within the HM ecosystem. Every implementation must preserve the semantics defined by this specification regardless of the underlying technology.

A Log Entry is composed of the following properties:

| Property | Required | Default Behavior | Description |
|----------|:--------:|:---------------:|-------------|
| Message | Yes | None | Human-readable description of the event. |
| Level | Yes | Information | Severity level of the event. |
| Timestamp | Yes | Automatically initialized | Date and time associated with the event. The default value is initialized when the Log Entry is created. |
| Source | No | None | Logical origin of the event, such as the application, service, component, or module where the event was generated. |
| TraceId | No | None | Technical identifier associated with the execution trace. |
| CorrelationId | No | None | Identifier used to correlate related operations across requests, services, or business workflows. |
| Exception | No | None | Serialized exception information represented as a JSON string. |
| Metadata | No | None | Collection of structured key-value pairs associated with the event. |

### Log Entry rules

- `Message`, `Level`, and `Timestamp` are the only non-nullable Log Entry properties.
- `Level` and `Timestamp` have default initialization behavior.
- `Timestamp` may be explicitly supplied or changed by the consumer; implementations must ensure that the resulting Log Entry contains valid timestamp information.
- All other Log Entry properties are optional.

---

## Log Context

A **Log Context** represents information that can be shared across multiple Log Entries within an execution context.

Its purpose is to provide contextual default values that can be applied to Log Entries.

When the same property exists in both the Log Context and the Log Entry, the value explicitly defined by the Log Entry takes precedence.

A Log Context may contain contextual information represented by the domain's Log Entry properties and structured Metadata.

The domain defines the semantics of Log Context. The mechanism used to maintain, activate, nest, or dispose contexts is an implementation concern and is not part of this domain specification.

In particular, the domain does not define remote concepts such as Logging Flows, Flow IDs, `PushScope`, or `PopScope`. Those belong to the distributed service and its communication contracts.

---

## Log Level

A **Log Level** defines the severity of a logging event.

The HM Logging domain defines the following severity levels:

- Trace
- Debug
- Information
- Warning
- Error
- Critical

`Information` has the numeric value `0` and is the default Log Level.

The numeric representation must remain consistent across HM Logging implementations that serialize or transport Log Level values.

---

## Metadata

**Metadata** is a collection of structured key-value pairs associated with a Log Entry or Log Context.

Metadata is part of the domain model, but its concrete representation must remain technology-agnostic.

Supported metadata values must represent the **nature of the data**, rather than being defined around a programming-language-specific type system. Implementations must map their native types to the corresponding domain-level semantic types consistently.

The exact validation and normalization behavior of metadata is an implementation concern.

### Metadata null values

Null metadata values are not retained in a normalized Log Entry or Log Context. They are removed during normalization to preserve data consistency.

### Metadata reserved keys

Metadata keys corresponding to Log Entry properties are reserved by the HM Logging domain and must not be used to introduce alternative representations of those properties.

The reserved keys are:

- Message
- Level
- Timestamp
- Source
- TraceId
- CorrelationId
- Exception
- Metadata

Reserved-key validation is an implementation responsibility, but all HM Logging implementations must preserve the semantic distinction between reserved properties and ordinary metadata.

---

## Exception

The `Exception` property contains serialized exception information as a JSON string.

The JSON representation must contain the information required to represent the exception as textual observability data without requiring the consumer or provider to preserve the original exception type.

The domain does not require the original programming-language exception type to be preserved.

Validation of whether the serialized string contains syntactically valid JSON is an implementation concern and is not a domain validation rule.

This design allows exceptions originating from different platforms, languages, or custom exception types to be represented consistently.

---

# Domain Semantics

## Property precedence

When contextual information and explicit Log Entry information provide the same property:

1. The explicit Log Entry value takes precedence.
2. The Log Context provides the value only when the Log Entry does not provide one.

The same precedence applies when Metadata from a Log Context and Metadata from a Log Entry contain the same key: the Log Entry value takes precedence.

## Context and scopes

The domain recognizes the concept of a Log Context but does not prescribe a single mechanism for maintaining it.

An implementation may provide scopes or another mechanism to associate a Log Context with Log Entries.

A local implementation may support nested scopes, independent scopes, or other valid scope-management semantics as long as the resulting Log Entries preserve the domain semantics.

The specific `BeginScope` API is an implementation-level concept of the Hm.Logging library and is not a requirement imposed by this domain specification.

## Distributed context

The HM Logging domain does not define the mechanism used to transport or maintain context across process or service boundaries.

Concepts such as Logging Flow, Flow ID, remote `PushScope`, and remote `PopScope` belong to the distributed logging service and its contracts. They must not be added to the core Log Entry model solely to support remote context management.

---

# Validation and Normalization Boundaries

The domain defines **what the data means and which properties are required**. It does not define every validation or normalization operation performed by an implementation.

The following responsibilities belong to implementations such as Hm.Logging and, where applicable, Hm.Logging.Service:

- validation of required values;
- null and whitespace handling;
- string normalization;
- metadata normalization;
- removal of null metadata values;
- validation of supported metadata types;
- validation of metadata reserved keys;
- validation of serialized Exception JSON format;
- validation of implementation-specific size or transport constraints.

For string values subject to normalization, a null, empty, or whitespace-only value is normalized to `null` where the corresponding property is optional.

These implementation-level rules must not change the semantic requirements of the domain.

---

# Domain Rules

Every HM Logging implementation must comply with the following rules:

1. Every Log Entry must contain a Message.
2. Every Log Entry must contain a Log Level. If no value is explicitly provided, the default is Information.
3. Every Log Entry must contain a Timestamp. A Timestamp is initialized by default when the Log Entry is created, but may be explicitly supplied or changed.
4. Source, TraceId, CorrelationId, Exception, and Metadata are optional.
5. Exception is represented as a serialized JSON string.
6. A Log Context provides contextual default values for Log Entries.
7. Explicit Log Entry values override values supplied by the Log Context.
8. Metadata from the Log Context and Log Entry is merged according to the property-precedence rule.
9. When duplicate metadata keys exist, the Log Entry value takes precedence.
10. Null metadata values are removed during normalization.
11. Metadata types must represent semantic data types rather than language-specific implementation types.
12. Reserved Metadata Keys correspond to Log Entry properties and are protected by implementations.
13. `Information` is Log Level numeric value `0`.
14. Domain concepts must remain independent of any particular programming language, transport, persistence technology, or distributed-service implementation.
15. Every implementation must preserve the semantic meaning of the concepts defined in this specification.

---

# Implementation Boundary

The HM Logging domain is the foundation shared by the HM Logging ecosystem.

The core `Hm.Logging` library may implement the domain locally, including Log Context and local scope management.

A distributed logging service may build additional protocol-level capabilities on top of the domain, including remote scope management and Logging Flows.

Those distributed capabilities must consume and preserve the domain model rather than modify the domain to accommodate transport-specific requirements.

---

# References

This specification is implemented by HM Logging projects, including:

- Hm.Logging
- Hm.Logging.Contracts
- Hm.Logging.Service

The specific APIs, protocols, providers, and persistence mechanisms used by those projects are implementation concerns and are not defined by this document.
