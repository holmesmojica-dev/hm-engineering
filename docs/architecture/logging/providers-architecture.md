# HM Logging Providers Architecture

**Component family:** `Hm.Logging.Providers.*`  
**Repository / solution:** `Hm.Logging.Providers`  
**Status:** Base architecture defined; Console v1 and Files v1 ready for implementation; ElasticSearch and EntityFramework detailed architecture pending  
**Date:** 24 September 2026

---

## 1. Purpose

This document defines the architecture of the official provider family for HM Logging.

Providers are destination adapters for the Core `Hm.Logging` pipeline. They receive a `LogEntry` that Core has already validated, normalized, enriched, and merged with applicable context, and they adapt that entry only as required by the destination they own.

This document is the authoritative architecture specification for the `Hm.Logging.Providers` solution and for provider-specific decisions that have been approved. It is intentionally evolutionary: provider sections are completed when that provider reaches architectural readiness. A provider whose detailed design is still pending must not be implemented by inventing missing architecture in code.

### 1.1 Source-of-truth hierarchy

Provider implementations must interpret HM engineering documentation in the following order:

1. `docs/domains/logging/Logging.md` — canonical HM Logging domain semantics.
2. Applicable HM engineering standards — quality, delivery, repository governance, and other cross-cutting rules.
3. `docs/architecture/logging/providers-architecture.md` — provider-family and provider-specific architecture defined by this document.
4. Component implementation documentation for details that do not alter the established architecture.

`Hm.Logging` Core remains authoritative for its public provider abstraction and runtime pipeline behavior. A provider must not duplicate or reinterpret Core responsibilities.

---

## 2. Architectural Role and Boundaries

The HM Logging ecosystem separates the following roles:

- `Hm.Logging` — Core library. Owns local logging behavior, normalization, validation, context merging, provider orchestration, provider-failure isolation, and Core observability behavior.
- `Hm.Logging.Providers.*` — destination-specific implementations of Core provider abstractions.
- `Hm.Logging.Contracts` — language-neutral protobuf/gRPC contracts for distributed logging. Independent from Core and Providers.
- `Hm.Logging.Service` — future deployable distributed logging integrator. It may use Core and one or more Providers but does not redefine provider semantics.

Conceptually:

```text
Embedded application
    -> Hm.Logging
        -> Hm.Logging.Providers.*

Distributed consumer
    -> Hm.Logging.Contracts
        -> Hm.Logging.Service
            -> Hm.Logging
                -> Hm.Logging.Providers.*
```

A provider can therefore be used directly by an application using Core. `Hm.Logging.Service` is not required in order to use a provider.

---

## 3. Solution and Package Architecture

### 3.1 Repository and solution

The official providers live in one repository and one solution named:

```text
Hm.Logging.Providers
```

The initial provider projects are:

```text
Hm.Logging.Providers.Console
Hm.Logging.Providers.Files
Hm.Logging.Providers.ElasticSearch
Hm.Logging.Providers.EntityFramework
```

The initial implementation order is:

1. Console
2. Files
3. ElasticSearch
4. EntityFramework

Console and Files have completed their v1 architectural design and may enter implementation. ElasticSearch and EntityFramework require dedicated architecture work before implementation.

### 3.2 Independent packages

The solution itself is not an aggregate NuGet package. Each provider is an independent package so consumers install only the destinations and dependencies they need.

Approved package IDs are:

```text
HDev.Hm.Logging.Providers.Console
HDev.Hm.Logging.Providers.Files
HDev.Hm.Logging.Providers.ElasticSearch
HDev.Hm.Logging.Providers.EntityFramework
```

Each provider references `HDev.Hm.Logging.Core` and implements the provider abstraction exposed by Core. Core must never reference provider packages.

Providers must not depend on each other unless a future explicit architecture decision establishes a concrete need. No such dependency exists in the current baseline.

### 3.3 Versioning and release independence

Provider packages share a repository and source history but have independent release identities and versions. Different providers may therefore have different package versions.

A single source commit may be the source of releases for more than one provider. Each provider release remains an independent Delivery execution and must publish only the package identified by that release intent.

The exact provider release-tag grammar and generic Delivery tag-parsing convention are intentionally not established by this architecture baseline. They must be decided before the first provider publication and aligned with the HM Release & Deployment Standard rather than invented ad hoc inside an individual provider workflow.

