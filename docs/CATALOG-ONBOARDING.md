# Trino Catalog Onboarding Guide

This guide explains how to onboard new catalogs to Trino deployments using Vault Secrets Operator.

## Overview

Each Trino deployment uses **three types of secrets** for catalog configuration:

1. **`{deployment}-secrets`** - Environment variables (key-value pairs)
2. **`{deployment}-certificates`** - TLS certificates and keys
3. **`{deployment}-configfiles`** - Configuration files

## Secret Types and Usage

### 1. Environment Variables Secret: `{deployment}-secrets`

**Purpose**: Store sensitive key-value pairs (passwords, tokens, connection strings)  
**Mounted as**: Environment variables in Trino pods  
**Vault Path**: `secret/data/{deployment}/secrets`

**Example Keys**:
- Database passwords
- API tokens
- Access keys
- Connection strings

### 2. Certificates Secret: `{deployment}-certificates`

**Purpose**: Store TLS certificates, private keys, and CA certificates  
**Mounted as**: Files in `/etc/trino/certificates/`  
**Vault Path**: `secret/data/{deployment}/certificates` or `pki/issue/{role}`

**Example Keys**:
- `tls.crt` - TLS certificate
- `tls.key` - Private key
- `ca.crt` - CA certificate
- `truststore.jks` - Java truststore
- `keystore.jks` - Java keystore

### 3. Config Files Secret: `{deployment}-configfiles`

**Purpose**: Store catalog configuration files and other config files  
**Mounted as**: Files in `/etc/trino/configfiles/`  
**Vault Path**: `secret/data/{deployment}/configfiles`

**Example Keys**:
- `mysql.properties` - MySQL catalog config
- `postgresql.properties` - PostgreSQL catalog config
- `hive.properties` - Hive catalog config
- `iceberg.properties` - Iceberg catalog config

## Configuration Example

### Vault Secrets Values File

Create `apps/vaules/vault-secrets/{deployment}.yaml`:

```yaml
vaultAuth:
  enabled: true
  name: trino-ipc1-maestro-vault-auth
  vaultConnectionRef: vault-connection
  kubernetes:
    role: trino-ipc1-maestro-role
    serviceAccount: trino
    audiences:
      - vault

vaultStaticSecrets:
  # 1. Environment Variables Secret
  - name: trino-ipc1-maestro-secrets
    enabled: true
    vaultAuthRef: trino-ipc1-maestro-vault-auth
    mount: secret
    path: trino-ipc1-maestro/secrets
    type: kv-v2
    refreshAfter: 30s
    hmacSecretData: true
    destination:
      name: trino-ipc1-maestro-secrets
      create: true
      type: Opaque
      labels:
        app: trino
        deployment: trino-ipc1-maestro
        secret-type: env-vars

  # 2. Certificates Secret
  - name: trino-ipc1-maestro-certificates
    enabled: true
    vaultAuthRef: trino-ipc1-maestro-vault-auth
    mount: pki
    path: pki/issue/trino-cert
    type: kv-v2
    refreshAfter: 24h
    destination:
      name: trino-ipc1-maestro-certificates
      create: true
      type: kubernetes.io/tls
      labels:
        app: trino
        deployment: trino-ipc1-maestro
        secret-type: certificates
      transformation:
        templates:
          tls.crt: |
            {{ get .Secrets "certificate" }}
          tls.key: |
            {{ get .Secrets "private_key" }}
          ca.crt: |
            {{ get .Secrets "issuing_ca" }}

  # 3. Config Files Secret
  - name: trino-ipc1-maestro-configfiles
    enabled: true
    vaultAuthRef: trino-ipc1-maestro-vault-auth
    mount: secret
    path: trino-ipc1-maestro/configfiles
    type: kv-v2
    refreshAfter: 60s
    hmacSecretData: true
    destination:
      name: trino-ipc1-maestro-configfiles
      create: true
      type: Opaque
      labels:
        app: trino
        deployment: trino-ipc1-maestro
        secret-type: config-files
```

## Onboarding New Catalogs

### Step 1: Store Secrets in Vault

#### A. Environment Variables (Passwords, Tokens)

```bash
# Store database credentials
vault kv put secret/trino-ipc1-maestro/secrets \
  mysql_password="SecurePassword123" \
  postgres_password="AnotherSecurePass" \
  s3_access_key="AKIAIOSFODNN7EXAMPLE" \
  s3_secret_key="wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
```

#### B. Certificates

