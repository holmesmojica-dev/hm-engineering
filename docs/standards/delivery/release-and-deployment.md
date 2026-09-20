# HM Release & Deployment Standard

## 1. Purpose and Scope

This document defines the baseline requirements for releasing, publishing,
promoting, and deploying software produced within the HM ecosystem.

It applies to deliverable software artifacts such as:

- packages;
- container images;
- web applications;
- APIs and services;
- desktop applications;
- mobile applications;
- reusable contract distributions.

The standard defines delivery guarantees rather than prescribing a universal
pipeline.

Projects MAY use different release and deployment models according to their
artifact type, architecture, operational requirements, and risk.

The terms **MUST**, **SHOULD**, and **MAY** express requirement levels:

- **MUST**: mandatory for compliance with the standard.
- **SHOULD**: recommended unless a documented reason justifies otherwise.
- **MAY**: optional.

---

## 2. Delivery Model

Every project that publishes or deploys software MUST define its delivery
model explicitly.

The model MUST identify, as applicable:

- what artifact or application is delivered;
- what event initiates publication or deployment;
- what validations are required;
- where the artifact is published;
- which environments receive deployments;
- how the delivered version is identified;
- how delivery can be traced back to source.

This standard does not prescribe a universal release trigger.

Valid models MAY include, among others:

- release-tag-driven publication;
- integration-to-main continuous deployment;
- manual promotion;
- environment-controlled deployment.

The selected model MUST preserve the guarantees established by this standard.

---

## 3. Validation Before Delivery

Software MUST pass all mandatory quality and compatibility validations
applicable to the project before it is published or deployed.

A failed mandatory validation MUST block normal delivery.

Applicable validations are defined by the relevant HM technology,
architecture, contract, and repository-governance standards.

Delivery automation MUST NOT silently bypass mandatory quality gates.

---

## 4. Artifact Identity and Immutability

Every released or deployed artifact MUST have an unambiguous identity.

Depending on the artifact type, identity MAY be represented by:

- semantic version;
- release tag;
- commit SHA;
- container digest;
- immutable build identifier;
- another equivalent immutable identifier.

Mutable aliases such as `latest`, `main`, or environment names MAY exist for
convenience, but MUST NOT be the only means of identifying a delivered
artifact when an immutable identity can be provided.

A production deployment MUST be traceable to the exact artifact that was
deployed.

---

## 5. Build Once and Deliver the Validated Artifact

When technically applicable, HM delivery pipelines SHOULD build an artifact
once and deliver or promote that same validated artifact.

A release or deployment SHOULD NOT rebuild, repackage, or otherwise modify an
artifact after the validation associated with that artifact has completed.

When multiple environments are used, promotion SHOULD preserve artifact
identity whenever the platform and delivery model allow it.

Environment-specific behavior SHOULD be provided through external
configuration rather than by rebuilding environment-specific application
binaries.

---

## 6. Delivery Traceability

Every release or production deployment MUST provide sufficient traceability
to reconstruct its delivery chain.

The intended relationship is:

`source commit -> validated build -> immutable artifact -> release/deployment -> destination or environment`

Where release tags are used, the tag MUST identify the source state associated
with the released artifact.

Where registries or package platforms are used, the published artifact SHOULD
be traceable to its repository and source revision.

Delivery automation SHOULD preserve identifiers that allow operators and
developers to determine what version is currently deployed.

---

## 7. Environments

Projects MAY define the environments required by their operational model.

Examples include:

- development;
- testing;
- staging;
- production.

This standard does not require every project to implement multiple
environments.

Environment requirements MUST be driven by project risk, architecture,
operational needs, and delivery complexity.

Git environments and Git branches MUST NOT be assumed to have a one-to-one
relationship.

A project MAY use a simple integration model while promoting the same artifact
through multiple execution environments.

---

## 8. Delivery Configuration

Delivery configuration MUST remain appropriately separated from application
source code.

Configuration values SHOULD be scoped according to where they are required.

For GitHub-based delivery:

- sensitive values MUST use GitHub Secrets or an approved secure secret
  management mechanism;
- non-sensitive operational values SHOULD use GitHub Variables where
  appropriate;
- environment-specific values SHOULD be scoped to the corresponding GitHub
  Environment when one exists.

The governing principle is:

**Sensitivity determines Secret vs Variable.  
Scope determines Repository vs Environment.**

Values belonging exclusively to a particular environment SHOULD NOT be given
repository-wide scope without a concrete reason.

Delivery workflows MUST NOT unnecessarily duplicate configuration already
owned by the runtime environment.

---

## 9. Delivery Credentials and Permissions

Delivery automation MUST follow the principle of least privilege.

Credentials MUST receive only the permissions required for their delivery
operation.

Write or deployment permissions SHOULD be scoped to the smallest practical
workflow, job, environment, repository, or artifact boundary.

Long-lived credentials SHOULD be avoided when secure temporary or federated
authentication is supported.