Solution-level Quality may validate the complete solution. Provider-specific Delivery must not publish unrelated provider packages merely because they share the same repository.

---

## 4. Common Provider Contract

### 4.1 Core boundary

Core exposes `ILogProvider` as the provider extension boundary. A provider implements the asynchronous write operation defined by that abstraction and receives the `LogEntry` produced by Core.

Before provider invocation, Core owns the cross-cutting logging pipeline, including applicable validation, normalization, enrichment, context/metadata merging, and provider orchestration.

A provider must therefore treat the received `LogEntry` as the normalized Core representation. It must not reimplement Core domain validation or reinterpret HM Logging semantics.

### 4.2 Destination adaptation

A provider owns only the adaptation required by its destination. Depending on the destination, this may include:

- serialization;
- output formatting;
- persistence mapping;
- destination-specific document or entity mapping;
- destination-specific I/O.

Mapping must preserve the semantic meaning of the Core `LogEntry`.

Console and Files use the normalized `LogEntry` directly as their source representation. They do not introduce a separate persistence/domain model merely for abstraction symmetry.

ElasticSearch and EntityFramework are expected to require explicit destination models or mappings, but their detailed design remains pending.

### 4.3 Failure boundary

Providers do not own generic retry, cross-cutting recovery, provider-failure observability, or failure isolation.

A provider must not swallow a destination failure merely to keep the logging call alive. Destination failures leave the provider operation and are handled by Core according to Core's provider-isolation and `ProviderFailureCallback` behavior.

Providers must not invoke the Core failure callback themselves.

Destination-specific retry behavior may only be introduced later when it is an explicit property of a particular destination architecture. No generic provider retry policy exists in the current baseline.

### 4.4 Cancellation

Providers must respect the `CancellationToken` supplied by Core while waiting for or performing cancellable destination work. Core remains responsible for the overall cancellation semantics of the logging operation.

### 4.5 Dependency injection

The provider family does not introduce an `HmLoggingBuilder`. Provider packages extend `IServiceCollection` directly and remain chainable with Core registration.

The naming convention is explicit and logging-specific:

```csharp
builder.Services
    .AddHmLogging(options => { /* Core options */ })
    .AddLoggingConsole(options => { /* Console options */ })
    .AddLoggingFiles(options => { /* Files options */ });
```

Future providers follow the same naming pattern:

```text
AddLoggingElasticSearch
AddLoggingEntityFramework
```

Each registration method owns registration of its provider implementation, options, and provider-internal dependencies. Consumers should not need to manually register the corresponding `ILogProvider` implementation.

### 4.6 Provider lifetime

Lifetime is selected according to provider behavior and dependencies; Singleton is not a universal provider rule.

The approved lifetimes currently are:

- Console — Singleton.
- Files — Singleton.
- ElasticSearch — pending detailed architecture.
- EntityFramework — pending detailed architecture; its persistence dependencies may affect lifetime selection.

### 4.7 Configuration lifetime

Provider options describe the configuration of a provider instance. The current provider baseline does not support hot reload.

Options are resolved for the provider lifecycle and remain stable for that instance. Configuration may originate from code, `appsettings.json`, environment variables, or another application configuration source, but changing the source does not mutate an already-created provider instance. Applying new provider configuration requires a new provider/application lifecycle.

---

## 5. Console Provider

### 5.1 Role

`Hm.Logging.Providers.Console` writes normalized HM Logging entries to process console output. It is intended for local development, diagnostics, command-line applications, container output, and other environments where stdout/stderr are the destination.

Registration is exposed through:

```csharp
AddLoggingConsole(...)
```

The provider is registered as Singleton.

### 5.2 Output format

Console supports two output formats:

- Text — default; human-readable console presentation.
- Json — structured JSON representation.

JSON must preserve semantic values. In particular, `HmLogLevel` must be serialized using its semantic name such as `"Information"` or `"Error"`, not its numeric enum value.

Visual Text settings do not alter JSON output.

### 5.3 Timestamp presentation

Text output supports a predefined timestamp-format option rather than requiring consumers to supply arbitrary .NET format strings. The provider must include an ISO 8601 option and may include a human-readable option approved during implementation.

JSON uses a standardized ISO representation and does not depend on the Text timestamp presentation option.

The exact names of the predefined timestamp enum values are an implementation-level API detail to finalize consistently during implementation; introducing arbitrary custom format-string semantics is outside the current architecture.

