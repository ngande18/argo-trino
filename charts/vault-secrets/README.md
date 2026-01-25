# Vault Secrets Helm Chart

This Helm chart creates VaultAuth and VaultStaticSecret resources for the Vault Secrets Operator.

## Prerequisites

- Kubernetes cluster with Vault Secrets Operator installed
- Vault server configured and accessible
- VaultConnection resource created

## Installation

```bash
helm install vault-secrets charts/vault-secrets -f values.yaml
```

## Configuration

### VaultAuth

Creates a VaultAuth resource for Kubernetes authentication to Vault.

```yaml
vaultAuth:
  enabled: true
  name: vault-auth
  vaultConnectionRef: vault-connection
  kubernetes:
    role: trino-role
    serviceAccount: trino
    audiences:
      - vault
    mount: kubernetes  # Optional, defaults to "kubernetes"
    tokenExpirationSeconds: 600  # Optional
```

### VaultStaticSecret

Creates VaultStaticSecret resources to sync secrets from Vault to Kubernetes.

```yaml
vaultStaticSecrets:
  - name: trino-db-credentials
    enabled: true
    vaultAuthRef: vault-auth
    mount: secret
    path: trino/database
    type: kv-v2
    refreshAfter: 30s
    hmacSecretData: true
    destination:
      name: trino-db-secret
      create: true
      labels:
        app: trino
      annotations:
        reloader.stakater.com/match: "true"
```

## Example Values File

### Basic Configuration

```yaml
# VaultAuth configuration
vaultAuth:
  enabled: true
  name: trino-vault-auth
  vaultConnectionRef: vault-connection
  kubernetes:
    role: trino-role
    serviceAccount: trino
    audiences:
      - vault

# VaultStaticSecret configuration
vaultStaticSecrets:
  # Database credentials
  - name: trino-db-credentials
    enabled: true
    vaultAuthRef: trino-vault-auth
    mount: secret
    path: trino/database
    type: kv-v2
    refreshAfter: 30s
    hmacSecretData: true
    destination:
      name: trino-db-secret
      create: true
  
  # S3 credentials
  - name: trino-s3-credentials
    enabled: true
    vaultAuthRef: trino-vault-auth
    mount: secret
    path: trino/s3
    type: kv-v2
    refreshAfter: 60s
    destination:
      name: trino-s3-secret
      create: true
```

### Advanced Configuration with Rollout Restart

```yaml
vaultStaticSecrets:
  - name: trino-credentials
    enabled: true
    vaultAuthRef: trino-vault-auth
    mount: secret
    path: trino/credentials
    type: kv-v2
    refreshAfter: 30s
    hmacSecretData: true
    # Automatically restart deployments when secret changes
    rolloutRestartTargets:
      - kind: Deployment
        name: trino-coordinator
      - kind: StatefulSet
        name: trino-worker
    destination:
      name: trino-credentials
      create: true
      labels:
        app: trino
        component: credentials
      annotations:
        reloader.stakater.com/match: "true"
```

### With Secret Transformation

```yaml
vaultStaticSecrets:
  - name: trino-transformed-secret
    enabled: true
    vaultAuthRef: trino-vault-auth
    mount: secret
    path: trino/config
    type: kv-v2
    destination:
      name: trino-config
      create: true
      transformation:
        excludes:
          - ".*-internal"
        includes:
          - ".*"
        templates:
          config.properties: |
            {{ get .Secrets "key1" }}={{ get .Secrets "value1" }}
            {{ get .Secrets "key2" }}={{ get .Secrets "value2" }}
```

## Parameters

### VaultAuth Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `vaultAuth.enabled` | Enable VaultAuth creation | `true` |
| `vaultAuth.name` | Name of VaultAuth resource | `vault-auth` |
| `vaultAuth.namespace` | Namespace for VaultAuth | Release namespace |
| `vaultAuth.vaultConnectionRef` | Reference to VaultConnection | `vault-connection` |
| `vaultAuth.kubernetes.role` | Vault Kubernetes role | `trino-role` |
| `vaultAuth.kubernetes.serviceAccount` | Kubernetes service account | `trino` |
| `vaultAuth.kubernetes.audiences` | Token audiences | `["vault"]` |
| `vaultAuth.kubernetes.mount` | Vault auth mount path | `kubernetes` |
| `vaultAuth.kubernetes.tokenExpirationSeconds` | Token expiration | - |

