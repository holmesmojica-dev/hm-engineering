# HM Protobuf & gRPC Contract Standard

## 1. Purpose and Scope

This document defines the engineering standards for Protocol Buffers and
gRPC contracts developed within the HM ecosystem.

It applies to reusable protobuf contracts intended for communication between
services, applications, or external consumers, regardless of the business
domain in which those contracts are used.

This standard defines cross-cutting rules for contract design, governance,
versioning, compatibility, validation, dependency management, generation,
and distribution.

Domain semantics, domain models, concrete messages, RPC operations,
application behavior, and domain-specific validation rules MUST be defined
by the corresponding domain and architecture documentation, not by this
standard.

The terms **MUST**, **SHOULD**, and **MAY** express requirement levels:

- **MUST**: mandatory for compliance with the standard.
- **SHOULD**: recommended unless a documented reason justifies otherwise.
- **MAY**: optional.

---

## 2. Source of Truth

HM-owned `.proto` files MUST be the canonical source of truth for protobuf
and gRPC contracts.

Generated source code MUST NOT define or override contract semantics.

Contract definitions MUST remain independent of a particular programming
language, runtime, framework, transport implementation, or service
implementation beyond what is intrinsically part of Protocol Buffers and
gRPC.

Language-specific libraries and generated artifacts represent the contracts;
they do not define them.

Git MUST remain the authoritative source for HM-owned schema source files and
their history.

External registries MAY provide publication, distribution, discovery, and
compatibility capabilities, but they MUST NOT replace the source repository
as the engineering source of truth.

---

## 3. Package and API Versioning

HM protobuf packages MUST use explicit API versioning.

The general naming convention is:

`hm.<domain>[.<component>].vN`

where `N` represents the protobuf API compatibility boundary.

Package identifiers MUST use lowercase naming consistent with protobuf
conventions.

The API version suffix (`v1`, `v2`, etc.) MUST NOT be interpreted as a
package-manager version.

A protobuf API version and the version of any distributed SDK or
language-specific package are related but distinct versioning concepts.

Language-specific generated namespaces MAY follow the conventions of their
target ecosystem.

Generated namespaces MUST NOT alter the semantic identity or versioning
boundary established by the protobuf package.

---

## 4. Compatibility

Once a protobuf package version has been publicly published, backward
compatibility MUST be preserved within that API version.

This requirement applies from the first publicly distributed release,
including preview or prerelease distributions.

Preview or prerelease status MUST NOT be treated as permission to introduce
arbitrary breaking changes into an already published API version.

Compatible additions MAY be introduced within the existing API version.

Existing field numbers MUST NOT be changed or reused.

Existing enum numeric values MUST NOT be changed or reused.

Removed fields and enum values MUST be reserved when necessary to prevent
future reuse.

Changes that cannot be introduced compatibly SHOULD introduce a new protobuf
API version, such as moving from `v1` to `v2`.

Intentional exceptions to compatibility requirements MUST be explicitly
documented, architecturally reviewed, and governed. Implementations MUST NOT
silently introduce incompatible contract changes.

---

## 5. Buf Tooling

Buf is the standard protobuf governance tool for HM contracts.

Projects MUST use the HM-approved Buf configuration baseline and a supported
Buf version compatible with the validation and governance requirements defined
by this standard.

HM-owned schemas MUST pass:

- formatting validation;
- protobuf build validation;
- lint validation;
- breaking-change validation when a compatibility baseline exists.

The default HM lint policy is:

`STANDARD`

The minimum HM breaking-change policy for versioned contracts is:

`FILE`

Projects MAY adopt stricter compatibility policies when justified by their
architecture.

Projects MUST NOT weaken the HM compatibility baseline without a documented
architectural decision.

---

## 6. Buf Schema Registry

Public reusable HM protobuf modules SHOULD be published to the Buf Schema
Registry (BSR).

The HM public BSR organization is:

`buf.build/hdev-hm`

BSR modules SHOULD represent meaningful reusable contract boundaries rather
than individual `.proto` files.

The general module convention is:

`buf.build/hdev-hm/<module>`

Module boundaries SHOULD follow stable technical or domain ownership
boundaries.

Generic shared or common modules SHOULD NOT be introduced preemptively.
They SHOULD be created only when concrete cross-module contracts justify
their existence.

Publication to BSR MUST NOT change Git's role as the authoritative source
repository.

---

## 7. External Protobuf Dependencies

External protobuf schemas SHOULD be consumed from their canonical upstream
source or registry module when available.

When dependencies are resolved through Buf, they MUST be declared explicitly
and their resolved versions MUST be recorded through `buf.lock`.

CI SHOULD verify before publication that committed dependency lock state remains
consistent with the declared module dependencies. A stale or inconsistent lock
state MUST block publication when it would make dependency resolution or the
validated schema graph ambiguous.

HM projects MUST NOT copy, fork, rebrand, or maintain local HM-owned copies
of external schemas solely to make those schemas available to the protobuf
toolchain.

External schemas MUST remain clearly distinguishable from HM-owned schemas.

When official Google API schemas are required, their canonical upstream
protobuf module SHOULD be used when available.

