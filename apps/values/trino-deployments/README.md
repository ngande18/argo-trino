# Trino Deployment Configuration Values

This directory contains deployment-specific configuration files that drive the creation of namespaces, AppProjects, and Applications in ArgoCD.

## Prerequisites

### Trino Helm Chart Setup

The Trino chart is stored as versioned `.tgz` files in the `charts/` directory. To add a new chart version:

```bash
# Add Trino Helm repository
helm repo add trinodb https://trinodb.github.io/charts
helm repo update

# Pull the chart as .tgz file (without extracting)
helm pull trinodb/trino --version 0.19.0 --destination charts/

# Verify the chart file
ls -lh charts/trino-*.tgz
```

**Benefits**:
- ✅ Chart files are committed to Git for version control
- ✅ Multiple versions can coexist (e.g., `trino-0.19.0.tgz`, `trino-0.20.0.tgz`)
- ✅ Each deployment can use a different chart version
- ✅ Easy rollback to previous versions

## Structure

Each Trino deployment has its own YAML configuration file:
- `trino-ipc1.yaml` - Configuration for trino-ipc1 deployment
- `trino-ipc2.yaml` - Configuration for trino-ipc2 deployment
- `cdaitrino-e3h.yaml` - Configuration for cdaitrino-e3h deployment (test environment)

## Configuration Format

Each deployment configuration file contains:

### 1. Deployment Information
```yaml
trinoDeployment:
  name: trino-ipc1
  description: Trino instance trino-ipc1
```

### 2. Cluster Information
```yaml
# Use the cluster name as registered in ArgoCD
cluster:
  name: ipc-cluster-1
```

**How to get cluster names:**
```bash
# List all clusters registered in ArgoCD
kubectl get secrets -n argocd -l argocd.argoproj.io/secret-type=cluster -o jsonpath='{range .items[*]}{.metadata.annotations.argocd\.argoproj\.io/cluster-name}{"\n"}{end}'

# Or using ArgoCD CLI
argocd cluster list
```

### 3. Git Repository Configuration
Defines the Git repository and revision for all resources (bootstrap and applications):
```yaml
git:
  repoURL: https://github.com/your-org/your-repo.git
  targetRevision: HEAD  # or feature/branch-name for testing
```

**Important**: This single `targetRevision` is used for:
- Bootstrap resources (namespace, AppProject)
- Application Helm values files (Trino, Vault)

This ensures all resources for a deployment use the same Git branch, making it easy to test changes in feature branches.

### 4. AppProject Configuration
Defines the ArgoCD AppProject for the cluster:
```yaml
appProject:
  name: trino-ipc1
  description: AppProject for trino-ipc1 cluster
  sourceRepos:
    - '*'
  destinations:
    - namespace: trino-ipc1
      name: ipc-cluster-1  # Use cluster name, not server URL
    - namespace: vault
      name: ipc-cluster-1
  clusterResourceWhitelist:
    - group: '*'
      kind: '*'
  namespaceResourceWhitelist:
    - group: '*'
      kind: '*'
  orphanedResources:
    warn: true
```

### 5. Application Configurations
Defines applications to be deployed to the cluster:

#### Trino Application
```yaml
applications:
  trino:
    enabled: true
    namespace: trino-ipc1
    chart:
      path: charts/trino-0.19.0.tgz  # Specify chart version
    valuesFile: apps/values/trino/trino-ipc1-values.yaml
    syncPolicy:
      automated:
        prune: true
        selfHeal: true
        allowEmpty: false
      syncOptions:
        - CreateNamespace=true
        - ServerSideApply=true
      retry:
        limit: 5
        backoff:
          duration: 5s
          factor: 2
          maxDuration: 3m
```

#### Vault Application
```yaml
  vault:
    enabled: true
    namespace: vault
    chart:
      repoURL: https://helm.releases.hashicorp.com
      name: vault
      version: 0.28.0
    valuesFile: apps/values/vault/trino-ipc1-values.yaml
    syncPolicy:
      automated:
        prune: true
        selfHeal: true
        allowEmpty: false
      syncOptions:
        - CreateNamespace=true
        - ServerSideApply=true
      retry:
        limit: 5
        backoff:
          duration: 5s
          factor: 2
          maxDuration: 3m
    ignoreDifferences:
      - group: admissionregistration.k8s.io
        kind: MutatingWebhookConfiguration
        jqPathExpressions:
          - '.webhooks[]?.clientConfig.caBundle'
```

