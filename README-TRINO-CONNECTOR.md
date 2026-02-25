# CloudBeaver – Trino Connector Setup

## Environment Overview

| Component | Value |
|-----------|-------|
| **CloudBeaver URL** | https://telecode-ipc1.aexp.com |
| **Trino Endpoint** | maestro-ipc1.aexp.com |
| **Trino Port** | 443 |
| **Trino Version** | 479 |
| **SSL** | Enabled |
| **Admin User** | cadmin |
| **Admin Secret** | hvm-secret-files |
| **Deployment** | Helm Chart |

---

## Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [SSL & Truststore Architecture](#ssl--truststore-architecture)
4. [CloudBeaver Login](#cloudbeaver-login)
5. [Add Trino Connector](#add-trino-connector)
6. [Custom Docker Image](#custom-docker-image)
7. [Certificate Management](#certificate-management)
8. [Troubleshooting](#troubleshooting)
9. [Security Notes](#security-notes)

---

## Overview

This document describes the procedure to configure a Trino connector in CloudBeaver deployed via Helm on Kubernetes. The setup includes:

- **CloudBeaver Instance**: https://telecode-ipc1.aexp.com
- **Trino Cluster**: maestro-ipc1.aexp.com:443
- **Custom Image**: Built with Trino JDBC driver version 479
- **Security**: SSL/TLS with Java truststore and certificates managed via HashiCorp Vault

---

## Prerequisites

### Infrastructure Requirements

✅ **CloudBeaver Deployment**
- Deployed via Helm chart on Kubernetes
- Custom Docker image with Trino JDBC driver v479
- Java truststore (`amex-truststore.jks`) injected via HashiCorp Vault

✅ **Network Access**
- CloudBeaver can reach `maestro-ipc1.aexp.com:443`
- SSL/TLS enabled with proper certificate validation

✅ **Credentials**
- Admin username: `cadmin`
- Admin password stored in Kubernetes secret: `hvm-secret-files`

### Retrieve Admin Password

If you need to retrieve the admin password:

```bash
kubectl get secret hvm-secret-files -n <namespace> -o jsonpath='{.data.cadmin-password}' | base64 -d
```

Replace `<namespace>` with your CloudBeaver namespace.

---

## SSL & Truststore Architecture

### Truststore Configuration

The Java truststore is managed through HashiCorp Vault and mounted into the CloudBeaver pod:

| Property | Value |
|----------|-------|
| **Truststore File** | `amex-truststore.jks` |
| **Mount Path** | `/etc/truststore/configfiles/output/amex-truststore.jks` |
| **Source** | HashiCorp Vault |
| **Certificate Management** | HashiCorp Vault (automated rotation) |

### JVM Configuration

CloudBeaver is configured with the following JVM options to use the truststore:

```bash
-Djavax.net.ssl.trustStore=/etc/truststore/configfiles/output/amex-truststore.jks
-Djavax.net.ssl.trustStorePassword=${TRUSTSTORE_PASSWORD}
```

**Note**: The truststore is added as part of Java environment variables using the `javax.net.ssl.*` system properties. These properties are set at JVM startup and apply globally to all SSL/TLS connections made by the CloudBeaver application.

This ensures:
- ✅ Proper TLS handshake with `maestro-ipc1.aexp.com`
- ✅ PKIX validation succeeds
- ✅ Certificates are managed centrally via Vault
- ✅ No hardcoded certificates in the image
- ✅ Truststore configured at Java environment level (not application level)

---

## CloudBeaver Login

### Step 1: Access CloudBeaver

Open your browser and navigate to:

```
https://telecode-ipc1.aexp.com
```

### Step 2: Login

Use the following credentials:

- **Username**: `cadmin`
- **Password**: Retrieved from `hvm-secret-files` Kubernetes secret

---

## Add Trino Connector

### Step 1: Navigate to Administration

1. Click **Administration** in the top menu
2. Select **Connections** from the left sidebar
3. Click **Create Connection** button

### Step 2: Select Trino Driver

1. In the driver selection dialog, choose **Trino**
2. Verify the driver version is **479** (included in custom image)

### Step 3: Configure Connection

Fill in the connection details:

| Field | Value | Notes |
|-------|-------|-------|
| **Connection Name** | `Trino Production` | Or any descriptive name |
| **Host** | `maestro-ipc1.aexp.com` | Trino coordinator endpoint |
| **Port** | `443` | HTTPS port |
| **SSL** | ✅ **Enabled** | Required for secure connection |
| **Catalog** | `hive` / `iceberg` / etc. | Based on your Trino catalogs |
| **Schema** | (optional) | Default schema if needed |
| **Authentication** | As per Trino config | e.g., LDAP, Kerberos, etc. |

### Step 4: Configure Driver Properties

Ensure the following SSL properties are set:

```properties
SSL=true
```

**Important**: 
- ✅ Do **NOT** disable SSL verification
- ✅ Truststore is configured at JVM level (no additional SSL config needed)
- ✅ Certificate validation will use the mounted `amex-truststore.jks`

### Step 5: Test Connection

1. Click **Test Connection** button
2. Expected result: ✅ **Connected**

If you encounter errors, see [Troubleshooting](#troubleshooting) section.

### Step 6: Save Connection

1. Click **Save** to create the connection
2. The connection will appear in the connections list

### Step 7: Make Connection Shared (Optional)

To make the connection available to all users:

1. Select the newly created connection
2. Go to the **Access** tab
3. Enable:
   - ✅ **Shared connection**
   - ✅ **Available for all users**
4. Click **Save**

---

## Custom Docker Image

### Image Details

The CloudBeaver deployment uses a custom Docker image that includes:

- **Base Image**: `dbeaver/cloudbeaver:25.2.5` (or your specific version)
- **Trino JDBC Driver**: Version **479** (matches Trino cluster version)
- **Truststore**: Mounted via Kubernetes secret from HashiCorp Vault

### Build Reference

Example Dockerfile structure:

```dockerfile
FROM dbeaver/cloudbeaver:25.2.5

# Add Trino JDBC driver v479
COPY trino-jdbc-479.jar /opt/cloudbeaver/drivers/

# Additional configurations if needed
# ...
```

### Why Custom Image?

- ✅ Ensures JDBC driver version matches Trino server version (479)
- ✅ Prevents driver compatibility issues
- ✅ Simplifies deployment (driver bundled in image)

---

## Certificate Management

### HashiCorp Vault Integration

All certificates and truststore files are managed through HashiCorp Vault:

```
HashiCorp Vault
    ↓
Kubernetes Secret
    ↓
Pod Volume Mount
    ↓
/etc/truststore/configfiles/output/amex-truststore.jks
```

### Benefits

- ✅ **Centralized Management**: All certificates in one place
- ✅ **Automated Rotation**: Vault handles certificate lifecycle
- ✅ **Secure Injection**: Secrets injected at runtime, not in image
- ✅ **No Hardcoding**: No certificates embedded in Docker images

### Certificate Chain

Ensure the truststore contains:
- Root CA certificate
- Intermediate CA certificates
- Certificate for `maestro-ipc1.aexp.com`

---

## Troubleshooting

### Issue 1: PKIX Path Building Failed

**Error Message**:
```
sun.security.validator.ValidatorException: PKIX path building failed
```

**Cause**: Missing CA certificate in truststore

**Solution**:

1. List truststore contents:
```bash
keytool -list -keystore /etc/truststore/configfiles/output/amex-truststore.jks
```

2. Verify CA for `maestro-ipc1.aexp.com` is present

3. If missing, add certificate to Vault and redeploy

### Issue 2: SSLHandshakeException

**Error Message**:
```
javax.net.ssl.SSLHandshakeException: Received fatal alert: handshake_failure
```

**Cause**: SSL/TLS configuration mismatch

**Solution**:

1. Test SSL connection manually:
```bash
openssl s_client -connect maestro-ipc1.aexp.com:443 -servername maestro-ipc1.aexp.com
```

2. Verify certificate chain is complete

3. Check Trino server SSL configuration

### Issue 3: Driver Version Mismatch

**Error Message**:
```
Incompatible Trino version
```

**Cause**: JDBC driver version doesn't match Trino server version

**Solution**:

1. Verify Trino server version:
```bash
curl -k https://maestro-ipc1.aexp.com:443/v1/info
```

2. Ensure custom image includes matching JDBC driver (v479)

3. Rebuild image if necessary

### Issue 4: Connection Timeout

**Error Message**:
```
Connection timeout
```

**Cause**: Network connectivity issue

**Solution**:

1. Test connectivity from CloudBeaver pod:
```bash
kubectl exec -it <cloudbeaver-pod> -n <namespace> -- curl -k https://maestro-ipc1.aexp.com:443
```

2. Check network policies and firewall rules

3. Verify DNS resolution

### Issue 5: Authentication Failed

**Error Message**:
```
Authentication failed
```

**Cause**: Invalid credentials or authentication method

**Solution**:

1. Verify authentication method matches Trino configuration
2. Check user credentials
3. Review Trino access control settings

---

## Security Notes

### Best Practices

⚠️ **DO NOT**:
- ❌ Disable SSL verification
- ❌ Embed truststore passwords in Helm values
- ❌ Hardcode certificates in Docker images
- ❌ Share admin credentials

✅ **DO**:
- ✅ Always use SSL/TLS for Trino connections
- ✅ Retrieve secrets from Kubernetes or Vault
- ✅ Restrict admin credentials to authorized personnel
- ✅ Use RBAC for CloudBeaver user access
- ✅ Regularly rotate certificates via Vault
- ✅ Monitor connection logs for security events

### Secret Management

All sensitive data should be stored in:
- **Kubernetes Secrets**: For runtime credentials
- **HashiCorp Vault**: For certificates and truststore
- **Never in**: Git repositories, Helm values, or Docker images

---

## Summary

### Quick Reference

| Component | Value |
|-----------|-------|
| **CloudBeaver URL** | https://telecode-ipc1.aexp.com |
| **Trino Host** | maestro-ipc1.aexp.com |
| **Trino Port** | 443 |
| **SSL** | ✅ Enabled |
| **JDBC Version** | 479 |
| **Truststore** | Provided via HashiCorp Vault |
| **Admin User** | cadmin |
| **Admin Secret** | hvm-secret-files |
| **Deployment** | Helm Chart |

### Connection Flow

```
User → CloudBeaver (telecode-ipc1.aexp.com)
         ↓
    Trino JDBC Driver (v479)
         ↓
    SSL/TLS (amex-truststore.jks)
         ↓
    Trino Cluster (maestro-ipc1.aexp.com:443)
```

---

## Additional Resources

- [CloudBeaver Documentation](https://cloudbeaver.io/docs/)
- [Trino Documentation](https://trino.io/docs/current/)
- [HashiCorp Vault Documentation](https://www.vaultproject.io/docs)
- [Kubernetes Secrets Management](https://kubernetes.io/docs/concepts/configuration/secret/)

---

## Support

For issues or questions:
1. Check [Troubleshooting](#troubleshooting) section
2. Review CloudBeaver logs: `kubectl logs <cloudbeaver-pod> -n <namespace>`
3. Contact your platform team

---

**Document Version**: 1.0  
**Last Updated**: 2026-02-25  
**Maintained By**: Platform Engineering Team