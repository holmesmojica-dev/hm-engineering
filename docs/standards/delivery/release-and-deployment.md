# HM Release & Deployment Standard

## 1. Purpose and Scope

This document defines how an HM project turns an eligible source state into a
published or deployed release. It is intentionally operational enough that a
new project or automation agent can derive the required delivery workflows
without redesigning the release process.

The terms **MUST**, **SHOULD**, and **MAY** express mandatory, recommended, and
optional requirements respectively.

---

## 2. Standard Delivery Boundary

Quality and Delivery are separate boundaries.

The standard progression is:

`Local/Pre-Commit -> Pull Request -> Main Quality -> Eligible Main Commit -> Release Tag -> Delivery`

For the delivery profiles defined by HM, a pushed release tag `v<SemVer>` is the
explicit delivery intent and the source of the public `ReleaseVersion`.
Delivery MUST verify Quality eligibility; it MUST NOT rerun normal Quality.

Public release tags MUST NOT contain SemVer build metadata (`+...`) when the
same release identity is exposed through registries that cannot represent it
consistently.

---

## 3. Delivery Preflight

Before producing or publishing an artifact, Delivery MUST fail closed unless it
can establish all applicable guarantees:

1. the release tag has the project's accepted `v<SemVer>` grammar;
2. the exact commit referenced by the tag is resolved;
3. that commit belongs to `main` history; it need not be current `main` HEAD;
4. that commit is eligible for Delivery because the mandatory Main gate passed;
5. `ReleaseVersion`, source commit, tag, and planned artifact identities are
   coherent;
6. no existing immutable publication conflicts with the intended release.

The resulting release identity is:

`Repository + SourceCommit + ReleaseTag + ReleaseVersion`

Every artifact/channel produced by the same release workflow MUST represent
that same release identity.

---

## 4. Build/Package Once and Artifact Integrity

A release artifact MUST be built or packaged once from the exact tagged source.
The exact artifact that passes Delivery artifact validation MUST be the artifact
that is attested, published, promoted, or deployed.

Publication jobs MUST NOT rebuild or silently repackage the artifact.
Cross-job artifact transfer MUST preserve integrity. When files are transferred
between jobs, the workflow SHOULD produce and verify a cryptographic manifest
(SHA-256 by default).

Artifact validation belongs to Delivery and MUST validate the final distributable
artifact, not merely the source from which it was produced.

---

## 5. Provenance, Permissions, and Credentials

Public or reusable artifacts SHOULD receive provenance/attestation when the
platform supports it. The attestation MUST refer to the same artifact that is
published.

GitHub Actions MUST use explicit least-privilege permissions. Elevated
permissions belong only to the job that requires them. External Actions SHOULD
be pinned to reviewed immutable full commit SHAs.

OIDC/federated short-lived authentication MUST be preferred when the destination
supports it. Long-lived publishing credentials SHOULD NOT be introduced when a
supported federated mechanism exists.

---

## 6. Publication Verification and Recovery

A publish command succeeding is not sufficient. Each required destination MUST
be remotely verified before its channel is considered complete.

Release workflows MUST be idempotent. On retry, an already-published immutable
identity MUST be verified against the expected release. If it is the expected
publication, the channel is treated as already completed. If identity or content
conflicts, Delivery MUST fail closed.

Multi-channel releases are not assumed to be transactional. A later-channel
failure MUST resume the same release safely rather than create a new version
solely because an earlier immutable channel already succeeded.

---

## 7. NuGet Delivery Profile

This profile applies whenever an HM project publishes a NuGet package.

The release workflow MUST:

1. derive `PackageVersion` from `ReleaseVersion`;
2. build/package once from the tagged source;
3. produce the `.nupkg` and, for compiled reusable libraries, the applicable
   `.snupkg`, portable PDBs, and Source Link information;