### 5.4 Colors

Console options include:

```csharp
public bool UseColors { get; set; } = true;
```

Colors apply only to Text output. `UseColors = false` produces Text without color control sequences. JSON never contains terminal color codes and ignores this option.

When enabled, colors are selected from a fixed HM palette according to Log Level. The palette is not consumer-configurable in v1. Exact terminal color values are an implementation detail.

### 5.5 stdout and stderr

Console options include `UseStandardErrorForErrors`, defaulting to `true`.

When enabled:

- Error and Critical are written to stderr.
- Trace, Debug, Information, and Warning are written to stdout.

When disabled, all levels are written to stdout.

### 5.6 Exception presentation

Text output supports two exception presentations:

- Compact — one-line-oriented presentation.
- Multiline — developer-readable multiline presentation; default.

`LogEntry.Exception` is textual observability data. Console does not assume that it can reconstruct or deserialize the originating runtime exception object.

JSON preserves exception information as textual data and ignores Text-only visual formatting.

### 5.7 Implementation-detail boundary

The following are intentionally not architectural contracts unless later evidence requires promotion to that level:

- exact Text field order;
- punctuation and separators;
- indentation and whitespace;
- exact ANSI/color values;
- exact JSON property ordering;
- internal helper classes or serialization implementation.

Implementation must preserve the behavioral decisions above without turning incidental formatting choices into ecosystem semantics.

---

## 6. Files Provider

### 6.1 Role and lifecycle

`Hm.Logging.Providers.Files` persists normalized HM Logging entries to files managed by the provider.

Registration is exposed through:

```csharp
AddLoggingFiles(...)
```

The provider is Singleton because its instance owns coordinated runtime state including file-level exclusion, daily maintenance state, and optional managed-storage accounting.

The provider does not run a background worker, timer, or `IHostedService` in v1. Maintenance is demand-driven by writes.

### 6.2 FilesProviderOptions

The approved conceptual configuration is:

```csharp
public sealed class FilesProviderOptions
{
    public string DirectoryPath { get; set; } = "logs";

    public bool GroupBySource { get; set; } = false;

    public uint? RetentionDays { get; set; } = 30;

    public FileSize? MaximumFileSize { get; set; } = FileSize.FromMB(100);

    public FileSize? MaximumTotalSize { get; set; }

    public FilesLogFormat Format { get; set; } = FilesLogFormat.Json;
}
```

Exact implementation syntax may evolve without changing these semantics.

### 6.3 DirectoryPath

The default managed root is the relative path:

```text
logs
```

A null, empty, or whitespace-only configured path normalizes to the default `logs` path rather than becoming a provider configuration failure.

The path may otherwise be any relative or absolute filesystem path that the host operating system can resolve, including an application-mounted or network-backed filesystem. Files does not implement remote storage protocols itself.

The required directory is created lazily on the first write when absent. Filesystem failures such as permission or path-access failures propagate naturally through the provider boundary.

The configured root directory is never automatically deleted by Files.

### 6.4 FileSize value type

`FileSize` belongs to `Hm.Logging.Providers.Files`, not Core. It is a small immutable value type with canonical internal representation in bytes:

```csharp
ulong Bytes
```

The public construction model provides factories conceptually equivalent to:

```csharp
FileSize.FromBytes(ulong bytes)
FileSize.FromKB(decimal kilobytes)
FileSize.FromMB(decimal megabytes)
FileSize.FromGB(decimal gigabytes)
```

All values must be strictly greater than zero. Zero and negative values are invalid and the public contract/XML documentation must state that invalid factory input throws `ArgumentOutOfRangeException` as applicable. Conversion overflow must also fail rather than silently wrap.

Fractional decimal values are valid when they produce a positive byte size, for example `FileSize.FromMB(1.5m)`.

Units are decimal SI units:

```text
1 KB = 1,000 bytes
1 MB = 1,000,000 bytes
1 GB = 1,000,000,000 bytes
```

Binary units are not implied by KB/MB/GB. If binary units are introduced in the future they must use explicit KiB/MiB/GiB semantics.

### 6.5 Formats

Files supports:

- Json — default.
- Text.

Both formats use UTF-8 without BOM.

Json is JSON Lines (JSONL): each `LogEntry` is serialized as one independent JSON object, with no enclosing array. The file extension is `.jsonl`.

