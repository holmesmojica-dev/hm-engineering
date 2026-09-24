# HM .NET Quality Standard

## 1. Purpose and Scope

This document defines the baseline engineering quality requirements for
.NET projects developed and maintained within the HM ecosystem.

It applies to .NET libraries, APIs, services, workers, applications, and
other .NET-based software unless a more specific approved standard
establishes stricter requirements.

This standard defines cross-cutting quality expectations. It does not
define application architecture, domain design, repository structure,
release strategy, deployment strategy, or package publication.

The terms **MUST**, **SHOULD**, and **MAY** express requirement levels:

-   **MUST**: mandatory for compliance with the standard.
-   **SHOULD**: recommended unless a documented reason justifies
    otherwise.
-   **MAY**: optional.

------------------------------------------------------------------------

## 2. .NET Platform Baseline

HM .NET projects MUST target a supported .NET version appropriate to the
project's lifecycle, requirements, and ecosystem constraints.

Projects SHOULD prefer supported platform versions that provide an
appropriate balance of stability, security, compatibility, and
maintainability.

The specific .NET and C# versions used by a project MUST be defined by
that project's architecture or build configuration, not by this
standard.

Framework and language upgrades MUST be evaluated deliberately,
considering compatibility, dependencies, operational impact, and
maintenance requirements.

Projects using versions outside their normal support lifecycle MUST
document the technical reason and relevant maintenance or security
implications.

------------------------------------------------------------------------

## 3. Build Quality

A clean build of a new HM .NET project MUST complete without errors or
compiler warnings.

Build failures MUST block integration.

Projects SHOULD maintain a zero-warning baseline.

Existing projects that contain accepted legacy warnings MAY establish a
documented baseline, but new warnings MUST NOT be introduced without
explicit justification.

Compiler diagnostics MUST NOT be broadly disabled merely to obtain a
successful build.

Suppressions MAY be used when technically justified and SHOULD be scoped
as narrowly as practical.

------------------------------------------------------------------------

## 4. Formatting

HM-authored C# code MUST follow deterministic formatting rules.

`.editorconfig` SHOULD be used as the repository-level source for
applicable formatting and coding-style configuration.

`dotnet format` MUST succeed for code subject to automated formatting
validation.

Formatting validation SHOULD occur early enough in the development
workflow to provide fast feedback before expensive quality checks
execute.

Generated code SHOULD NOT be reformatted unless the generation mechanism
explicitly requires it.

------------------------------------------------------------------------

## 5. Static Analysis

HM .NET projects MUST use compiler diagnostics and Roslyn-based static
analysis as part of their quality validation.

Static analysis MUST execute automatically before code is integrated
into the protected main branch.

Projects MAY use additional analyzers when their architecture or risk
profile justifies them.

This standard does not mandate a specific third-party analyzer package
or analyzer version.

Analyzer selection SHOULD avoid redundant tooling that produces
substantially equivalent diagnostics without meaningful additional
value.

Static-analysis rules MUST NOT be globally disabled merely to make
validation pass.

Any intentional suppression of a relevant diagnostic SHOULD be
technically justified and scoped to the smallest practical surface.

------------------------------------------------------------------------

## 6. Automated Testing

HM-authored behavior SHOULD be covered by automated tests appropriate to
the type and risk of the project.

Projects containing business logic, reusable behavior, transformations,
validation, integration boundaries, or other testable behavior MUST
maintain automated tests for that behavior.

Tests MUST be sufficiently deterministic to provide reliable CI
feedback.

Tests MUST NOT depend unnecessarily on developer-specific machine state.

New HM .NET projects SHOULD use the HM-approved .NET testing
infrastructure appropriate to their requirements.

Microsoft Testing Platform SHOULD be preferred when it is compatible
with the selected test framework and project requirements.

This standard does not prescribe a specific version of the testing
platform, test framework, or coverage tooling.

