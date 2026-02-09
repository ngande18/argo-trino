# Trino Catalog Deployment (GitOps + Vault)

## Purpose

This document describes the **standardized process for deploying additional Trino catalogs** using a **GitOps-driven approach**, with **HashiCorp Vault** as the system of record for all sensitive information.

The goal is to ensure:

* Secure handling of all catalog-related secrets
* Consistent, repeatable catalog deployments
* A single source of truth per Trino instance
* Zero manual secret handling in the cluster

---

## Core Principle

> **No catalog is added unless all required sensitive information is already available in HashiCorp Vault and injected into the target namespace.**

Catalog configuration and secrets are deliberately **decoupled**:

* **Vault** manages sensitive data
* **Trino Helm values.yaml** manages catalog definitions

---

## High-Level Workflow

1. Identify the catalog type and its required secrets
2. Ensure all sensitive information is present in Vault
3. Verify Vault → Kubernetes secret injection for the namespace
4. Define the catalog properties in Trino `values.yaml`
5. Deploy via GitOps (Helm / ApplicationSet)
6. Trino automatically loads the new catalog

---

## Step 1: Identify Required Sensitive Information

Each catalog may require a different set of secrets depending on the connector and authentication model.

### Common Examples

| Catalog Type            | Sensitive Information Required                   |
| ----------------------- | ------------------------------------------------ |
| S3 / Object Storage     | Access Key, Secret Key, Session Token (optional) |
| Kerberos-backed sources | `krb5.conf`, keytab file                         |
| TLS-secured sources     | Java keystore, truststore, passwords             |
| Databases (JDBC)        | Username, password, TLS material                 |

Before proceeding, confirm **exactly which secrets** are needed.

---

## Step 2: Vault Secret Preparation

All sensitive information **must be stored in HashiCorp Vault** under the correct cluster and namespace path.

### Standard Vault Layout

```
<cluster-name>/<namespace>/
  ├── kv/        # passwords, tokens, access keys
  ├── config/    # keytabs, keystores, krb5.conf
  └── tls/       # certificates and private keys
```

### Examples

* S3 access keys → `kv/`
* Java keystore (`.jks`, `.p12`) → `config/` (base64 encoded)
* Kerberos keytab → `config/` (base64 encoded)

Once stored, Vault injection must already be configured for the namespace.

---

## Step 3: Verify Vault Injection for Namespace

Before adding a catalog:

* Confirm the namespace already uses the **custom Vault secrets Helm chart**
* Ensure Kubernetes Secrets are being created successfully
* Validate:

  * Key-value secrets are exposed as environment variables
  * Config files are mounted at the expected Trino paths
  * Binary files are decoded via init containers

If injection is missing or incomplete, **do not proceed with catalog creation**.

---

## Step 4: Define Catalog Properties

Once secrets are available, define the catalog configuration.

Catalogs are managed **centrally** in the Trino Helm `values.yaml` file.

### Key Design Rule

> **Each Trino instance has exactly one values.yaml that acts as the single source of truth for all its catalogs.**

### Example Catalog Definition (Conceptual)

* Connector type
* Authentication method
* References to env vars or mounted files
* No hardcoded secrets

Sensitive values must always reference:

* Environment variables injected from Vault
* Files mounted from Vault-backed Kubernetes Secrets

---

## Step 5: GitOps Deployment

Once the catalog configuration is committed:

* GitOps controller (ApplicationSet) syncs the change
* Helm renders updated Trino configuration
* New catalog files are generated automatically
* Trino reloads catalogs without manual intervention

No direct changes are made inside the cluster.

---

## Trino Behavior

* Trino reads catalog definitions at startup
* Each catalog appears as a `.properties` file
* All authentication material is resolved at runtime
* No secret values are logged or stored in plaintext

---

## Operational Guidelines

### Do

* Treat Vault as the single source of truth for secrets
* Validate secret injection before catalog changes
* Keep catalog logic declarative
* Use one values.yaml per Trino instance

### Do Not

* Hardcode secrets in values.yaml
* Create Kubernetes Secrets manually
* Reuse secrets across namespaces improperly
* Add catalogs without Vault readiness

---

## Pull Request (PR)–Driven Catalog Lifecycle

All Trino catalog changes follow a **Pull Request–based workflow** to ensure auditability, peer review, and controlled rollout.

### Why PR-Based Management

* Enforces separation of duties (author vs reviewer)
* Provides a clear change history for catalogs and secrets
* Prevents accidental or unauthorized catalog creation
* Aligns with GitOps best practices

---

## End-to-End PR Workflow

### Step 1: Create a Feature Branch

Create a new branch from the main (master) branch:

* Branch naming example:

  * `feature/add-iceberg-catalog`
  * `feature/add-s3-backed-catalog`

This branch will contain **both**:

* Vault-related secret updates (Azure Cloud Vault → HashiCorp Vault)
* Trino catalog configuration changes

---

### Step 2: Add / Update Sensitive Information

As part of the same branch:

* Upload required sensitive information to **Azure Cloud Vault**
* Sync secrets into **HashiCorp Vault** under the correct path
* Validate secrets exist for the target:

  * Cluster
  * Namespace

No Kubernetes Secrets are created manually.

---

### Step 3: Update Trino Catalog Configuration

Update the Trino Helm `values.yaml` for the target instance:

* Add a new catalog entry under `additionalCatalogs`
* Reference:

  * Environment variables (for KV secrets)
  * Mounted files (for keystores, keytabs, TLS)
* Do **not** embed secret values

This file remains the **single source of truth** for all catalogs in that Trino instance.

---

### Step 4: Open a Pull Request

Open a PR targeting the `master` branch.

The PR should include:

* Clear description of the catalog being added
* Connector type and data source
* List of required secrets
* Confirmation that Vault injection is complete

---

### Step 5: Peer Review and Validation

Reviewers validate:

* Correct Vault paths and namespace usage
* No hardcoded secrets in Git
* Proper catalog property configuration
* Alignment with security and naming standards

PR approval is required before merge.

---

### Step 6: Merge and Automatic Deployment

Once the PR is merged:

* GitHub becomes the trigger point
* GitOps controller detects changes
* Helm re-renders Trino configuration
* New catalog `.properties` file is generated
* Trino instances automatically load the catalog

No manual deployment or restart steps are required.

---

## Operational Guarantees

This workflow guarantees:

* Every catalog change is traceable to a PR
* Every catalog has a reviewer-approved design
* Secrets always exist before catalogs are activated
* Trino instances remain declarative and reproducible

---

## Summary

By enforcing a PR-driven workflow, Trino catalog management becomes:

* Secure
* Auditable
* Repeatable
* Fully GitOps-aligned

Catalogs are added only through code review, secrets are always Vault-backed, and Trino stays consistent across all environments.

---

*This PR workflow can be extended with CI checks, policy validation, or automated secret verification if needed.*
