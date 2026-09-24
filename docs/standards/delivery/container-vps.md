# HM Container + VPS Delivery Profile

## 1. Purpose and Applicability

This profile applies when an HM application is packaged as an OCI/Docker image,
published to a container registry, and deployed to an HM-managed VPS using
Docker Compose.

It extends `release-and-deployment.md`; all Delivery Foundation requirements
remain mandatory.

---

## 2. Standard Flow

The standard flow is:

`Eligible Main -> v<SemVer> Tag -> Preflight -> Build image once -> Validate final image -> Publish GHCR -> Verify -> Deploy exact image -> Health verification -> Mark known-good -> Controlled cleanup -> GitHub Release`

A failure before production is changed MUST abort Delivery without rollback. A
failure after production is changed and before the candidate becomes known-good
MUST invoke the automatic rollback policy.

---

## 3. Registry and Image Identity

GHCR (`ghcr.io`) is the default HM registry for this profile unless project
architecture documents another registry.

The release MUST preserve all of these identities:

- `ReleaseTag` / `ReleaseVersion` for human release identity;
- full source commit SHA for source traceability;
- a source-SHA image tag for operational traceability;
- OCI image digest (`sha256:...`) as the immutable image identity.

Human-readable tags MAY exist, including release-version or source-SHA tags, but
production deployment MUST use or resolve to the exact immutable digest that was
published and verified. Mutable aliases such as `main` MUST NOT be the
operational source of truth for production.

All image tags emitted by one release workflow MUST resolve to the same image
content/digest when they claim the same release artifact.

---

## 4. Build Once and Validate the Final Container

The workflow MUST build the production container image once from the exact
tagged source. It MUST validate the final OCI image itself before production
deployment; validating only the .NET source/build outside the container is not
sufficient Delivery validation.

Applicable validation SHOULD include image creation success, expected image
identity/metadata, container startup, and a non-destructive health or smoke check
against the final image where practical.

The validated image MUST be the same image published to GHCR and deployed.
Delivery MUST NOT rebuild the image on the VPS.

---

## 5. GHCR Publication and Verification

The workflow MUST publish the validated image to GHCR before connecting the VPS
to the candidate deployment.

After publication, it MUST resolve/verify the remote image and expected digest.
If the publication cannot be verified, deployment MUST NOT begin.

Registry permissions MUST follow least privilege. The VPS SHOULD use a dedicated
read-only registry credential when authentication is required.

---

## 6. VPS Deployment Model

The VPS is a runtime host, not a build server.

The deployment workflow MUST:

1. identify the exact verified candidate digest;
2. connect using the approved deployment identity;
3. pull the exact candidate image;
4. validate the effective Docker Compose configuration;
5. start/update the service with Docker Compose;
6. wait for the service/container health condition;
7. execute the required application health verification;
8. only then mark the candidate as known-good.

Production services SHOULD expose application ports only as required by the
runtime architecture. For a reverse-proxied single-host service, binding the
application to loopback and exposing public traffic through the reverse proxy is
the preferred baseline.

Runtime secrets/configuration MUST remain external to the image.

---

## 7. Known-Good State and Deployment History

A candidate MUST NOT replace known-good state until all mandatory deployment and
post-deployment verification succeeds.

The minimum identity recorded for a successful deployment is:

`ReleaseTag + ReleaseVersion + SourceCommit + ImageTag + ImageDigest`

Git/GitHub provides the logical release history and GHCR provides the retained
immutable artifact history. HM MUST NOT create a second full artifact-history
system merely to duplicate those sources.

Operational state MUST at minimum preserve a pointer to the current/last
known-good immutable image digest. Deployment records SHOULD retain enough
information to identify successful deployments and rollback events.

---

## 8. Automatic Rollback

If production has already been changed and a mandatory deployment/startup/health
verification fails, the workflow MUST automatically attempt to restore the
previous known-good image.

Automatic rollback MUST:

1. use the previously published immutable known-good artifact;
2. never rebuild historical source;
3. redeploy it through the same runtime mechanism;
4. run the same mandatory post-deployment verification;
5. preserve the previous known-good pointer unless a new candidate succeeds;
6. report both the original deployment failure and rollback outcome.

If automatic rollback also fails, the workflow MUST fail critically and require
operator intervention; it MUST NOT report successful Delivery.

---

## 9. Manual Rollback Workflow

Projects using this profile MUST provide a manually triggered rollback workflow
(for GitHub Actions, `workflow_dispatch` or an equivalent governed mechanism).

Manual rollback exists for defects discovered after Delivery succeeded, such as
business or functional defects that health checks could not detect.

The operator MUST be able to select a previously known-good release/image, not
only the immediately previous one. The workflow MUST verify that the selected
immutable artifact exists in GHCR, deploy that exact digest, run the normal
post-deployment verification, and record the outcome.

Manual rollback MUST NOT rebuild source and MUST NOT rerun normal Quality.

---

## 10. Retention and Cleanup

GHCR is the long-term source of deployable historical container artifacts.
Retention MUST preserve the known-good versions needed by the project's rollback
policy.

The VPS is only an operational cache. By default, after a successful deployment
and known-good update, it SHOULD retain at most three local images of the HM
application:

1. the currently running image;
2. the previous known-good image;
3. one additional recent fallback image.

Older local application images SHOULD be removed with targeted cleanup. A broad
or indiscriminate Docker prune MUST NOT be used by the deployment workflow.

If rollback requires an older retained release, the workflow pulls the exact
immutable image again from GHCR.

---

## 11. VPS Security Baseline

A production VPS using this profile MUST apply least privilege and a deny-by-
default network posture appropriate to the service.

The baseline is:

- SSH key authentication for automation; password authentication SHOULD be
  disabled for production administration;
- direct root SSH SHOULD be disabled;
- human administration and CI deployment identities SHOULD be separate;
- CI SSH capabilities SHOULD be restricted where practical (for example no
  unnecessary forwarding, TTY, or X11);
- membership/access to the Docker daemon MUST be treated as host-root-equivalent
  privilege;
- deployment/configuration files containing secrets MUST use restrictive file
  permissions;
- firewall ingress MUST expose only required ports;
- public HTTP(S) SHOULD terminate at the approved reverse proxy/TLS layer when
  that architecture is used;
- SSH host identity MUST be verified (for example through managed
  `known_hosts`); disabling host-key verification is not acceptable.

Brute-force protection MAY use Fail2ban or an equivalent mechanism when the
host's exposure/risk model justifies it.

---

## 12. GitHub Release

After the container has been published, deployed, verified, and marked
known-good, the release workflow MUST create or verify the GitHub Release for the
triggering tag. A deployment that was automatically rolled back MUST NOT be
recorded as a successfully completed release deployment.