```bash
# Store TLS certificates
vault kv put secret/trino-ipc1-maestro/certificates \
  tls.crt=@/path/to/cert.pem \
  tls.key=@/path/to/key.pem \
  ca.crt=@/path/to/ca.pem
```

Or use PKI engine for auto-generated certificates:
```bash
# Issue certificate from Vault PKI
vault write pki/issue/trino-cert \
  common_name="trino-ipc1-maestro.example.com" \
  ttl="720h"
```

#### C. Configuration Files

```bash
# Store catalog configuration files
vault kv put secret/trino-ipc1-maestro/configfiles \
  mysql.properties="$(cat <<EOF
connector.name=mysql
connection-url=jdbc:mysql://mysql-server:3306
connection-user=trino
connection-password=\${ENV:MYSQL_PASSWORD}
EOF
)" \
  postgresql.properties="$(cat <<EOF
connector.name=postgresql
connection-url=jdbc:postgresql://postgres-server:5432/database
connection-user=trino
connection-password=\${ENV:POSTGRES_PASSWORD}
EOF
)"
```

### Step 2: Configure Trino Helm Values

Update `apps/vaules/trino/{deployment}.yaml`:

```yaml
# Trino Helm values
coordinator:
  env:
    # Environment variables from secrets
    - name: MYSQL_PASSWORD
      valueFrom:
        secretKeyRef:
          name: trino-ipc1-maestro-secrets
          key: mysql_password
    - name: POSTGRES_PASSWORD
      valueFrom:
        secretKeyRef:
          name: trino-ipc1-maestro-secrets
          key: postgres_password
    - name: S3_ACCESS_KEY
      valueFrom:
        secretKeyRef:
          name: trino-ipc1-maestro-secrets
          key: s3_access_key
    - name: S3_SECRET_KEY
      valueFrom:
        secretKeyRef:
          name: trino-ipc1-maestro-secrets
          key: s3_secret_key

  volumeMounts:
    # Mount certificates
    - name: certificates
      mountPath: /etc/trino/certificates
      readOnly: true
    # Mount config files
    - name: configfiles
      mountPath: /etc/trino/configfiles
      readOnly: true

  volumes:
    # Certificates volume
    - name: certificates
      secret:
        secretName: trino-ipc1-maestro-certificates
    # Config files volume
    - name: configfiles
      secret:
        secretName: trino-ipc1-maestro-configfiles

# Apply same configuration to workers
worker:
  env: # Same as coordinator
  volumeMounts: # Same as coordinator
  volumes: # Same as coordinator

# Catalog configurations
additionalCatalogs:
  mysql: |
    connector.name=mysql
    connection-url=jdbc:mysql://mysql-server:3306
    connection-user=trino
    connection-password=${ENV:MYSQL_PASSWORD}
  
  postgresql: |
    connector.name=postgresql
    connection-url=jdbc:postgresql://postgres-server:5432/database
    connection-user=trino
    connection-password=${ENV:POSTGRES_PASSWORD}
  
  iceberg: |
    connector.name=iceberg
    iceberg.catalog.type=hive_metastore
    hive.metastore.uri=thrift://hive-metastore:9083
    hive.s3.aws-access-key=${ENV:S3_ACCESS_KEY}
    hive.s3.aws-secret-key=${ENV:S3_SECRET_KEY}
```

### Step 3: Reference Config Files

For catalogs that need external config files:

```yaml
additionalCatalogs:
  mysql: |
    # Reference the mounted config file
    include=/etc/trino/configfiles/mysql.properties
  
  postgresql: |
    include=/etc/trino/configfiles/postgresql.properties
```

## Secret Naming Convention

| Deployment | Secrets Secret | Certificates Secret | Config Files Secret |
|------------|----------------|---------------------|---------------------|
| trino-ipc1-maestro | `trino-ipc1-maestro-secrets` | `trino-ipc1-maestro-certificates` | `trino-ipc1-maestro-configfiles` |
| trino-ipc2-worker | `trino-ipc2-worker-secrets` | `trino-ipc2-worker-certificates` | `trino-ipc2-worker-configfiles` |
| cdaitrino-e3h | `cdaitrino-e3h-secrets` | `cdaitrino-e3h-certificates` | `cdaitrino-e3h-configfiles` |

## Vault Path Convention

| Secret Type | Vault Path Pattern |
|-------------|-------------------|
| Environment Variables | `secret/data/{deployment}/secrets` |
| Certificates (KV) | `secret/data/{deployment}/certificates` |
| Certificates (PKI) | `pki/issue/{role}` |
| Config Files | `secret/data/{deployment}/configfiles` |

