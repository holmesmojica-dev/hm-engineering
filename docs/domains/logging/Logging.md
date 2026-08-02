# HM Logging Domain

**Version:** 1.0

**Status:** Draft

**Last Updated:** 2026-08-02

---

# Purpose

The HM Logging domain defines the canonical concepts used to represent logging events across the HM ecosystem.

Its purpose is to provide a technology-agnostic specification shared by every HM Logging implementation, ensuring consistency across libraries, services, providers, and distributed systems.

This specification defines the domain itself, not its implementations.

---

# Concepts

## Log Entry

A **Log Entry** represents a single event generated during the execution of an application.

It is the canonical representation of a logging event within the HM ecosystem. Every implementation must preserve the semantics defined by this specification regardless of the underlying technology.

A Log Entry is composed of the following properties:

| Property | Default Behavior | Description |
|----------|:--------:|-------------|
| Message | Required | Human-readable description of the event. |
| Level | Defaults to Information | Severity level of the event. |
| Timestamp | Automatically initialized (UTC) | UTC date and time when the event occurred. |
| Source | Optional | Logical origin of the event, such as the application, service, component, or module where the event was generated. |
| TraceId | Optional | Technical identifier of the current execution flow used for distributed tracing. |
| CorrelationId | Optional | Identifier used to correlate related operations across requests, services, or business workflows. |
| Exception | Optional | Serialized exception information associated with the event. |
| Metadata | Optional | Collection of structured key-value pairs associated with the event. |

---

## Log Context

A **Log Context** represents information shared across multiple Log Entries within the same execution flow.

Its purpose is to provide default values that are automatically applied to newly created Log Entries.

When the same property exists in both the Log Context and the Log Entry, the value defined by the Log Entry takes precedence.

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

---

# Domain Rules

Every HM Logging implementation must comply with the following rules:

1. Every Log Entry must contain a **Message**.
2. Every Log Entry has a **Log Level**. If no value is explicitly provided, the default level is **Information**.
3. Every Log Entry has a **Timestamp** expressed in UTC.
4. Source, TraceId, CorrelationId, Exception, and Metadata are optional.
5. A Log Context provides default values for Log Entries.
6. Values explicitly defined in a Log Entry override values provided by the Log Context.
7. Metadata from the Log Context and the Log Entry must be merged.
8. When duplicate metadata keys exist, values defined in the Log Entry take precedence.
9. Every implementation must preserve the semantic meaning of the concepts defined in this specification.

---

# References

This specification is implemented by HM Logging projects, including:

- Hm.Logging
- Hm.Logging.gRPC