If a build tool requires physical `.proto` files for import resolution,
canonical external dependencies MAY be materialized into temporary or
intermediate build locations.

Such materialization MUST:

- originate from the declared dependency;
- remain reproducible;
- remain outside HM-owned schema source locations;
- NOT redefine ownership of the external schema;
- NOT result in modified external schemas being treated as canonical HM
  contracts.

---

## 8. Generated Code

Generated protobuf and gRPC source code SHOULD NOT be committed to HM source
repositories when deterministic generation can occur as part of the build.

Generated code MUST NOT be manually modified.

Generated code SHOULD be placed in generated, build, or intermediate output
locations appropriate to the target ecosystem.

Generated code SHOULD be excluded from coverage, duplication,
maintainability, and similar source-quality metrics when those metrics would
measure generated implementation rather than HM-authored code.

Projects MAY distribute generated artifacts when required or beneficial for
consumers.

Any exception requiring generated source code to be version-controlled MUST
be explicitly justified by the target ecosystem or distribution mechanism.

---

## 9. Language-Specific Integration

Canonical protobuf schemas MUST remain language-neutral even when one or more
official consumers use a specific programming language.

Language-specific concerns SHOULD be implemented outside the canonical
contract definition unless they are required for wire-contract definition or
generated-code configuration.

These concerns include, among others:

- adapters;
- runtime mappings;
- package-manager integration;
- language-specific validation;
- serialization helpers;
- generated-code packaging.

Language-specific distributions MAY include HM-owned `.proto` schemas
alongside generated or compiled contract artifacts when doing so improves
interoperability, tooling, or consumer experience.

External schemas MUST NOT be redistributed as HM-owned schemas.

Language-specific implementations MUST comply with the applicable HM
technology standards.

They MUST NOT silently expand, restrict, or reinterpret the semantics defined
by the canonical contract and its governing architecture.

Canonical schema comments and protocol documentation MUST describe public
contract semantics in language-neutral terms.

Language-specific documentation MAY provide examples in the primary language
of a distribution or SDK, but those examples MUST be clearly distinguishable
from protocol requirements and MUST NOT redefine the canonical contract.

When schema comments can be propagated into generated documentation or IDE
assistance, projects SHOULD preserve those comments so consumers receive useful
guidance without making generated code the source of truth.

---

## 10. Validation and CI

Changes to HM protobuf contracts MUST be automatically validated before
integration into a protected main branch.

At minimum, protobuf validation MUST verify:

1. formatting;
2. Buf lint;
3. Buf build;
4. breaking compatibility against the appropriate baseline.

Compatibility checks MUST compare against the actual integration or published
baseline applicable to the change.

Once a public contract version exists, release validation SHOULD additionally
verify compatibility against the relevant successfully published contract when
doing so provides additional protection beyond the integration baseline.

When a registry provides immutable publication identities, projects SHOULD
prefer an immutable registry reference over a mutable label for automated
release compatibility validation.

Projects that persist the last successful registry publication as CI/CD state
MUST distinguish an explicit first-baseline/reset state from missing, empty, or
invalid state. Missing state MUST NOT silently disable compatibility validation.

A project MAY establish a stricter policy that forbids compatibility overrides
within an API version. Such a policy SHOULD be documented by the project
architecture.

Failures in mandatory protobuf governance checks MUST block integration unless
an explicitly governed exception applies.

Compilation or generation SHOULD also be validated for officially supported
language targets.

Language-specific quality validation belongs to the applicable HM technology
standard and complements, rather than replaces, protobuf validation.

---

## 11. Distribution and Publication

Every published protobuf contract MUST be traceable to a specific validated
Git commit.

Publication MUST preserve the relationship between:

`source commit -> validation -> release/distribution -> published schema`

Publication automation MUST NOT modify the contract after its validation.

When the same protobuf contracts are distributed through multiple channels,
those distributions MUST originate from the same authoritative contract
source and release state.

The mechanism that triggers publication is NOT defined by this standard.

Release triggers, artifact promotion, deployment strategies, provenance,
attestation, and general release governance MUST follow the applicable HM
delivery standards.

Protobuf API versioning MUST remain conceptually separate from distribution
or package-manager versioning.

For example, an API may remain within:

`hm.<domain>.<component>.v1`

while compatible language-specific distributions evolve through multiple
package releases.

A package-manager major version change does not automatically require a new
protobuf API version.

Likewise, introducing a new protobuf API version does not by itself define
the version number of every language-specific distribution.

---

## 12. Exceptions and Evolution

This standard establishes the HM baseline for protobuf and gRPC contracts.

Project architecture MAY establish stricter requirements when justified by
the characteristics of the project.

A project that intentionally deviates from a **MUST** requirement MUST
document:

- the requirement being deviated from;
- the reason for the deviation;
- the architectural consequences;
- the scope of the exception.

Implementation tooling MUST NOT silently reinterpret, weaken, or bypass this
standard.

Repeated project-specific decisions that demonstrate broader applicability
SHOULD be evaluated for promotion into this standard.

Changes to this standard SHOULD be driven by evidence from real HM projects,
interoperability requirements, and ecosystem evolution rather than
speculative future needs.