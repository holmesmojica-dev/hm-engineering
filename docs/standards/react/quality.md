# HM React Quality Standard

## 1. Purpose and Scope

This document defines the baseline engineering quality requirements for React
projects developed and maintained within the HM ecosystem.

It applies to React applications, frontends, reusable libraries, and other
React-based software unless a more specific approved standard establishes
stricter requirements.

This standard defines cross-cutting quality expectations. It does not define
application architecture, visual design, styling strategy, routing, SEO,
build-tool selection, release strategy, or deployment strategy.

The terms **MUST**, **SHOULD**, and **MAY** express requirement levels:

- **MUST**: mandatory for compliance with the standard.
- **SHOULD**: recommended unless a documented reason justifies otherwise.
- **MAY**: optional.

---

## 2. Language and Type Safety

TypeScript SHOULD be the preferred language for HM React projects.

When TypeScript is used, projects SHOULD enable strict type checking and
maintain a type-safe codebase appropriate to the project's architecture.

Type errors MUST block integration when type checking forms part of the
project's supported build or validation process.

Projects MUST NOT broadly disable type-safety rules merely to make validation
pass.

Intentional type-safety exceptions SHOULD be technically justified and scoped
to the smallest practical surface.

The specific TypeScript, JavaScript, React, or related platform versions used
by a project MUST be defined by that project's architecture or build
configuration, not by this standard.

---

## 3. Build Quality

A production build MUST complete successfully before changes are integrated
into the protected main branch.

Build failures MUST block integration.

The build process SHOULD detect applicable compilation, type-checking, module
resolution, and production-bundling failures.

Projects MUST NOT rely on undeclared developer-machine state for a successful
production build.

A clean environment SHOULD be capable of reproducing the validated production
build from the repository and its declared dependencies.

---

## 4. Formatting

HM-authored frontend code MUST follow deterministic formatting rules.

Projects MUST provide an automated mechanism to verify formatting without
requiring source files to be modified during CI validation.

Formatting validation SHOULD execute early enough to provide fast feedback.

Formatting rules SHOULD apply consistently to the relevant HM-authored source
and configuration files.

Generated or externally maintained code SHOULD be excluded when formatting it
would provide no meaningful quality benefit.

This standard does not prescribe a specific formatting tool or version.

---

## 5. Static Analysis and Linting

HM React projects MUST use automated static analysis appropriate to the
language and React ecosystem.

Linting MUST execute automatically before changes are integrated into the
protected main branch.

Applicable React-specific correctness rules SHOULD be enabled.

Projects MUST NOT broadly disable relevant linting rules merely to make
validation pass.

Intentional suppressions SHOULD be technically justified and scoped to the
smallest practical surface.

Additional linting rules MAY be introduced when justified by project
architecture, risk, or maintainability requirements.

This standard does not prescribe a specific linter, plugin, or version.

---

## 6. Accessibility Quality

User-facing HM React applications SHOULD integrate automated accessibility
validation into their development and quality process.

Applicable static accessibility rules SHOULD be evaluated during linting or an
equivalent automated validation stage.

Automated accessibility analysis MUST be treated as a quality safeguard, not
as proof that the application is fully accessible.

Projects SHOULD supplement static validation with appropriate component,
interaction, or manual accessibility testing when the user experience or risk
justifies it.

Accessibility rules MUST NOT be broadly disabled solely to make automated
validation pass.

---

## 7. Automated Testing

HM-authored behavior SHOULD be covered by automated tests appropriate to the
type and risk of the project.

Components, hooks, utilities, state transitions, validation logic, and other
testable behavior MUST be tested when failure would materially affect expected
application behavior.

Tests SHOULD prefer observable behavior and public interactions over internal
implementation details.

UI tests SHOULD interact with components in ways that reasonably represent how
consumers or users perceive and use them.

Tests MUST be sufficiently deterministic to provide reliable CI feedback.

Tests MUST NOT unnecessarily depend on developer-specific machine state.

The choice of unit, component, integration, end-to-end, or other test types
MUST be driven by the behavior and risk being validated rather than by a
requirement to maximize test count.

This standard does not prescribe a specific test framework, DOM environment,
testing library, or version.

---

## 8. Code Coverage

HM React projects with automated tests SHOULD measure code coverage.

Coverage MUST be interpreted as a quality signal rather than as proof of
correctness.

Coverage reporting SHOULD focus on HM-authored code.