### VaultStaticSecret Parameters

| Parameter | Description | Required |
|-----------|-------------|----------|
| `name` | Name of VaultStaticSecret | Yes |
| `enabled` | Enable this secret | Yes |
| `vaultAuthRef` | Reference to VaultAuth | Yes |
| `mount` | Vault mount path | Yes |
| `path` | Secret path in Vault | Yes |
| `type` | Secret type (kv-v1, kv-v2) | No (default: kv-v2) |
| `version` | Specific secret version | No |
| `refreshAfter` | Refresh interval | No |
| `hmacSecretData` | HMAC secret data | No |
| `rolloutRestartTargets` | Resources to restart on change | No |
| `destination.name` | Kubernetes secret name | Yes |
| `destination.create` | Create secret if not exists | No (default: true) |
| `destination.labels` | Labels for secret | No |
| `destination.annotations` | Annotations for secret | No |
| `destination.type` | Kubernetes secret type | No |
| `destination.transformation` | Secret transformation rules | No |

## Usage with ArgoCD

### ApplicationSet Integration

```yaml
# apps/applicationsets/vault-secrets/applicationset.yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: vault-secrets-applicationset
  namespace: argocd
spec:
  generators:
    - git:
        repoURL: https://github.com/your-org/your-repo.git
        revision: HEAD
        files:
          - path: "apps/values/trino-deployments/*.yaml"
  template:
    metadata:
      name: 'vault-secrets-{{.trinoDeployment.name}}'
      namespace: argocd
    spec:
      project: '{{.appProject.name}}'
      source:
        repoURL: '{{.git.repoURL}}'
        targetRevision: '{{.git.targetRevision}}'
        path: charts/vault-secrets
        helm:
          releaseName: vault-secrets
          valueFiles:
            - ../../{{.applications.vaultSecrets.valuesFile}}
      destination:
        name: '{{.cluster.name}}'
        namespace: '{{.applications.trino.namespace}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

### Deployment Configuration

```yaml
# apps/values/trino-deployments/trino-ipc1.yaml
applications:
  trino:
    # ... trino config ...
  
  vaultSecrets:
    enabled: true
    valuesFile: apps/values/vault-secrets/trino-ipc1-values.yaml
```

### Values File

```yaml
# apps/values/vault-secrets/trino-ipc1-values.yaml
vaultAuth:
  enabled: true
  name: trino-ipc1-vault-auth
  vaultConnectionRef: vault-connection
  kubernetes:
    role: trino-ipc1-role
    serviceAccount: trino
    audiences:
      - vault

vaultStaticSecrets:
  - name: trino-ipc1-db-credentials
    enabled: true
    vaultAuthRef: trino-ipc1-vault-auth
    mount: secret
    path: trino-ipc1/database
    type: kv-v2
    refreshAfter: 30s
    hmacSecretData: true
    destination:
      name: trino-db-secret
      create: true
```

## Testing

```bash
# Lint the chart
helm lint charts/vault-secrets

# Template the chart
helm template test charts/vault-secrets -f values.yaml

# Dry run
helm install test charts/vault-secrets -f values.yaml --dry-run --debug

# Install
helm install vault-secrets charts/vault-secrets -f values.yaml
```

## Troubleshooting

### VaultAuth Not Working

Check VaultAuth status:
```bash
kubectl get vaultauth -n <namespace>
kubectl describe vaultauth <name> -n <namespace>
```

### VaultStaticSecret Not Syncing

Check VaultStaticSecret status:
```bash
kubectl get vaultstaticsecret -n <namespace>
kubectl describe vaultstaticsecret <name> -n <namespace>
```

Check Vault Secrets Operator logs:
```bash
kubectl logs -n vault-secrets-operator-system -l app=vault-secrets-operator
```

## References

- [Vault Secrets Operator Documentation](https://developer.hashicorp.com/vault/docs/platform/k8s/vso)
- [VaultAuth API Reference](https://developer.hashicorp.com/vault/docs/platform/k8s/vso/api-reference#vaultauth)
- [VaultStaticSecret API Reference](https://developer.hashicorp.com/vault/docs/platform/k8s/vso/api-reference#vaultstaticsecret)

# Made with Bob