JSON must serialize `HmLogLevel` using the semantic level name rather than a numeric enum value.

Text uses `.log` and provides a human-readable representation conceptually consistent with Console Text output, without terminal-specific color behavior. It must preserve relevant `LogEntry` information, but exact field order, indentation, line breaks, metadata layout, and exception visual layout remain implementation details.

### 6.6 UTC day model and naming

UTC is the authoritative clock for:

- logical day boundaries;
- daily file naming;
- daily rotation;
- retention eligibility;
- daily maintenance state.

Files does not use filesystem creation or last-write timestamps as the source of truth for retention. The logical date encoded in a recognized Files-managed filename is authoritative.

Without source grouping, the logical daily stream is represented conceptually as:

```text
logs/
└── logs-2026-09-24.jsonl
```

or the equivalent `.log` file for Text.

### 6.7 GroupBySource

`GroupBySource` defaults to `false`.

When `false`, `LogEntry.Source` does not affect physical storage. Entries from all Sources share the applicable daily stream, while Source remains present in the log data itself.

When `true`, Files physically partitions logs by normalized Source:

```text
logs/
├── payments/
│   └── payments-2026-09-24.jsonl
├── catalog/
│   └── catalog-2026-09-24.jsonl
└── unknown/
    └── unknown-2026-09-24.jsonl
```

Source normalization must be deterministic and safe for filesystem use. It must prevent path traversal and neutralize path separators and invalid path characters. The approved conceptual behavior includes lowercase normalization and conversion of spaces to `-`; the exact sanitization algorithm is an implementation detail provided that the same Source deterministically maps to the same safe container.

A missing, empty, or whitespace-only Source maps to the fixed container name `unknown`. This fallback is not configurable in v1.

A Source directory is created lazily when needed. When Files-managed cleanup leaves a Source directory empty, that empty Source directory is removed. It is recreated if the Source later produces logs. The root `DirectoryPath` remains.

The filename prefix follows the physical stream/container identity so a copied file remains self-descriptive.

### 6.8 Daily rotation and segmentation

Each UTC day is one logical log stream per physical grouping. `MaximumFileSize` may divide that logical day into numbered physical segments.

The naming convention is conceptually:

```text
payments-2026-09-24.jsonl
payments-2026-09-24.1.jsonl
payments-2026-09-24.2.jsonl
```

The same convention applies to `.log` Text files.

A `LogEntry` is never split across files.

`MaximumFileSize` defaults to 100 MB. When it is `null`, no per-file size limit or size-based segmentation is applied.

When configured, `MaximumFileSize` is a strict maximum:

- if the serialized entry itself exceeds the configured maximum, the write is rejected;
- if appending the complete entry would exceed the current segment maximum, the entry is written to the next segment;
- no percentage, tolerance, or oversize margin is permitted.

A consumer that legitimately needs larger individual entries must raise the limit or set it to `null`.

### 6.9 RetentionDays

`RetentionDays` defaults to 30. `null` disables age-based cleanup.

Retention operates on complete logical UTC days. All recognized segments belonging to an expired day are removed together; retention does not arbitrarily remove individual segments from an otherwise retained day.

Only files recognized as Files-managed according to the provider naming convention are eligible for automatic deletion. Unknown or unparseable files are not deleted.

After cleanup, empty Source directories are removed when source grouping is enabled. The configured root remains.

### 6.10 MaximumTotalSize

`MaximumTotalSize` defaults to `null`. When `null`, Files does not perform global managed-storage capacity tracking merely for this feature.

When configured, the limit applies only to Files-managed files under the configured provider storage, not arbitrary foreign files that happen to exist in the same directory tree.

The Singleton maintains an internal `CurrentTotalSize` after reconciliation so ordinary writes do not require a complete filesystem scan. Each successful Files-managed write increments that state and each Files-managed deletion decrements it.

Before a write, Files evaluates global capacity using the complete serialized entry size. The global `MaximumTotalSize` check occurs before per-file `MaximumFileSize` segment selection.

If the new entry does not fit within `MaximumTotalSize`, Files attempts to free capacity by deleting eligible old UTC days, oldest first. The current UTC day must never be deleted merely to make room for a new write.

If sufficient capacity still cannot be obtained after all eligible old days have been removed, the new write is rejected and the failure propagates to Core.