OIDC or equivalent short-lived identity mechanisms SHOULD be preferred when
the destination platform supports them.

Production credentials MUST NOT be exposed to workflows or jobs that do not
require production access.

---

## 10. Production Deployment

Production deployment MUST occur only after the artifact and source state have
passed all mandatory validations.

The deployed version MUST be identifiable.

Where an executable application or service can be operationally verified,
production deployment SHOULD include an appropriate post-deployment
verification.

Depending on the system, verification MAY include:

- health checks;
- smoke tests;
- availability checks;
- application startup verification;
- other non-destructive operational checks.

A deployment MUST NOT be considered successfully completed when its mandatory
post-deployment verification fails.

---

## 11. Rollback and Recovery

Every production deployment MUST have a defined recovery or rollback strategy.

The strategy MUST make it possible to restore an operationally acceptable
state when a deployment fails or introduces unacceptable behavior.

The mechanism MAY vary according to the architecture and delivery model.

Examples include:

- redeploying a previously validated artifact;
- restoring a previous immutable container image;
- switching to a previous application release;
- platform-native rollback;
- another documented recovery mechanism.

Production rollback MUST NOT depend on rebuilding an unknown or historically
different version of the application when a previously validated artifact can
be retained and reused.

Projects using staging or other operationally significant pre-production
environments SHOULD define rollback or recovery there when project risk
justifies it.

Rollback procedures SHOULD be periodically validated when the operational
risk of the project warrants it.

---

## 12. Package and Artifact Publication

Published packages and reusable artifacts MUST correspond to a validated
source state.

Publication MUST preserve a clear relationship between the published version
and its source commit or release tag.

When a single release is distributed through multiple channels, those
distributions SHOULD originate from the same authoritative source state.

Publication automation MUST NOT silently alter the semantics of an artifact
after its release validation.

Package-manager versioning and protocol/API compatibility versioning MUST
remain distinct when both concepts exist.

---

## 13. Provenance and Attestation

Packages and distributable artifacts published to public or private registries
SHOULD include provenance or attestation when the registry, build platform,
and publication mechanism reasonably support it.

Provenance SHOULD allow the published artifact to be associated with:

- its source repository;
- source commit;
- release tag when applicable;
- validated build or workflow;
- publication process.

The artifact represented by provenance SHOULD be the same artifact that was
validated and published.

Provenance and attestation provide evidence of artifact origin and build
history but MUST NOT be assumed to be equivalent to a cryptographic package
signature unless an actual signing mechanism is used.

Projects MAY introduce explicit artifact signing when security or distribution
requirements justify it.

---

## 14. Client Version Awareness

Client applications SHOULD be capable of identifying their deployed or
installed version when doing so provides operational or user value.

Web applications with long-lived client sessions SHOULD support detection of
a newer deployed version when practical.

When a newer version is detected, the application SHOULD provide an
appropriate mechanism to inform the user that an update is available.

Mobile and desktop applications SHOULD similarly support update awareness when
appropriate to their distribution model.

The delivery process SHOULD expose sufficient build or version identity to
support these mechanisms.

User-interface behavior and presentation of update notifications belong to the
applicable client architecture or user-experience standards.

---

## 15. Required Updates

A client application MAY require an update when continuing with the installed
or loaded version would create a concrete operational concern.

Examples include:

- incompatibility with backend contracts;
- security vulnerabilities;
- unsupported application versions;
- required protocol changes;
- other explicitly defined compatibility constraints.

Normal releases SHOULD NOT force client updates without a concrete reason.

Projects requiring forced-update behavior MUST define how minimum supported
versions are determined and communicated.

---

## 16. Manual Delivery

Manual delivery MAY be supported when justified by the project.

Manual execution MUST NOT bypass mandatory validation, traceability,
credential protection, artifact identity, or production recovery
requirements.

A manually initiated production deployment MUST provide the same essential
delivery guarantees as an automatically triggered deployment.

---

## 17. Relationship with Repository Governance

Repository governance determines how source changes are protected, validated,
reviewed, and integrated.

This standard begins at the delivery boundary and governs how validated source
states become published or deployed software.

A repository's branching strategy MUST NOT implicitly define its environment
architecture unless the project explicitly chooses and documents that model.

Protected branches, Pull Request integration, repository secrets, workflow
security, and source-control traceability are governed by the applicable HM
Git & GitHub Repository Governance Standard.

---

## 18. Exceptions and Evolution

Delivery architecture SHOULD evolve from real project requirements rather than
from speculative complexity.

This standard intentionally does not prescribe a universal:

- release trigger;
- branching strategy;
- number of environments;
- deployment platform;
- artifact registry;
- package manager;
- container technology;
- approval count;
- rollback implementation;
- deployment frequency.

Projects MAY introduce stricter delivery controls according to operational,
security, contractual, regulatory, or collaboration requirements.

An intentional deviation from a **MUST** requirement MUST document:

- the requirement being deviated from;
- the reason;
- the scope of the exception;
- relevant security, quality, traceability, or operational consequences.