## Common Catalog Examples

### MySQL Catalog

**Vault Secrets**:
```bash
vault kv put secret/trino-ipc1-maestro/secrets \
  mysql_password="SecurePassword"
```

**Catalog Config**:
```properties
connector.name=mysql
connection-url=jdbc:mysql://mysql-server:3306
connection-user=trino
connection-password=${ENV:MYSQL_PASSWORD}
```

### PostgreSQL with SSL

**Vault Secrets**:
```bash
# Store password
vault kv put secret/trino-ipc1-maestro/secrets \
  postgres_password="SecurePassword"

# Store SSL certificates
vault kv put secret/trino-ipc1-maestro/certificates \
  postgres-client.crt=@client.crt \
  postgres-client.key=@client.key \
  postgres-ca.crt=@ca.crt
```

**Catalog Config**:
```properties
connector.name=postgresql
connection-url=jdbc:postgresql://postgres-server:5432/database?ssl=true&sslmode=require
connection-user=trino
connection-password=${ENV:POSTGRES_PASSWORD}
postgresql.ssl.cert=/etc/trino/certificates/postgres-client.crt
postgresql.ssl.key=/etc/trino/certificates/postgres-client.key
postgresql.ssl.root-cert=/etc/trino/certificates/postgres-ca.crt
```

### Iceberg with S3

**Vault Secrets**:
```bash
vault kv put secret/trino-ipc1-maestro/secrets \
  s3_access_key="AKIAIOSFODNN7EXAMPLE" \
  s3_secret_key="wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
```

**Catalog Config**:
```properties
connector.name=iceberg
iceberg.catalog.type=hive_metastore
hive.metastore.uri=thrift://hive-metastore:9083
hive.s3.aws-access-key=${ENV:S3_ACCESS_KEY}
hive.s3.aws-secret-key=${ENV:S3_SECRET_KEY}
hive.s3.endpoint=https://s3.amazonaws.com
```

### Hive with Kerberos

**Vault Secrets**:
```bash
# Store keytab file
vault kv put secret/trino-ipc1-maestro/configfiles \
  trino.keytab=@trino.keytab \
  krb5.conf=@krb5.conf
```

**Catalog Config**:
```properties
connector.name=hive
hive.metastore.uri=thrift://hive-metastore:9083
hive.metastore.authentication.type=KERBEROS
hive.metastore.service.principal=hive/hive-metastore@REALM
hive.metastore.client.principal=trino@REALM
hive.metastore.client.keytab=/etc/trino/configfiles/trino.keytab
hive.hdfs.authentication.type=KERBEROS
hive.hdfs.impersonation.enabled=true
hive.hdfs.trino.principal=trino@REALM
hive.hdfs.trino.keytab=/etc/trino/configfiles/trino.keytab
```

## Security Best Practices

1. **Use Environment Variables for Passwords**: Never hardcode passwords in catalog configs
2. **Rotate Secrets Regularly**: Set appropriate `refreshAfter` intervals
3. **Use PKI for Certificates**: Auto-generate and rotate certificates
4. **Enable HMAC**: Use `hmacSecretData: true` to detect secret changes
5. **Least Privilege**: Grant minimal Vault permissions per deployment
6. **Audit Access**: Monitor Vault audit logs for secret access

## Troubleshooting

### Secrets Not Syncing

```bash
# Check VaultStaticSecret status
kubectl get vaultstaticsecret -n trino-ipc1-maestro

# Check secret exists
kubectl get secret trino-ipc1-maestro-secrets -n trino-ipc1-maestro

# Check Vault Secrets Operator logs
kubectl logs -n vault-secrets-operator-system \
  -l app=vault-secrets-operator
```

### Environment Variables Not Available

```bash
# Check pod environment
kubectl exec -n trino-ipc1-maestro trino-coordinator-0 -- env | grep MYSQL

# Check secret is mounted
kubectl describe pod -n trino-ipc1-maestro trino-coordinator-0
```

### Config Files Not Found

```bash
# Check volume mount
kubectl exec -n trino-ipc1-maestro trino-coordinator-0 -- \
  ls -la /etc/trino/configfiles/

# Check file contents
kubectl exec -n trino-ipc1-maestro trino-coordinator-0 -- \
  cat /etc/trino/configfiles/mysql.properties
```

## Made with Bob