`MaximumTotalSize` and `MaximumFileSize` are independent constraints. Architecture does not require one to be numerically less than or equal to the other; the applicable constraint may reject a particular write first.

### 6.11 Reconciliation and demand-driven maintenance

Files stores the last successfully reconciled UTC day as instance state, conceptually:

```csharp
DateOnly? _lastMaintenanceDateUtc;
```

A boolean is insufficient because the provider must distinguish maintenance performed for the current UTC day from maintenance performed for a previous day.

On each write, the provider compares the current UTC date with the stored maintenance date. If they match, normal write processing continues. If the stored value is `null` or represents a different UTC day, one caller performs maintenance while concurrent callers are coordinated so the same daily maintenance is not performed concurrently.

The first write after provider/application startup therefore follows the same maintenance path as the first write of any later UTC day. No separate eager startup-initialization lifecycle is required.

Daily maintenance must reconstruct/reconcile the relevant state from the actual Files-managed storage. Conceptually it includes:

1. resolve/create the managed directory when needed;
2. discover recognized Files-managed files;
3. apply `RetentionDays` when enabled;
4. remove empty Source directories created by Files when applicable;
5. reconcile `CurrentTotalSize` when `MaximumTotalSize` is enabled;
6. record the current UTC date as the last successfully maintained day;
7. continue with the triggering write.

The maintenance date is updated only after successful maintenance. A failed maintenance attempt must not mark the day as reconciled.

No auxiliary persistent state file is required for `CurrentTotalSize` or the maintenance date. A new provider instance reconstructs operational state from managed files.

External/manual modification of Files-managed logs while the application is running can temporarily desynchronize the in-memory total-size state. Documentation should therefore treat the Files directory as provider-managed storage and discourage external mutation during execution. The next reconciliation repairs state from the filesystem.

### 6.12 Concurrency and integrity

Files must guarantee asynchronous exclusion per physical file within a single provider instance. Concurrent writes targeting the same physical file are serialized so the bytes of two entries cannot interleave. Writes targeting different files may proceed concurrently when global capacity coordination does not require broader serialization.

The complete entry is serialized/formatted before exclusive append so a JSONL entry or Text entry is treated as one write unit for integrity and size decisions.

When `MaximumTotalSize` is enabled, capacity check/reservation, deletion when necessary, write accounting, and concurrent writes must be coordinated so two writers cannot independently pass the same capacity check and violate the configured maximum.

The exact synchronization primitives, lock registry, reservation strategy, and internal classes are implementation details.

The concurrency guarantee is process/provider-instance local. Files does not provide a distributed filesystem lock for multiple application processes or replicas writing the same physical file. Deployments requiring multiple writers should use separate physical paths/files per instance or a centralized provider designed for that topology.

Files v1 does not introduce a background queue, batching pipeline, generic retries, or background writer.

### 6.13 Write completion and durability

A successful Files write means the complete entry has been written through the provider's stream/I/O operation before `WriteAsync` returns.

Files does not force a physical `fsync`/equivalent durable-storage flush for every log entry. It therefore does not promise that an entry acknowledged immediately before abrupt power or storage failure has reached permanent media.

This boundary preserves entry integrity and practical logging performance without claiming transactional durability that the provider does not implement.

### 6.14 Failure behavior

Files prefers meaningful platform/runtime exceptions over an unnecessary custom exception hierarchy.

Filesystem failures such as directory creation, file opening, permissions, or write failures should propagate their native exception when that exception already communicates the failure adequately.

Provider-contract failures such as an entry exceeding `MaximumFileSize` or inability to admit an entry under `MaximumTotalSize` should use an appropriate standard exception with an explicit diagnostic message unless implementation work establishes a concrete programmatic need for a dedicated public exception type.

Files must not swallow failures, perform generic retries, or call Core observability callbacks. Failures leave the provider boundary and Core owns provider-failure isolation and observability.

### 6.15 Configuration stability

`FilesProviderOptions` is stable for the lifetime of the Singleton provider. Files v1 does not hot-reload `DirectoryPath`, `GroupBySource`, `RetentionDays`, `MaximumFileSize`, `MaximumTotalSize`, or `Format`.

A new configuration takes effect through a new provider/application lifecycle.

---

## 7. ElasticSearch Provider — Current Boundary

The approved project and package identities are:

```text
Hm.Logging.Providers.ElasticSearch
HDev.Hm.Logging.Providers.ElasticSearch
```

