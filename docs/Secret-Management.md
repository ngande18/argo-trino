# GitOps-Based Secrets Management with HashiCorp Vault

## Overview

This document describes the **GitOps-driven approach** used to manage secrets and sensitive information for Kubernetes workloads using **HashiCorp Vault**, **Helm**, and **ApplicationSets**.

The solution is designed to:

* Centralize all sensitive data in Vault
* Keep Kubernetes clusters free of plaintext secrets in Git
* Support multi-instance and multi-namespace deployments
* Handle **key-value secrets**, **config files**, and **TLS assets** consistently

All secrets are rendered into Kubernetes **Secrets** at deploy time using a **custom Helm chart** maintained by our team.

---

## High-Level Architecture

**Key Components**

* HashiCorp Vault (system of record for secrets)
* GitOps controller (ApplicationSet-driven deployments)
* Custom Helm chart for secret materialization
* Kubernetes clusters (example: E7000)

**Flow Summary**

1. Secrets are stored securely in Vault, organized by cluster and namespace
2. ApplicationSets deploy applications per instance
3. Helm values define Vault mount paths and secret metadata
4. Helm templates read from Vault and generate Kubernetes Secrets
5. Applications consume secrets via env vars or mounted files

---

## Vault Secret Organization

Secrets in Vault are structured hierarchically for clarity and isolation.

```
<cluster-name>/
  └── <namespace>/
       ├── kv/        # Key-value secrets (passwords, tokens)
       ├── config/    # Config files (krb5.conf, keytabs, keystores)
       └── tls/       # TLS certificates and keys
```

### Example

```
e7000/
  └── trino-namespace/
       ├── kv/
       │    ├── db_password
       │    └── ldap_bind_password
       ├── config/
       │    ├── krb5.conf
       │    └── trino.keytab
       └── tls/
            ├── tls.crt
            └── tls.key
```

---

## Secret Types Supported

### 1. Key-Value Secrets (Passwords / Tokens)

* Stored under the `kv` path in Vault
* Rendered as Kubernetes Secrets
* Automatically injected as **environment variables** into pods

**Usage in Application (Example)**

```bash
export DB_PASSWORD=$DB_PASSWORD
```

The environment variable name is derived from the Helm values configuration.

---

### 2. Config Files (Plain Text & Binary)

#### Plain Text Files

* Examples: `krb5.conf`, application config files
* Stored as plaintext in Vault
* Mounted directly into the container filesystem

**Mount Location (Trino Example)**

```
/etc/trino/config-files/krb5.conf
```

#### Binary Files (Keytabs, Java Keystores)

* Examples: `.keytab`, `.jks`, `.p12`
* Stored in Vault as **base64-encoded content**
* Decoded at runtime using an **init container**

**Init Container Responsibilities**

* Read base64-encoded secret from mounted path
* Decode to binary format
* Write to shared volume for main container

---

### 3. TLS Assets

* Certificates and keys stored under the `tls` path
* Rendered as Kubernetes TLS Secrets
* Mounted or referenced by applications as required

---

## Helm Chart Design

A **custom Helm chart** is used to materialize secrets from Vault.

### Values File Responsibilities

* Define Vault mount points
* Specify secret type (kv / config / tls)
* Map Vault paths to Kubernetes Secret names
* Control env var injection vs file mounts

**Key Characteristics**

* One values file per application instance
* Same chart reused across clusters and namespaces
* No secret values stored in Git

---

## GitOps Integration with ApplicationSets

* Each ApplicationSet represents an application instance
* ApplicationSet passes instance-specific values to Helm
* Helm renders secrets dynamically from Vault
* Changes in Git = declarative updates
* No manual secret creation in clusters

---

## Trino-Specific Implementation Notes

### Environment Variables

* All key-value secrets are injected automatically
* Trino uses standard `ENV` references

### Config Files

* Plain text configs mounted directly under Trino config directory
* Binary configs decoded via init containers

### Init Container Pattern

* Shared emptyDir volume
* Init container decodes base64 files
* Main container consumes decoded artifacts

---

## Scripts and Automation

A `scripts/` directory is maintained to simplify secret operations.

**Typical Use Cases**

* Bulk upload secrets to Vault
* Standardize secret creation
* Reduce manual errors

Scripts are self-driven and can be reused across environments.

---

## Security and Best Practices

* No plaintext secrets in Git repositories
* Vault is the single source of truth
* Namespace-level isolation in Vault
* Reusable Helm logic, minimal duplication
* GitOps ensures auditability and consistency

---

## Summary

This GitOps-based Vault integration provides:

* Secure, scalable secret management
* Clean separation of concerns
* Reusable patterns across clusters and applications
* Seamless integration with Trino and other workloads

This README can be extended further with diagrams, onboarding steps, or troubleshooting sections as needed.
