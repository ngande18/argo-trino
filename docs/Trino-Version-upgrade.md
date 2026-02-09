# Trino Version Upgrade – Design & Validation Considerations

## Purpose

This document defines the **mandatory considerations and validation steps** before onboarding or upgrading to a **new Trino version** in our platform.

The intent is to:

* Prevent regressions in authentication and authorization
* Ensure compatibility with existing GitOps workflows
* Maintain consistency across Trino images, Helm, and ApplicationSets

This document must be reviewed **before** any Trino version bump is proposed via Pull Request.

---

## Core Constraint: LDAP Group Authentication

### Current State

* We are **not using Trino’s upstream/default LDAP group authentication**
* We rely on a **custom LDAP Active Directory group authentication plugin**
* This custom behavior is critical for authorization and access control

As a result, **every Trino version upgrade must explicitly validate LDAP behavior**.

---

## Step 1: Validate Upstream LDAP Group Support

Before building or deploying a new Trino version:

1. Review the Trino release notes for LDAP-related changes
2. Deploy the new Trino version in a test environment **without custom plugins**
3. Validate whether:

   * Upstream LDAP group authentication works as expected
   * Group resolution, mapping, and authorization rules behave correctly

### Decision Point

* ✅ **If upstream LDAP group auth works**:

  * The standard Trino image can be used
* ❌ **If upstream LDAP group auth does not work or regresses**:

  * A custom Trino image **must be built** with our LDAP AD plugin

---

## Step 2: Custom Trino Image Build (If Required)

### Image Build Strategy

Custom Trino images are built using a **dedicated image build pipeline**.

* Image build definitions live under the `image-builds/` directory
* The custom LDAP AD plugin is included during the image build
* No manual image creation is permitted

### GitHub Workflow Integration

* Image builds are driven by **GitHub Actions workflows**
* The workflow:

  * Pulls the target Trino version
  * Injects the custom LDAP AD plugin
  * Builds and tags the new image
  * Pushes the image to the approved container registry

This ensures:

* Reproducibility
* Auditability
* Consistent image provenance

---

## Step 3: Helm Chart and Repository Handling

### Helm Dependency Requirement

For ApplicationSets to function correctly:

* Helm charts **must be locally available**
* Remote-only Helm references are not supported in this workflow

### Required Action

For a new Trino version:

1. Perform a `helm pull` of the Trino chart
2. Save (vendor) the chart into the designated repository directory
3. Ensure the chart version aligns with the Trino version being tested

This step is mandatory to:

* Enable deterministic GitOps behavior
* Allow ApplicationSets to resolve charts reliably

---

## Step 4: ApplicationSet Compatibility Validation

Before proposing a version upgrade:

* Validate that ApplicationSets:

  * Reference the correct chart path
  * Reference the correct Trino image tag
  * Continue to render Helm templates correctly

Any incompatibility must be resolved **before** opening a PR.

---

## Step 5: Upgrade Readiness Checklist

A Trino version upgrade is considered **ready for PR** only if:

* [ ] Upstream LDAP group auth is validated
* [ ] Custom image build decision is documented
* [ ] Custom image (if needed) is built via GitHub workflow
* [ ] Helm chart is pulled and vendored locally
* [ ] ApplicationSets render successfully
* [ ] No breaking changes identified in authentication paths

---

## Pull Request Expectations

Every Trino version upgrade PR must include:

* Target Trino version
* LDAP validation outcome
* Image strategy (upstream vs custom)
* Helm chart version
* Confirmation of ApplicationSet compatibility

PRs missing this information should not be approved.

---

## Summary

Trino version upgrades are **not a simple version bump**.

They require careful validation across:

* Authentication (LDAP group behavior)
* Image build strategy
* Helm chart management
* GitOps and ApplicationSet compatibility

By following this document, we ensure Trino upgrades remain secure, predictable, and production-safe.

---

*This document can be extended with version-specific findings or historical upgrade notes as needed.*