Generated code and other mechanically produced artifacts SHOULD be excluded
when their inclusion would distort the quality signal.

Other exclusions MAY be introduced when they are justified, reproducible, and
do not conceal meaningful untested behavior.

This standard does not define a universal minimum coverage percentage.

Projects MAY establish stricter coverage requirements based on architecture,
maturity, or risk.

Coverage expectations SHOULD favor meaningful coverage of behavior and new or
changed code over artificial increases intended only to satisfy a numeric
threshold.

---

## 9. SonarQube Quality Analysis

Actively maintained HM React projects SHOULD integrate with the HM-approved
SonarQube quality process.

Projects intended for production use, public distribution, or reuse across the
HM ecosystem MUST use an automated Quality Gate unless an explicit technical
constraint prevents it.

The Quality Gate SHOULD evaluate HM-authored code and avoid allowing generated
or external code to distort quality metrics.

Quality analysis SHOULD include applicable concerns such as:

- reliability;
- maintainability;
- security findings;
- code duplication;
- test coverage.

A failing mandatory Quality Gate MUST block integration into the protected
main branch.

Project-specific SonarQube configuration MAY establish stricter requirements.

This standard does not prescribe a specific SonarQube edition, hosting model,
or version.

---

## 10. Dependency and Installation Quality

Project dependencies MUST be declared through the project's supported package
management mechanism.

Dependency resolution MUST be reproducible through a committed lockfile or an
equivalent deterministic dependency mechanism.

CI SHOULD use the deterministic installation mechanism provided by the
selected package manager.

Projects MUST NOT rely on undeclared globally installed packages or
developer-machine dependencies for their normal build and validation process.

Known dependency vulnerabilities MUST be evaluated according to their
severity, exploitability, project exposure, and available remediation.

Relevant unresolved vulnerabilities SHOULD be documented when they cannot be
remediated immediately.

Automated dependency-update tooling is NOT required by this standard.

This standard does not prescribe a specific package manager or version.

---

## 11. Local Quality Validation

Projects SHOULD provide a clear and reproducible way for developers to execute
the essential quality validations locally before opening or updating a Pull
Request.

Local validation SHOULD reasonably reflect the checks that will later protect
integration in CI.

Projects MAY use Git hooks or other local automation to provide earlier
feedback for formatting, linting, tests, or other inexpensive validations.

Local hooks MUST NOT be treated as a replacement for server-side CI
validation because they may be unavailable, bypassed, or executed in a
different environment.

The standard defines the required quality outcome, not a mandatory local
automation tool.

---

## 12. Continuous Integration Quality Gates

Quality validation MUST occur before changes are integrated into the protected
main branch.

Validation SHOULD be organized to provide fast feedback before more expensive
analysis.

A typical validation progression is:

`format -> lint/static analysis -> tests -> coverage -> production build -> quality analysis`

The exact workflow MAY reorder or parallelize checks when doing so preserves
the required quality guarantees and improves execution efficiency.

At minimum, applicable React validation MUST include:

1. formatting validation;
2. static analysis and linting;
3. automated tests when the project contains testable behavior;
4. production build validation.

Coverage and Quality Gate validation MUST additionally be enforced where
required by this standard or by the project's architecture.

CI MUST NOT silently ignore mandatory validation failures.

---

## 13. Relationship with Other HM Standards

This standard governs React code quality.

Repository governance, branch protection, Pull Request requirements, and
general GitHub workflow rules belong to the applicable HM Git/GitHub
standards.

Release triggers, artifact creation, publication, deployment, rollback,
provenance, and attestation belong to the applicable HM delivery standards.

General concerns that are not inherently specific to React SHOULD be governed
by the corresponding cross-cutting HM standard when one exists.

Project architecture MAY establish additional requirements but MUST NOT
silently weaken mandatory requirements from applicable HM standards.

---

## 14. Exceptions and Evolution

This document establishes the HM baseline for React quality.

Projects MAY establish stricter requirements.

An intentional deviation from a **MUST** requirement MUST document:

- the requirement being deviated from;
- the technical reason;
- the scope of the exception;
- relevant quality or operational consequences.

Quality rules SHOULD evolve based on evidence from real HM projects and
changes in the frontend ecosystem.

Specific language, framework, runtime, package-manager, testing, build, and
tooling versions belong to project architecture or build configuration and
MUST NOT be fixed by this standard.

Tool selection and implementation mechanisms MAY evolve without changing the
underlying quality principles established by this standard.