A project MAY use another supported testing infrastructure when
technically justified.

The choice of unit, integration, contract, end-to-end, or other test
types MUST be driven by the behavior and risk being validated rather
than by a requirement to maximize test count.

------------------------------------------------------------------------

## 7. Code Coverage

HM .NET projects with automated tests SHOULD measure code coverage.

Coverage MUST be interpreted as a quality signal rather than as proof of
correctness.

Coverage reporting SHOULD focus on HM-authored code.

Generated code SHOULD be excluded when including it would distort the
quality signal.

Infrastructure or mechanically generated artifacts MAY also be excluded
when their inclusion provides little meaningful information, provided
exclusions are justified and reproducible.

This standard does not define a universal minimum coverage percentage.

Projects MAY establish stricter coverage requirements based on their
architecture, maturity, or risk profile.

Coverage requirements SHOULD favor meaningful coverage of new or changed
code over artificial increases intended only to satisfy a numeric
threshold.

------------------------------------------------------------------------

## 8. SonarQube Quality Analysis

Actively maintained HM .NET projects SHOULD integrate with the
HM-approved SonarQube quality process.

Projects intended for public distribution, production use, or reuse
across the HM ecosystem MUST use an automated Quality Gate unless an
explicit technical constraint prevents it.

The Quality Gate SHOULD evaluate HM-authored code and SHOULD avoid
allowing generated code to distort maintainability, duplication, or
coverage metrics.

Quality analysis SHOULD include applicable concerns such as:

-   reliability;
-   maintainability;
-   security findings;
-   code duplication;
-   test coverage.

A failing mandatory Quality Gate MUST block integration into the
protected main branch.

Project-specific SonarQube configuration MAY be stricter than this
standard.

This standard does not prescribe a specific SonarQube edition, hosting
model, or version.

------------------------------------------------------------------------

## 9. Generated Code

Generated source MUST NOT be treated as equivalent to HM-authored source
for quality metrics when doing so would produce misleading results.

Generated code SHOULD be excluded from:

-   code coverage requirements;
-   duplication analysis;
-   maintainability measurements;
-   formatting enforcement;
-   other source-quality metrics that primarily evaluate human-authored
    code.

Generated code MUST NOT normally be manually modified.

When generated code must be version-controlled or analyzed due to the
requirements of a particular technology, the project SHOULD document
that decision.

------------------------------------------------------------------------

## 10. Dependency and Restore Quality

Dependencies MUST be declared through the appropriate .NET project or
centralized dependency-management mechanism.

Dependency versions SHOULD be explicit and reproducible.

Projects SHOULD use centralized package version management when it
provides meaningful consistency across multiple projects within the same
repository.

A clean environment MUST be able to restore the dependencies required to
build and validate the project.

Projects MUST NOT rely on undeclared developer-machine dependencies for
their normal build or validation process.

Known dependency vulnerabilities MUST be evaluated according to their
severity, exploitability, project exposure, and available remediation.

Relevant unresolved vulnerabilities SHOULD be documented when they
cannot be remediated immediately.

Automated dependency-update tooling is NOT required by this standard.

For publicly distributed or reusable .NET packages containing compiled
code, projects SHOULD provide source mapping and portable debugging
symbols when the selected package registry and build model reasonably
support them.

Source Link-compatible source mapping and portable PDB/symbol-package
publication are the preferred .NET mechanisms when applicable. The exact
package, tool, and version used to provide those capabilities belong to
project build configuration and MUST NOT be fixed by this standard.

Published source and symbols MUST correspond to the same validated
source state and binaries represented by the released package.

------------------------------------------------------------------------

## 11. Quality Lifecycle and Blocking Gates

HM .NET quality validation MUST use the following lifecycle unless an approved
project-specific standard is stricter. Delivery MUST NOT be used to repeat
Quality.

### 11.1 Local / Pre-Commit

The local/pre-commit baseline MUST execute, in this order:

`dotnet restore -> dotnet format -> build -> tests`

The build MUST include the configured compiler and Roslyn analyzer rules. A
failure in any applicable step MUST block the local quality gate and therefore
block normal progression to Pull Request.

When the repository contains HM-owned Protocol Buffers governed by the HM
Protobuf/gRPC standard, local validation MUST additionally execute the
applicable `buf format`, `buf lint`, and `buf build` checks.

### 11.2 Pull Request

Every .NET Pull Request targeting the protected integration branch MUST pass:

1. `dotnet format` verification;
2. dependency restore;
3. Release build with compiler/analyzer validation;
4. automated tests;
5. coverage generation;
6. Sonar analysis and the blocking Quality Gate;
7. NuGet Security Audit for projects that consume NuGet packages;
8. GitHub Dependency Review.

NuGet Security Audit MUST evaluate direct and transitive dependencies. Low
severity findings (`NU1901`) are warnings by default. Moderate, high, and
critical findings (`NU1902`, `NU1903`, `NU1904`) MUST block the Pull Request
unless an explicitly governed exception exists.

Dependency Review MUST block dependency changes at moderate severity or higher
by default.

When Protocol Buffers apply, the Pull Request MUST also pass `buf format`,
`buf lint`, `buf build`, and the compatibility/breaking checks required by the
HM Protobuf/gRPC standard.

A failed or indeterminate mandatory PR check MUST block merge.

### 11.3 Main

After integration to `main`, the mandatory conceptual quality gate is:

`coverage -> Sonar analysis -> Quality Gate`

Build and test execution MAY be required mechanically to produce the coverage
and Sonar inputs; when so, they are implementation mechanics of this gate and
not additional Main policies.

Main MUST NOT normally repeat formatting, Dependency Review, NuGet Security
Audit, or Buf quality checks already enforced at the Pull Request boundary.

A mandatory Main Quality Gate MUST be fail-closed. If its required result cannot
be determined, the commit MUST NOT be considered eligible for Delivery.

### 11.4 Phase progression

The blocking progression is:

`Local/Pre-Commit -> Pull Request -> Main -> Delivery eligibility`

Each validation SHOULD execute at the earliest phase capable of blocking the
defect. A later phase MUST repeat a validation only when the repetition
establishes a distinct guarantee for that phase.

Delivery workflows MUST verify that the tagged source commit is an eligible
Main commit, but MUST NOT rerun normal Quality checks such as formatting, tests,
coverage, Sonar, NuGet Security Audit, Dependency Review, or Buf quality checks.

------------------------------------------------------------------------

## 12. Relationship with Other HM Standards

This standard governs .NET code quality.

Repository governance, branch protection, Pull Request requirements, and
general GitHub workflow rules belong to the applicable HM Git/GitHub
standards.

Release triggers, artifact creation, package publication, container
publication, deployment, provenance, and attestation belong to the
applicable HM delivery standards.

Protocol Buffers and gRPC contract governance belongs to the applicable
HM Protobuf/gRPC standards.

Project architecture MAY define additional requirements but MUST NOT
silently weaken mandatory requirements from applicable HM standards.

------------------------------------------------------------------------

## 13. Exceptions and Evolution

This document establishes the HM baseline for .NET quality.

Projects MAY establish stricter requirements.

An intentional deviation from a **MUST** requirement MUST document:

-   the requirement being deviated from;
-   the technical reason;
-   the scope of the exception;
-   relevant quality or operational consequences.

Quality rules SHOULD evolve based on evidence from real HM projects and
changes in the .NET ecosystem.

Specific language, platform, framework, and tooling versions belong to
project architecture or build configuration and MUST NOT be fixed by
this standard unless a version is intrinsically required to define a
standardized protocol or compatibility boundary.

Tool versions and specific implementation mechanisms MAY evolve without
changing the underlying quality principles established by this standard.