## How It Works

1. **Bootstrap Resources**: The `apps/applicationsets/appprojects.yaml` ApplicationSet reads these deployment configuration files and uses the Helm chart in `apps/bootstrap/` to create:
   - Namespace for the deployment
   - AppProject with appropriate permissions

2. **Application Generation**: The ApplicationSets in `apps/applicationsets/trino/` and `apps/applicationsets/vault/` read these configuration files and generate Application resources for each enabled application.

3. **Git Generator**: All ApplicationSets use the Git generator to automatically discover cluster configuration files:
   ```yaml
   generators:
     - git:
         repoURL: https://github.com/your-org/your-repo.git
         revision: HEAD
         files:
           - path: "apps/values/trino-deployments/*.yaml"
   ```

## Adding a New Deployment

To add a new Trino deployment:

1. Create a new configuration file: `apps/values/trino-deployments/<deployment-name>.yaml`
2. Copy the structure from an existing deployment file
3. Update all deployment-specific values:
   - `trinoDeployment.name` and `trinoDeployment.description`
   - `cluster.name` (get from ArgoCD using the command above)
   - `git.targetRevision` (use `HEAD` for main branch or feature branch for testing)
   - `appProject.name` and `appProject.description`
   - `appProject.destinations` (update namespaces and cluster names)
   - `applications.trino.namespace`
   - `applications.trino.valuesFile`
   - `applications.vault.valuesFile`
4. Create corresponding Helm values files:
   - `apps/values/trino/<deployment-name>-values.yaml`
   - `apps/values/vault/<deployment-name>-values.yaml`
5. Commit and push changes - ArgoCD will automatically detect and create resources

**Note**: Multiple Trino deployments can run on the same Kubernetes cluster by using different namespaces.

## Disabling Applications

To disable an application for a specific cluster, set `enabled: false`:
```yaml
applications:
  trino:
    enabled: false
```

## Testing Changes with Feature Branches

To test changes in a feature branch:

1. Create a feature branch in your Git repository
2. Update the deployment configuration:
   ```yaml
   git:
     targetRevision: feature/my-test-branch
   ```
3. Commit and push - ArgoCD will deploy from the feature branch
4. All resources (bootstrap and applications) will use the feature branch

This is useful for testing changes before merging to main.

## Customizing Sync Policies

Each application can have its own sync policy. Modify the `syncPolicy` section in the deployment configuration file to customize:
- Automated sync behavior
- Sync options
- Retry logic

## Best Practices

1. **Consistent Naming**: Use consistent naming patterns across clusters
2. **Version Control**: Always version control cluster configurations
3. **Documentation**: Document any cluster-specific customizations
4. **Testing**: Test configuration changes in a non-production cluster first
5. **Validation**: Validate YAML syntax before committing

## Troubleshooting

### ApplicationSet Not Generating Resources
- Verify the Git repository URL is correct in the ApplicationSet
- Check that the deployment configuration file is in the correct path (`apps/values/trino-deployments/`)
- Ensure the YAML syntax is valid
- Check ArgoCD ApplicationSet controller logs

### Application Not Syncing
- Verify the AppProject exists and has correct permissions
- Check that the values file path is correct
- Ensure the Helm chart repository is accessible
- Review Application sync status and events in ArgoCD UI

## Key Differences from Cluster-Based Configuration

This deployment-based approach differs from traditional cluster-based configuration:

1. **Multiple deployments per cluster**: You can run multiple Trino instances on the same Kubernetes cluster
2. **Deployment-centric naming**: Resources are named after the deployment, not the cluster
3. **Cluster name references**: Uses ArgoCD cluster names instead of server URLs
4. **Single Git configuration**: One `git.targetRevision` controls all resources for a deployment

# Made with Bob