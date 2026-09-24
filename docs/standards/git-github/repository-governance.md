# HM Git & GitHub Repository Governance Standard

## 1. Purpose and Scope

This document defines the baseline governance requirements for Git repositories
and GitHub-hosted projects maintained within the HM ecosystem.

It establishes common requirements for source control protection, change
integration, automated validation, traceability, configuration, secrets, and
workflow security.

The level of additional governance required by a project MAY vary according to
its size, risk, team structure, environments, delivery model, and operational
criticality.

This standard does not prescribe a universal branching model, environment
model, approval policy, release strategy, or deployment strategy.

The terms **MUST**, **SHOULD**, and **MAY** express requirement levels:

- **MUST**: mandatory for compliance with the standard.
- **SHOULD**: recommended unless a documented reason justifies otherwise.
- **MAY**: optional.

---

## 2. Protected Integration Branches

The primary integration branch MUST be protected.

Any additional branch acting as a controlled integration boundary SHOULD also
be protected.

Normal development changes MUST NOT be pushed directly to protected integration
branches.

Changes MUST normally reach a protected integration branch through a Pull
Request.

Repository rulesets or equivalent GitHub protection mechanisms SHOULD enforce
these guarantees whenever technically possible.

The standard does not require a particular branch name or branching model.

Projects MAY introduce additional integration branches when justified by their
development or operational model.

---

## 3. Development Branches

Development work SHOULD occur on branches separate from protected integration
branches.

Branches SHOULD have clear and identifiable purposes.

Projects MAY establish branch naming conventions appropriate to their workflow.

This standard does not prescribe universal branch categories such as
`feature`, `fix`, `develop`, `release`, or environment-specific branches.

More complex branching strategies MUST be introduced deliberately and SHOULD
solve an actual collaboration, integration, release, or operational need.

---

## 4. Pull Request Integration

Pull Requests are the standard mechanism for integrating normal changes into
protected branches.

A Pull Request MUST provide enough information to understand the purpose and
scope of the change.

Required automated validations MUST succeed before merge.

A failing mandatory validation MUST block normal integration.

Human approval requirements MAY vary according to project governance.

Projects maintained by a single developer are not required by this standard
to introduce artificial approval requirements.

Projects involving multiple contributors, higher operational risk, or stronger
segregation-of-duty requirements SHOULD define an appropriate review and
approval policy.

---

## 5. Automated Validation

Changes MUST pass the automated validations applicable to the technologies and
architecture of the project before integration.

This standard does not define technology-specific validation commands.

Applicable HM technology standards define requirements such as:

- formatting;
- compilation or build validation;
- static analysis;
- linting;
- automated testing;
- coverage;
- Quality Gates;
- contract compatibility checks.

Repository governance MUST ensure that mandatory validations cannot be silently
ignored during normal integration.

Validation SHOULD provide fast feedback where practical while preserving the
required quality guarantees.

---

## 6. Change Traceability

Repository history MUST preserve sufficient information to trace an integrated
change through its development and integration lifecycle.

At minimum, it SHOULD be possible to identify the relationship between:

`change -> branch -> commit(s) -> Pull Request -> merge -> integrated commit`

Commit messages MUST clearly communicate the purpose of the recorded change.

Pull Request titles and descriptions SHOULD provide meaningful context about
the change being integrated.

Merge history MUST preserve a clear relationship with the Pull Request that
authorized the integration.

The specific merge strategy MAY vary by project provided that meaningful
traceability is preserved.

Release and deployment traceability are governed by the applicable HM delivery
standards.

---

## 7. Governance Levels and Project-Specific Controls

This document establishes a minimum governance baseline, not a universal
maximum level of control.

Projects MUST evaluate whether additional controls are justified by factors
such as:

- number of contributors;
- operational criticality;
- security exposure;
- regulatory or contractual requirements;
- number of execution environments;
- deployment risk;
- external consumers;
- segregation-of-duty requirements.

Additional controls MAY include:

- mandatory reviewers;
- multiple approvals;
- code owners;
- additional protected branches;
- environment-specific approvals;
- stricter validation gates;
- restricted deployment permissions.

Such controls SHOULD be introduced in response to concrete project needs rather
than applied universally without justification.

Environment and branching models MUST NOT be assumed to have a one-to-one
relationship.

---

## 8. Configuration and Environment Separation

Application and operational configuration MUST remain appropriately separated
from source code when values vary by environment or contain sensitive
information.

Projects SHOULD support environment-specific configuration when multiple
execution environments require different values.

Non-sensitive configuration MAY include values such as:

- host names;
- server addresses;
- ports;
- service names;
- public endpoints;
- environment identifiers;
- operational parameters.

The mechanism used to represent environment-specific configuration MAY vary
according to the technology.

Local development SHOULD support environment-specific configuration through
mechanisms appropriate to the selected technology.

Configuration required by CI/CD SHOULD use GitHub configuration mechanisms
appropriate to its sensitivity and scope.

---

## 9. Secrets and Sensitive Information

Secrets MUST NOT be committed to source control.

Sensitive information includes, but is not limited to:

- passwords;
- private keys;
- access tokens;
- API keys;
- credentials;
- signing material;
- other values that grant privileged access.

Local development secrets MUST use secure mechanisms appropriate to the
technology or development environment.

CI/CD secrets MUST use GitHub Secrets or another approved secure secret
management mechanism.

Projects MAY provide example or template configuration files, but those files
MUST NOT contain real secrets.

Secrets MUST NOT be exposed through workflow output, logs, generated artifacts,
or diagnostic information.

A compromised secret MUST be revocable and replaceable without requiring a
source-code change.

Secrets SHOULD be rotated when exposure is suspected or when operational
security requirements demand it.

---

## 10. GitHub Variables, Secrets, and Environments

GitHub configuration MUST distinguish between sensitive and non-sensitive
values.

Sensitive values MUST use GitHub Secrets or an equivalent secure mechanism.

Non-sensitive operational configuration SHOULD use GitHub Variables when
appropriate.

When a project uses multiple deployment or execution environments,
environment-specific Variables and Secrets SHOULD be scoped through the
corresponding GitHub Environment or equivalent mechanism.

Environment-specific configuration SHOULD remain isolated so that one
environment does not unnecessarily receive configuration or credentials
belonging to another.

GitHub Environments MAY introduce additional protection or approval controls
when justified by project risk.

The existence of multiple environments does not require a specific Git
branching strategy.

---

## 11. Least Privilege

Repository users, automation, workflows, tokens, applications, and deployment
identities MUST receive only the permissions required to perform their intended
operations.

GitHub Actions workflow permissions SHOULD be explicitly constrained whenever
practical.

Write permissions MUST NOT be granted when read-only access is sufficient.

Environment and production credentials SHOULD be scoped to the smallest
necessary operational surface.

Long-lived privileged credentials SHOULD be avoided when a secure temporary or
federated authentication mechanism is available.

OIDC or equivalent short-lived identity mechanisms SHOULD be preferred for
external services that support them.

---

## 12. GitHub Actions Security

Third-party GitHub Actions MUST be treated as executable supply-chain
dependencies.

External Actions SHOULD be pinned to an immutable full commit SHA.

Version tags alone SHOULD NOT be considered equivalent to immutable pinning for
security-sensitive workflows.

Pinned SHAs belong to the version-controlled workflow definition and MUST NOT
be treated as secrets.

Workflow permissions MUST follow the principle of least privilege.

Sensitive values MUST NOT be passed unnecessarily between jobs, steps, or
environments.

Workflow design SHOULD minimize the execution of untrusted code in privileged
contexts.

Security-sensitive workflow changes SHOULD receive additional scrutiny
appropriate to the risk of the repository.

---

## 13. Exceptions and Bypass

Governance controls MUST NOT be routinely bypassed.

Emergency or exceptional bypass MAY be permitted when repository governance
explicitly supports it.

A bypass SHOULD be:

- intentional;
- limited in scope;
- attributable to an authorized actor;
- auditable;
- followed by the required validation as soon as practical.

Projects with higher operational or security risk MAY prohibit bypass entirely
or require additional authorization.

---

## 14. Relationship with Other HM Standards

This standard governs Git and GitHub repository governance.

Technology-specific quality requirements belong to their corresponding HM
standards.

Release triggers, artifact creation, package or container publication,
deployment, environment promotion, rollback, provenance, and attestation
belong to the applicable HM delivery standards.

Project architecture MAY define additional governance requirements but MUST
NOT silently weaken mandatory requirements from applicable HM standards.

---

## 15. Evolution of Governance

HM repository governance SHOULD evolve according to evidence from real
projects and collaboration models.

Additional governance SHOULD be introduced when project scale, team structure,
risk, security, or operational complexity demonstrates a concrete need.

This standard intentionally does not prescribe a universal:

- branching model;
- number of environments;
- reviewer count;
- approval model;
- merge strategy;
- release strategy;
- deployment strategy.

These controls MAY become additional HM standards when sufficient practical
experience demonstrates that they are broadly reusable.

An intentional deviation from a **MUST** requirement MUST document:

- the requirement being deviated from;
- the reason;
- the scope of the exception;
- relevant security, quality, or traceability consequences.
---

## 16. HM Quality and Delivery Workflow Boundaries

For HM projects governed by the standard Quality/Delivery model, repository
configuration MUST preserve the blocking chain:

`Local/Pre-Commit -> Pull Request required checks -> Main Quality eligibility -> Release Tag -> Delivery`

Pull Request branch protection MUST require the applicable checks defined by the
technology quality standards. A commit whose mandatory Main Quality Gate fails
or cannot be determined MUST NOT be treated as Delivery-eligible.

Release workflows MUST be triggered by the approved `v<SemVer>` release tag for
profiles defined by the HM Release & Deployment Standard. Tag-triggered Delivery
MUST verify the exact tagged commit, `main` membership, and Main eligibility; it
MUST NOT use the tag as a bypass around repository Quality gates.

Workflow permissions MUST default to read-only and be elevated at job scope only
for the operation that requires them, including OIDC attestation/publication,
registry publication, deployment, or GitHub Release creation.