4. validate the final package before publication;
5. attest the exact distributable artifacts when supported;
6. publish through NuGet Trusted Publishing/OIDC when available;
7. verify the expected `PackageId + PackageVersion` remotely.

### 7.1 Mandatory NuGet icon

Every HM NuGet package MUST have an icon. Absence of the icon is a publication
blocker.

Delivery validation MUST prove that:

- package metadata declares the icon;
- the referenced icon file exists in the project source;
- the icon is included at the declared path inside the final `.nupkg`;
- the packaged icon is a valid file of the declared/expected format.

README, license, XML documentation, repository metadata, symbols, and other
package contents MUST also be validated when required by the package definition.

Trusted Publisher configuration SHOULD be scoped to the intended repository,
workflow, owner, package, and minimum publication capability supported by
NuGet.org.

---

## 8. BSR Delivery Profile

This profile applies when canonical HM Protocol Buffer schemas are published to
the Buf Schema Registry (BSR).

`buf format`, `buf lint`, `buf build`, and normal compatibility Quality checks
MUST NOT be repeated by Delivery; they belong to the earlier Quality lifecycle.

BSR Delivery MUST:

1. publish the canonical schema state associated with the release identity;
2. use the release tag/identity defined by the release;
3. remotely resolve and verify the resulting immutable BSR publication;
4. verify that the remote schema content/descriptor corresponds to the expected
   local release state;
5. only after successful verification, persist the immutable BSR commit ID as
   the last trusted BSR publication/baseline.

### 8.1 BSR publication retry

The standard BSR publication policy is exactly two attempts. After the first
failed publication attempt, the workflow MUST wait 10 seconds and retry once.
If the second attempt fails, Delivery MUST fail.

The retry MUST NOT weaken post-publication identity/content verification.

---

## 9. NuGet + BSR Multi-Channel Profile

A project MAY publish NuGet only, BSR only, or both. A channel that does not
apply MUST NOT be introduced as an artificial dependency.

When both NuGet and BSR are channels of the same release, the mandatory order is:

`NuGet publish -> NuGet verify -> BSR publish -> BSR verify -> persist trusted BSR commit ID -> GitHub Release`

BSR MUST NOT begin until NuGet is `published` or independently verified as the
already-correct publication for the same release identity.

Both channels MUST represent the same `SourceCommit + ReleaseTag + ReleaseVersion`.
Package-manager versioning and protobuf API compatibility versioning remain
conceptually distinct even when coordinated by one release.

---

## 10. GitHub Release Record

For tag-driven public releases, the GitHub Release MUST be created or verified
only after every required publication/deployment channel for that workflow has
completed successfully.

The GitHub Release MUST correspond to the triggering tag. Release-specific notes
SHOULD live in the release record rather than mutable static package metadata.
The release workflow MUST be safe to retry if the GitHub Release already exists
and corresponds to the expected tag/release identity.

---

## 11. Container and VPS Delivery

Container publication and deployment to an HM-managed VPS are governed by the
companion standard:

`container-vps.md`

That profile extends this foundation with OCI identity, GHCR, final-image
validation, immutable deployment, post-deployment health verification,
known-good state, rollback, and local image retention.

---

## 12. Configuration and Environments

Secrets MUST remain outside source control and use GitHub Secrets or another
approved secret store. Non-sensitive workflow configuration SHOULD use scoped
Variables. Environment-specific credentials/configuration SHOULD use GitHub
Environments when appropriate.

Sensitivity determines **Secret vs Variable**. Scope determines **Repository vs
Environment**.

Artifact, configuration, and secrets are separate concerns. Environment-specific
behavior SHOULD be supplied through runtime configuration rather than rebuilding
the artifact.

---

## 13. Exceptions and Evolution

Projects MAY define stricter controls. A deviation from a **MUST** requirement
requires an explicit documented technical reason, scope, and consequences.

New delivery profiles SHOULD be added only when a real HM project establishes a
new artifact or destination model. Existing profiles MUST be reused rather than
redesigned per project.
