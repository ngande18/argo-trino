# Trino SQL Server Connector with Kerberos Authentication

Complete guide for configuring Trino to connect to SQL Server using Kerberos authentication with keytabs managed by HashiCorp Vault.

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Understanding JAAS](#understanding-jaas)
- [Configuration](#configuration)
  - [1. Helm Values Configuration](#1-helm-values-configuration)
  - [2. Catalog Files](#2-catalog-files)
  - [3. Secrets Management](#3-secrets-management)
- [Deployment](#deployment)
- [Validation](#validation)
- [Troubleshooting](#troubleshooting)
- [Rotation and Maintenance](#rotation-and-maintenance)
- [DBA/AD Team Checklist](#dbaad-team-checklist)

---

## Overview

This setup enables Trino to authenticate to SQL Server using Kerberos with keytabs, providing:

- **Per-catalog authentication**: Each SQL Server connector uses its own keytab and principal
- **Secure credential management**: Keytabs stored in Kubernetes Secrets (sealed/encrypted in GitOps)
- **No password storage**: Kerberos eliminates need for SQL passwords in configuration
- **Enterprise-grade security**: Leverages Active Directory for authentication

### Authentication Flow

```
Trino → JDBC Driver → Java → JAAS → Keytab → Active Directory → SQL Server
```

---

## Architecture

### Components

1. **JAAS (Java Authentication and Authorization Service)**: Java security framework that tells Java how to authenticate
2. **Kerberos Configuration (`krb5.conf`)**: Defines realm, KDC lookup settings
3. **JAAS Configuration (`login.conf`)**: Maps catalog names to keytabs and principals
4. **Keytabs**: Binary files containing encrypted Kerberos credentials
5. **Catalog Properties**: Trino connector configuration with JDBC URLs

### File Locations in Pods

```
/etc/trino/
├── login.conf                          # JAAS configuration (from additionalConfigFiles)
├── krb5.conf                           # Kerberos configuration (from additionalConfigFiles)
└── krb/                                # Keytab directory (from secretMounts)
    ├── wgpvissqlha40/
    │   └── svc.keytab                  # Keytab for wgpvissqlha40 catalog
    └── wgpvissqlha600/
        └── svc.keytab                  # Keytab for wgpvissqlha600 catalog
```

---

## Prerequisites

### Active Directory / SQL Server Side

- [ ] Service Principal Names (SPNs) registered for each SQL Server instance
- [ ] Service accounts created with appropriate permissions
- [ ] Keytabs generated for each service account
- [ ] SQL Server instances joined to AD domain
- [ ] Domain trust configured (if multi-domain)

### Kubernetes Cluster

- [ ] Trino Helm chart deployed
- [ ] Namespace created (e.g., `cdai01trino-e3h`)
- [ ] Network connectivity to AD KDCs and SQL Server instances
- [ ] DNS resolution working for domain names
- [ ] Secret management solution (SealedSecrets/ExternalSecrets/Vault)

### Tools Required

- `kubectl` CLI
- `helm` CLI
- `kubeseal` (if using SealedSecrets)
- Kerberos tools (`kinit`, `klist`) for validation

---

## Understanding JAAS

### What is JAAS?

**JAAS = Java Authentication and Authorization Service**

A Java security framework used to:
- **Authenticate** a user or service (prove identity)
- **Authorize** what that identity is allowed to do

### Why JAAS Matters for Trino + SQL Server + Kerberos

When you use `authenticationScheme=JavaKerberos`, the SQL Server JDBC driver asks Java:
> "Give me Kerberos credentials."

Java looks at JAAS config (`login.conf`) to know:
- Which keytab to use
- Which principal to use
- Whether to use ticket cache
- Whether to prompt for password

**Without JAAS, Java does not know how to authenticate using Kerberos.**

### JAAS Configuration Block Example

```java
wgpvissqlha40 {
    com.sun.security.auth.module.Krb5LoginModule required
    useKeyTab=true
    keyTab="/etc/trino/krb/wgpvissqlha40/svc.keytab"
    principal="svc.sql@ADS.AEXP.COM"
    storeKey=true
    useTicketCache=false
    debug=false;
};
```

This tells Java:
> "For login named `wgpvissqlha40`, use Kerberos with this keytab and principal."

### When You Need JAAS

| Authentication Type | Need JAAS? |
|---------------------|------------|
| SQL username/password | ❌ No |
| NTLM_PASSWORD | ❌ No |
| Kerberos with keytab | ✅ **Yes** |

---

## Configuration

### 1. Helm Values Configuration

Add the following to your `values.yaml` file. This configuration:
- Defines JAAS and Kerberos configuration files
- Mounts keytab secrets
- Sets JVM flags to enable Kerberos

```yaml
#################### Kerberos JAAS + krb5 config ####################
# Non-secret config stored in Git (login.conf + krb5.conf)
additionalConfigFiles:
  login.conf: |-
    # JAAS entries (one block per catalog/connector)
    # Names are used by jaasConfigurationName in catalog properties
    wgpvissqlha40 {
      com.sun.security.auth.module.Krb5LoginModule required
      useKeyTab=true
      keyTab="/etc/trino/krb/wgpvissqlha40/svc.keytab"
      principal="svc.sql-wgpvissqlha40@ADS.AEXP.COM"
      storeKey=true
      useTicketCache=false
      debug=false;
    };

    wgpvissqlha600 {
      com.sun.security.auth.module.Krb5LoginModule required
      useKeyTab=true
      keyTab="/etc/trino/krb/wgpvissqlha600/svc.keytab"
      principal="svc.sql-starburste3h@ADS.AEXP.COM"
      storeKey=true
      useTicketCache=false
      debug=false;
    };

  krb5.conf: |-
    [libdefaults]
      default_realm = ADS.AEXP.COM
      dns_lookup_kdc = true
      dns_lookup_realm = true

    [domain_realm]
      .gso.aexp.com = ADS.AEXP.COM
      gso.aexp.com = ADS.AEXP.COM

#################### Coordinator - mount keytabs + JVM flags ####################
coordinator:
  # Mount sealed/encrypted Secrets here (Secrets are created separately)
  secretMounts:
    - name: trino-krb-wgpvissqlha40
      secretName: trino-krb-wgpvissqlha40
      path: /etc/trino/krb/wgpvissqlha40
    - name: trino-krb-wgpvissqlha600
      secretName: trino-krb-wgpvissqlha600
      path: /etc/trino/krb/wgpvissqlha600

  # JVM flags used by coordinator JVM
  additionalJVMConfig:
    - "-Djava.security.krb5.conf=/etc/trino/krb5.conf"
    - "-Djava.security.auth.login.config=/etc/trino/login.conf"
    - "-Djavax.security.auth.useSubjectCredsOnly=false"
    # Optional: enable Kerberos debug temporarily if troubleshooting
    # - "-Dsun.security.krb5.debug=true"

#################### Worker - same mounts + JVM flags (important) ####################
worker:
  secretMounts:
    - name: trino-krb-wgpvissqlha40
      secretName: trino-krb-wgpvissqlha40
      path: /etc/trino/krb/wgpvissqlha40
    - name: trino-krb-wgpvissqlha600
      secretName: trino-krb-wgpvissqlha600
      path: /etc/trino/krb/wgpvissqlha600

  additionalJVMConfig:
    - "-Djava.security.krb5.conf=/etc/trino/krb5.conf"
    - "-Djava.security.auth.login.config=/etc/trino/login.conf"
    - "-Djavax.security.auth.useSubjectCredsOnly=false"
```

#### Important Notes

- ✅ `login.conf` and `krb5.conf` live under `/etc/trino/` (via `additionalConfigFiles`)
- ✅ JVM flags point to `/etc/trino/login.conf` and `/etc/trino/krb5.conf`
- ✅ `keyTab` lines in `login.conf` reference secret-mounted paths
- ❌ **Do not commit keytab files to Git** - use SealedSecrets/ExternalSecrets/Vault
- ✅ Both coordinator and worker must have identical configuration

---

### 2. Catalog Files

Create catalog property files and commit them to your chart repo under `catalog/` directory.

#### Example: `catalog/wgpvissqlha40_dbdba.properties`

```properties
connector.name=sqlserver
connection-url=jdbc:sqlserver://wgpvissqlha40.gso.aexp.com:1433;databaseName=dbdba;encrypt=true;trustServerCertificate=false;authenticationScheme=JavaKerberos;integratedSecurity=true;jaasConfigurationName=wgpvissqlha40
sqlserver.stored-procedure-table-function-enabled=true
```

#### Example: `catalog/wgpvissqlha600_dbdba.properties`

```properties
connector.name=sqlserver
connection-url=jdbc:sqlserver://wgpvissqlha600.gso.aexp.com:1433;databaseName=dbdba;encrypt=true;trustServerCertificate=false;authenticationScheme=JavaKerberos;integratedSecurity=true;jaasConfigurationName=wgpvissqlha600
sqlserver.stored-procedure-table-function-enabled=true
```

#### Key Points

- ✅ Keep `connection-url` on **one line**
- ✅ `jaasConfigurationName` must **exactly match** the block name in `login.conf`
- ✅ Include `authenticationScheme=JavaKerberos` and `integratedSecurity=true`
- ❌ **Do not** include `connection-user` or `connection-password` (Kerberos uses keytab only)

---

### 3. Secrets Management

Keytabs are sensitive credentials and must be stored securely.

#### Option A: Imperative Secret Creation (for testing)

```bash
NS=cdai01trino-e3h

# Create secret for wgpvissqlha40 keytab
kubectl -n $NS create secret generic trino-krb-wgpvissqlha40 \
  --from-file=svc.keytab=/path/to/wgpvissqlha40.svc.keytab

# Create secret for wgpvissqlha600 keytab
kubectl -n $NS create secret generic trino-krb-wgpvissqlha600 \
  --from-file=svc.keytab=/path/to/wgpvissqlha600.svc.keytab
```

#### Option B: GitOps with SealedSecrets (recommended)

1. **Create plain secret YAML** (locally, not committed):
   ```bash
   kubectl create secret generic trino-krb-wgpvissqlha40 \
     --from-file=svc.keytab=/path/to/wgpvissqlha40.svc.keytab \
     --dry-run=client -o yaml > secret.yaml
   ```

2. **Seal the secret**:
   ```bash
   kubeseal < secret.yaml > sealed-secret.yaml
   ```

3. **Commit sealed secret** to GitOps repo:
   ```bash
   git add sealed-secret.yaml
   git commit -m "Add sealed keytab secret for wgpvissqlha40"
   git push
   ```

4. **Controller unseals** and creates real Secret in cluster

#### Option C: External Secrets Operator

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: trino-krb-wgpvissqlha40
  namespace: cdai01trino-e3h
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: trino-krb-wgpvissqlha40
    creationPolicy: Owner
  data:
    - secretKey: svc.keytab
      remoteRef:
        key: secret/data/trino/keytabs/wgpvissqlha40
        property: keytab
```

#### Security Best Practices

- ✅ Set `defaultMode: 0400` on secret volumes (read-only for owner)
- ✅ Ensure Trino pod `runAsUser` matches keytab file ownership
- ✅ Use RBAC to restrict Secret access
- ✅ Rotate keytabs regularly (see [Rotation](#rotation-and-maintenance))

---

## Deployment

### 1. Apply Helm Chart

```bash
# Using Helm directly
helm upgrade --install trino ./chart \
  -n cdai01trino-e3h \
  -f values.yaml

# Or let GitOps controller sync (ArgoCD, Flux, etc.)
```

### 2. Wait for Rollout

```bash
kubectl rollout status deployment/trino-coordinator -n cdai01trino-e3h
kubectl rollout status deployment/trino-worker -n cdai01trino-e3h
```

### 3. Verify Pods Running

```bash
kubectl get pods -n cdai01trino-e3h -l app.kubernetes.io/name=trino
```

---

## Validation

Follow these steps to validate your Kerberos configuration.

### Step 1: Verify Files and Mounts

Check that configuration files and keytabs are mounted correctly.

```bash
# Get coordinator pod name
COORD=$(kubectl get pod -n cdai01trino-e3h \
  -l app.kubernetes.io/component=coordinator \
  -o name | head -n1)

# Check files in coordinator
kubectl exec -n cdai01trino-e3h -it $COORD -- /bin/sh -c \
  "ls -l /etc/trino /etc/trino/krb; cat /etc/trino/login.conf | head -n 50"

# Get worker pod name
WORKER=$(kubectl get pod -n cdai01trino-e3h \
  -l app.kubernetes.io/component=worker \
  -o name | head -n1)

# Check files in worker
kubectl exec -n cdai01trino-e3h -it $WORKER -- /bin/sh -c \
  "ls -l /etc/trino /etc/trino/krb; cat /etc/trino/login.conf | head -n 50"
```

**Expected output:**
```
/etc/trino:
-rw-r--r-- 1 trino trino  xxx login.conf
-rw-r--r-- 1 trino trino  xxx krb5.conf

/etc/trino/krb:
drwxr-xr-x 2 trino trino wgpvissqlha40
drwxr-xr-x 2 trino trino wgpvissqlha600

/etc/trino/krb/wgpvissqlha40:
-r-------- 1 trino trino xxx svc.keytab
```

---

### Step 2: Inspect Keytab Principals

Verify the keytab contains the correct principal.

```bash
# Inside coordinator pod (if klist available)
kubectl exec -n cdai01trino-e3h -it $COORD -- \
  klist -k /etc/trino/krb/wgpvissqlha40/svc.keytab

kubectl exec -n cdai01trino-e3h -it $COORD -- \
  klist -k /etc/trino/krb/wgpvissqlha600/svc.keytab
```

**Expected output:**
```
Keytab name: FILE:/etc/trino/krb/wgpvissqlha40/svc.keytab
KVNO Principal
---- --------------------------------------------------------------------------
   2 svc.sql-wgpvissqlha40@ADS.AEXP.COM
```

---

### Step 3: Test Kerberos Authentication (Optional)

If `kinit` is available in the container:

```bash
kubectl exec -n cdai01trino-e3h -it $COORD -- /bin/sh -c "
  kinit -k -t /etc/trino/krb/wgpvissqlha40/svc.keytab svc.sql-wgpvissqlha40@ADS.AEXP.COM
  klist
  kdestroy
"
```

**Expected output:**
```
Ticket cache: FILE:/tmp/krb5cc_1000
Default principal: svc.sql-wgpvissqlha40@ADS.AEXP.COM

Valid starting     Expires            Service principal
03/13/26 01:00:00  03/13/26 11:00:00  krbtgt/ADS.AEXP.COM@ADS.AEXP.COM
```

---

### Step 4: Run Trino Query

Execute a test query using the SQL Server connector.

```bash
# Using Trino CLI (if available in image)
kubectl exec -n cdai01trino-e3h -it $COORD -- \
  trino --server https://localhost:8443 \
        --user <http_user> \
        --catalog wgpvissqlha40_dbdba \
        --schema dbo \
        --execute 'SELECT 1 AS test'

# Or using curl
kubectl exec -n cdai01trino-e3h -it $COORD -- \
  curl -k -u '<http_user>:<http_pass>' \
       -X POST \
       --data 'SELECT 1' \
       https://localhost:8443/v1/statement -i
```

**Expected output:**
```
 test
------
    1
(1 row)
```

---

### Step 5: Check Trino Logs

Look for Kerberos-related log entries.

```bash
kubectl logs -n cdai01trino-e3h $COORD --since=5m | \
  grep -i -E 'kerberos|GSS|Login failed|ClientConnectionId|auth_scheme' -A4 -B4
```

**Good signs:**
- No `Login failed` errors
- No fallback to NTLM or SQL authentication
- Successful GSS context establishment

**Bad signs:**
- `Login failed. The login is from an untrusted domain`
- `Cannot find key of appropriate type to decrypt AP REP`
- `Server not found in Kerberos database`

---

### Step 6: Verify on SQL Server (Authoritative Check)

Ask your DBA to run this query on the SQL Server instance immediately after your Trino test:

```sql
SELECT 
    session_id,
    auth_scheme,
    login_name,
    client_net_address,
    connect_time
FROM sys.dm_exec_connections
WHERE connect_time > DATEADD(MINUTE, -10, GETDATE())
ORDER BY connect_time DESC;
```

**Expected result:**
- `auth_scheme` = `KERBEROS`
- `login_name` matches the account mapped to the SPN

**If you see:**
- `auth_scheme` = `NTLM` → Kerberos failed, driver fell back
- `auth_scheme` = `SQL` → Using SQL authentication (wrong config)

---

## Troubleshooting

### Common Issues and Solutions

#### Issue 1: Login Failed - Untrusted Domain

**Error:**
```
Login failed. The login is from an untrusted domain and cannot be used with Integrated authentication.
```

**Causes:**
- SQL Server not joined to AD domain
- Domain trust not configured between domains
- SQL Server service account not in correct domain

**Solution:**
- Verify SQL Server is domain-joined: `systeminfo | findstr /B /C:"Domain"`
- Check domain trust with AD team
- Ensure SQL Server service account is in the correct domain

---

#### Issue 2: Cannot Find Key to Decrypt

**Error:**
```
Cannot find key of appropriate type to decrypt AP REP - AES256 CTS mode with HMAC SHA1-96
```

**Causes:**
- Keytab encryption type mismatch
- Keytab not generated correctly
- Principal name mismatch

**Solution:**
- Regenerate keytab with correct encryption types
- Verify principal name matches exactly: `klist -k /path/to/keytab`
- Check SQL Server supports AES encryption (Windows Server 2008+)

---

#### Issue 3: Server Not Found in Kerberos Database

**Error:**
```
Server not found in Kerberos database (7)
```

**Causes:**
- SPN not registered in Active Directory
- SPN registered to wrong account
- DNS resolution issues

**Solution:**
- Verify SPN registration:
  ```powershell
  setspn -L ADS\svc.sql-starburste3h
  ```
- Register SPN if missing:
  ```powershell
  setspn -S MSSQLSvc/wgpvissqlha40.gso.aexp.com:1433 ADS\svc.sql-starburste3h
  ```
- Test DNS resolution from Trino pods:
  ```bash
  kubectl exec -n cdai01trino-e3h -it $COORD -- nslookup wgpvissqlha40.gso.aexp.com
  ```

---

#### Issue 4: Driver Falls Back to SQL Authentication

**Symptoms:**
- Query works but SQL Server shows `auth_scheme = SQL` or `NTLM`
- No Kerberos errors in logs

**Causes:**
- `connection-user` and `connection-password` present in catalog properties
- `authenticationScheme` not set to `JavaKerberos`
- `jaasConfigurationName` missing or incorrect

**Solution:**
- Remove `connection-user` and `connection-password` from catalog properties
- Verify catalog properties:
  ```properties
  authenticationScheme=JavaKerberos
  integratedSecurity=true
  jaasConfigurationName=wgpvissqlha40
  ```
- Ensure `jaasConfigurationName` matches block name in `login.conf`

---

#### Issue 5: Keytab File Not Found

**Error:**
```
java.io.FileNotFoundException: /etc/trino/krb/wgpvissqlha40/svc.keytab (No such file or directory)
```

**Causes:**
- Secret not created or not mounted
- Path mismatch between `login.conf` and `secretMounts`
- Secret name mismatch

**Solution:**
- Verify secret exists:
  ```bash
  kubectl get secret -n cdai01trino-e3h trino-krb-wgpvissqlha40
  ```
- Check secret is mounted:
  ```bash
  kubectl exec -n cdai01trino-e3h -it $COORD -- ls -l /etc/trino/krb/wgpvissqlha40/
  ```
- Verify paths match in `values.yaml`:
  - `secretMounts[].path` = `/etc/trino/krb/wgpvissqlha40`
  - `login.conf` `keyTab` = `/etc/trino/krb/wgpvissqlha40/svc.keytab`

---

#### Issue 6: KDC Unreachable

**Error:**
```
Cannot contact any KDC for realm 'ADS.AEXP.COM'
```

**Causes:**
- Network connectivity issues
- DNS not resolving KDC
- Firewall blocking Kerberos ports (88, 464)

**Solution:**
- Test DNS from pod:
  ```bash
  kubectl exec -n cdai01trino-e3h -it $COORD -- nslookup _kerberos._tcp.ads.aexp.com
  ```
- Test KDC connectivity:
  ```bash
  kubectl exec -n cdai01trino-e3h -it $COORD -- nc -zv <kdc-hostname> 88
  ```
- Check firewall rules with network team

---

### Enable Debug Logging

For detailed Kerberos troubleshooting, enable debug logging:

1. **Update `values.yaml`:**
   ```yaml
   coordinator:
     additionalJVMConfig:
       - "-Dsun.security.krb5.debug=true"
   
   worker:
     additionalJVMConfig:
       - "-Dsun.security.krb5.debug=true"
   ```

2. **Update `login.conf`:**
   ```java
   wgpvissqlha40 {
     com.sun.security.auth.module.Krb5LoginModule required
     useKeyTab=true
     keyTab="/etc/trino/krb/wgpvissqlha40/svc.keytab"
     principal="svc.sql-wgpvissqlha40@ADS.AEXP.COM"
     storeKey=true
     useTicketCache=false
     debug=true;  # Enable debug
   };
   ```

3. **Redeploy and check logs:**
   ```bash
   kubectl logs -n cdai01trino-e3h $COORD --since=5m | grep -i krb5
   ```

**Remember to disable debug logging after troubleshooting** (it's very verbose).

---

## Rotation and Maintenance

### Keytab Rotation

When keytabs need to be rotated (e.g., password change, security policy):

1. **Obtain new keytab** from AD team

2. **Update Secret:**
   ```bash
   # Imperative
   kubectl create secret generic trino-krb-wgpvissqlha40 \
     --from-file=svc.keytab=/path/to/new-keytab.keytab \
     --dry-run=client -o yaml | kubectl apply -f -
   
   # Or update sealed secret in GitOps repo
   ```

3. **Restart Trino pods:**
   ```bash
   kubectl rollout restart deployment/trino-coordinator -n cdai01trino-e3h
   kubectl rollout restart deployment/trino-worker -n cdai01trino-e3h
   ```

4. **Validate** using steps in [Validation](#validation) section

**Note:** If the principal name remains the same, no changes to `login.conf` are needed.

---

### Adding a New SQL Server Connector

To add a new SQL Server instance with Kerberos authentication:

1. **Update `values.yaml`** - add JAAS block:
   ```yaml
   additionalConfigFiles:
     login.conf: |-
       # ... existing blocks ...
       
       new_sqlserver {
         com.sun.security.auth.module.Krb5LoginModule required
         useKeyTab=true
         keyTab="/etc/trino/krb/new_sqlserver/svc.keytab"
         principal="svc.sql-newsql@ADS.AEXP.COM"
         storeKey=true
         useTicketCache=false
         debug=false;
       };
   ```

2. **Update secret mounts** in `values.yaml`:
   ```yaml
   coordinator:
     secretMounts:
       # ... existing mounts ...
       - name: trino-krb-new-sqlserver
         secretName: trino-krb-new-sqlserver
         path: /etc/trino/krb/new_sqlserver
   
   worker:
     secretMounts:
       # ... existing mounts ...
       - name: trino-krb-new-sqlserver
         secretName: trino-krb-new-sqlserver
         path: /etc/trino/krb/new_sqlserver
   ```

3. **Create catalog file** `catalog/new_sqlserver_dbname.properties`:
   ```properties
   connector.name=sqlserver
   connection-url=jdbc:sqlserver://newsql.gso.aexp.com:1433;databaseName=dbname;encrypt=true;trustServerCertificate=false;authenticationScheme=JavaKerberos;integratedSecurity=true;jaasConfigurationName=new_sqlserver
   ```

4. **Create Secret** with new keytab (sealed/encrypted)

5. **Commit changes** to GitOps repo

6. **Sync/Deploy** and validate

---

### Monitoring

Set up monitoring for Kerberos authentication:

1. **Metrics to track:**
   - Failed authentication attempts
   - Ticket expiration warnings
   - Connection failures by catalog

2. **Log aggregation:**
   - Collect Trino logs with Kerberos keywords
   - Alert on `Login failed` patterns

3. **SQL Server monitoring:**
   - Track `auth_scheme` distribution
   - Alert on unexpected NTLM/SQL authentication

---

## DBA/AD Team Checklist

Provide this checklist to your DBA and Active Directory teams:

### Active Directory Team

- [ ] **Service accounts created:**
  - `ADS\svc.sql-wgpvissqlha40`
  - `ADS\svc.sql-starburste3h`
  - (Add others as needed)

- [ ] **SPNs registered** for each SQL Server instance:
  ```powershell
  # Verify existing SPNs
  setspn -L ADS\svc.sql-starburste3h
  
  # Register SPN if missing
  setspn -S MSSQLSvc/wgpvissqlha40.gso.aexp.com:1433 ADS\svc.sql-starburste3h
  setspn -S MSSQLSvc/wgpvissqlha600.gso.aexp.com:1433 ADS\svc.sql-starburste3h
  ```

- [ ] **Keytabs generated** for each service account:
  ```powershell
  ktpass -princ MSSQLSvc/wgpvissqlha40.gso.aexp.com:1433@ADS.AEXP.COM \
         -mapuser ADS\svc.sql-starburste3h \
         -pass <password> \
         -out wgpvissqlha40.keytab \
         -ptype KRB5_NT_PRINCIPAL \
         -crypto AES256-SHA1
  ```

- [ ] **Domain trust configured** (if multi-domain environment)

- [ ] **Encryption types enabled:**
  - AES256-CTS-HMAC-SHA1-96
  - AES128-CTS-HMAC-SHA1-96

---

### SQL Server DBA Team

- [ ] **SQL Server instances domain-joined:**
  ```sql
  -- Verify domain membership
  SELECT SERVERPROPERTY('ComputerNamePhysicalNetBIOS') AS ServerName,
         SERVERPROPERTY('MachineName') AS MachineName
  ```

- [ ] **SQL Server service running under correct account:**
  - Check SQL Server Configuration Manager
  - Service account should match SPN registration

- [ ] **Kerberos authentication enabled:**
  ```sql
  -- Check authentication mode
  SELECT SERVERPROPERTY('IsIntegratedSecurityOnly') AS WindowsAuthOnly
  ```

- [ ] **Firewall rules allow Kerberos:**
  - Port 88 (Kerberos)
  - Port 464 (Kerberos password change)
  - Port 1433 (SQL Server)

- [ ] **Test Kerberos authentication:**
  ```sql
  -- After Trino connection attempt, verify auth scheme
  SELECT session_id, auth_scheme, login_name, client_net_address
  FROM sys.dm_exec_connections
  WHERE connect_time > DATEADD(MINUTE, -10, GETDATE())
  ORDER BY connect_time DESC;
  ```

- [ ] **Grant necessary permissions** to service accounts:
  ```sql
  -- Example: grant read access
  USE [dbdba];
  CREATE LOGIN [ADS\svc.sql-starburste3h] FROM WINDOWS;
  CREATE USER [ADS\svc.sql-starburste3h] FOR LOGIN [ADS\svc.sql-starburste3h];
  ALTER ROLE db_datareader ADD MEMBER [ADS\svc.sql-starburste3h];
  ```

---

### Network Team

- [ ] **DNS resolution working:**
  - Forward lookup: `wgpvissqlha40.gso.aexp.com` → IP
  - Reverse lookup: IP → `wgpvissqlha40.gso.aexp.com`
  - SRV records: `_kerberos._tcp.ads.aexp.com`

- [ ] **Network connectivity:**
  - Trino pods → SQL Server instances (port 1433)
  - Trino pods → AD KDCs (port 88, 464)

- [ ] **Firewall rules:**
  - Allow Kerberos traffic (UDP/TCP 88, 464)
  - Allow SQL Server traffic (TCP 1433)

---

## Reference

### Useful Commands

```bash
# Get pod names
kubectl get pods -n cdai01trino-e3h -l app.kubernetes.io/name=trino

# Check secret
kubectl get secret -n cdai01trino-e3h trino-krb-wgpvissqlha40 -o yaml

# Describe secret mount
kubectl describe pod -n cdai01trino-e3h <pod-name> | grep -A10 "Mounts:"

# View logs
kubectl logs -n cdai01trino-e3h <pod-name> --since=10m

# Exec into pod
kubectl exec -n cdai01trino-e3h -it <pod-name> -- /bin/bash

# Test DNS from pod
kubectl exec -n cdai01trino-e3h -it <pod-name> -- nslookup wgpvissqlha40.gso.aexp.com

# Test port connectivity
kubectl exec -n cdai01trino-e3h -it <pod-name> -- nc -zv wgpvissqlha40.gso.aexp.com 1433
```

---

### JDBC URL Parameters Reference

| Parameter | Value | Purpose |
|-----------|-------|---------|
| `authenticationScheme` | `JavaKerberos` | Use Kerberos authentication |
| `integratedSecurity` | `true` | Use Windows integrated security |
| `jaasConfigurationName` | `<block-name>` | JAAS config block to use |
| `encrypt` | `true` | Encrypt connection |
| `trustServerCertificate` | `false` | Validate server certificate |
| `databaseName` | `<db-name>` | Target database |

---

### File Permissions

Recommended permissions for keytab files:

```bash
# Keytab should be readable only by Trino user
chmod 400 /etc/trino/krb/*/svc.keytab
chown trino:trino /etc/trino/krb/*/svc.keytab
```

In Kubernetes, set via Secret volume mount:
```yaml
secretMounts:
  - name: trino-krb-wgpvissqlha40
    secretName: trino-krb-wgpvissqlha40
    path: /etc/trino/krb/wgpvissqlha40
    defaultMode: 0400  # Read-only for owner
```

---

## Additional Resources

- [Trino SQL Server Connector Documentation](https://trino.io/docs/current/connector/sqlserver.html)
- [Microsoft SQL Server Kerberos Configuration](https://docs.microsoft.com/en-us/sql/database-engine/configure-windows/register-a-service-principal-name-for-kerberos-connections)
- [Java Kerberos Requirements](https://docs.oracle.com/javase/8/docs/technotes/guides/security/jgss/tutorials/KerberosReq.html)
- [JAAS Configuration Guide](https://docs.oracle.com/javase/8/docs/technotes/guides/security/jgss/tutorials/LoginConfigFile.html)

---

## Support

For issues or questions:

1. Check [Troubleshooting](#troubleshooting) section
2. Review Trino logs with debug enabled
3. Verify with DBA using SQL Server DMV queries
4. Contact platform team with:
   - Trino pod logs (with debug enabled)
   - SQL Server connection logs
   - Network connectivity test results

---

**Document Version:** 1.0  
**Last Updated:** 2026-03-13  
**Maintained By:** Platform Engineering Team
