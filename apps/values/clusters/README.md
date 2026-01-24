# Cluster Configuration Values

This directory contains cluster-specific configuration files that drive the creation of AppProjects and ApplicationSets in ArgoCD.

## Structure

Each cluster has its own YAML configuration file:
- `trino-ipc1.yaml` - Configuration for trino-ipc1 cluster
- `trino-ipc2.yaml` - Configuration for trino-ipc2 cluster

## Configuration Format

Each cluster configuration file contains:

### 1. Cluster Information
```yaml
cluster:
  name: trino-ipc1
  server: https://kubernetes.default.svc
```

### 2. AppProject Configuration
Defines the ArgoCD AppProject for the cluster:
```yaml
appProject:
  name: trino-ipc1
  description: AppProject for trino-ipc1 cluster
  sourceRepos:
    - '*'
  destinations:
    - namespace: trino-ipc1
      server: https://kubernetes.default.svc
    - namespace: vault
      server: https://kubernetes.default.svc
  clusterResourceWhitelist:
    - group: '*'
      kind: '*'
  namespaceResourceWhitelist:
    - group: '*'
      kind: '*'
  orphanedResources:
    warn: true
```

### 3. Application Configurations
Defines applications to be deployed to the cluster:

#### Trino Application
```yaml
applications:
  trino:
    enabled: true
    namespace: trino-ipc1
    chart:
      repoURL: https://trinodb.github.io/charts
      name: trino
      version: 0.19.0
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

### 4. Values Repository
Git repository containing Helm values files:
```yaml
valuesRepo:
  url: https://github.com/your-org/your-repo.git
  targetRevision: HEAD
```

## How It Works

1. **AppProject Generation**: The `apps/applicationsets/appprojects-template.yaml` ApplicationSet reads these cluster configuration files and generates an AppProject for each cluster.

2. **Application Generation**: The ApplicationSets in `apps/applicationsets/trino/` and `apps/applicationsets/vault/` read these configuration files and generate Application resources for each enabled application in each cluster.

3. **Git Generator**: All ApplicationSets use the Git generator to automatically discover cluster configuration files:
   ```yaml
   generators:
     - git:
         repoURL: https://github.com/your-org/your-repo.git
         revision: HEAD
         files:
           - path: "apps/values/clusters/*.yaml"
   ```

## Adding a New Cluster

To add a new cluster:

1. Create a new configuration file: `apps/values/clusters/<cluster-name>.yaml`
2. Copy the structure from an existing cluster file
3. Update all cluster-specific values:
   - `cluster.name`
   - `appProject.name` and `appProject.description`
   - `appProject.destinations` (namespaces)
   - `applications.trino.namespace`
   - `applications.trino.valuesFile`
   - `applications.vault.valuesFile`
4. Create corresponding Helm values files:
   - `apps/values/trino/<cluster-name>-values.yaml`
   - `apps/values/vault/<cluster-name>-values.yaml`
5. Commit and push changes - ArgoCD will automatically detect and create resources

## Disabling Applications

To disable an application for a specific cluster, set `enabled: false`:
```yaml
applications:
  trino:
    enabled: false
```

## Customizing Sync Policies

Each application can have its own sync policy. Modify the `syncPolicy` section in the cluster configuration file to customize:
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
- Check that the cluster configuration file is in the correct path
- Ensure the YAML syntax is valid
- Check ArgoCD ApplicationSet controller logs

### Application Not Syncing
- Verify the AppProject exists and has correct permissions
- Check that the values file path is correct
- Ensure the Helm chart repository is accessible
- Review Application sync status and events in ArgoCD UI

## Migration from Static Configuration

The old static configuration files are:
- `apps/applicationsets/appprojects.yaml` (static AppProjects)
- `apps/applicationsets/trino/applicationset.yaml` (static list generator)
- `apps/applicationsets/vault/applicationset.yaml` (static list generator)

The new template-based files are:
- `apps/applicationsets/appprojects-template.yaml` (dynamic AppProject generator)
- `apps/applicationsets/trino/applicationset-template.yaml` (dynamic Trino generator)
- `apps/applicationsets/vault/applicationset-template.yaml` (dynamic Vault generator)

To migrate:
1. Review and update cluster configuration files
2. Update Git repository URLs in template files
3. Apply the new template-based ApplicationSets
4. Verify resources are created correctly
5. Remove old static ApplicationSets once verified

# Made with Bob