Its role is to send normalized HM Logging entries to Elasticsearch as a centralized searchable/indexed destination. It may be used directly by an application or later by `Hm.Logging.Service`; Service is not a prerequisite.

The provider does not provision or operate Elasticsearch infrastructure.

An explicit destination-document mapping is expected, but index strategy, configuration, lifecycle, batching, client dependencies, retry behavior, failure semantics beyond the common provider boundary, and other destination-specific decisions remain open. They require a dedicated architecture session before implementation.

---

## 8. EntityFramework Provider — Current Boundary

The approved project and package identities are:

```text
Hm.Logging.Providers.EntityFramework
HDev.Hm.Logging.Providers.EntityFramework
```

Its role is persistence through Entity Framework Core while allowing the consuming application to choose the EF Core database provider/engine.

Database schema migration and database lifecycle are outside the provider's responsibility. The consuming application or a future `Hm.Logging.Service` deployment owns migration execution and schema lifecycle.

An explicit persistence mapping is expected. Dynamic normalized metadata will require a persistence representation suitable for the selected database model, but the exact schema, metadata storage representation, `DbContext` integration, provider lifetime, transaction behavior, and related persistence decisions remain open. They require a dedicated architecture session before implementation.

---

## 9. Quality and Delivery Application

The provider solution and projects are governed by the applicable HM engineering standards, including:

- `docs/standards/dotnet/quality.md`;
- `docs/standards/git-github/repository-governance.md`;
- `docs/standards/delivery/release-and-deployment.md`.

Quality should validate the complete solution where applicable so integration problems between projects, shared build configuration, and repository-wide quality rules are detected before Main eligibility.

Each public provider NuGet package must independently satisfy the HM NuGet Delivery Profile, including the mandatory package icon and applicable package validation, symbols, source mapping, provenance, and remote publication verification requirements.

Provider-specific Delivery must publish only the provider represented by the release intent. The exact multi-package tag convention remains an explicit pre-publication design item.

---

## 10. Architectural Invariants

The current Providers baseline establishes these invariants:

1. Providers depend on Core; Core does not depend on Providers.
2. Contracts remains independent from Providers.
3. A provider receives Core-normalized `LogEntry` data and does not redefine Core domain semantics.
4. Destination-specific serialization/mapping belongs to the provider.
5. Generic provider failure isolation and observability belong to Core.
6. Generic retries are not a provider-family behavior.
7. Provider registration uses explicit `IServiceCollection` extensions named `AddLoggingXxx`; no `HmLoggingBuilder` is introduced.
8. Provider lifetime is selected per destination; Singleton is not universal.
9. Provider packages are independently consumable and independently versioned.
10. Console v1 and Files v1 are architecturally ready for implementation.
11. ElasticSearch and EntityFramework must complete detailed architecture before implementation.
12. Implementation details must not silently become new ecosystem semantics.

---

## 11. Current Implementation Plan

The approved progression is incremental rather than waiting for every provider to be fully designed:

```text
Providers base architecture
    -> Console architecture
    -> Console implementation / validation / publication
    -> Files architecture
    -> Files implementation / validation / publication
    -> ElasticSearch dedicated architecture
    -> architecture document update
    -> ElasticSearch implementation / validation / publication
    -> EntityFramework dedicated architecture
    -> architecture document update
    -> EntityFramework implementation / validation / publication
```

Console and Files architecture is already complete in this baseline, so implementation may begin after this documentation state is committed.

The architecture document must evolve when later provider decisions are approved. It should record meaningful architectural milestones and not incidental implementation changes.

---

## 12. Open Design Items

The following items are deliberately open and must not be inferred from implementation convenience:

- exact provider release-tag grammar and generic Delivery parsing for independent packages in the shared repository;
- detailed ElasticSearch architecture;
- detailed EntityFramework architecture;
- any future dependency policy that would intentionally permit one provider to depend on another;
- any future provider-specific retry behavior;
- any future hot-reload requirement.

These open items do not block implementation of Console or Files.

---

## 13. Current Architectural Baseline

As of 24 September 2026, the `Hm.Logging.Providers` solution architecture, common provider responsibilities, Console v1 architecture, and Files v1 architecture are defined and ready for implementation.

ElasticSearch and EntityFramework remain approved members of the initial provider catalog but require separate detailed architecture work before implementation. This document is the canonical place to incorporate those decisions as they are established.
