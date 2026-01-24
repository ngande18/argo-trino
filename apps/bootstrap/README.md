# Bootstrap Helm Chart

This Helm chart creates the foundational resources for each Trino cluster: a namespace and an AppProject.

## Overview

The bootstrap chart is used by the `appprojects` ApplicationSet to automatically generate namespaces and AppProjects based on cluster configuration files.

## Structure

```
bootstrap/
├── Chart.yaml              # Helm chart metadata
├── values.yaml             # Default values (placeholders)
├── templates/
│   ├── namespace.yaml      # Namespace template
│   └── appproject.yaml     # AppProject template
└── README.md              # This file
```

## Templates

### namespace.yaml

Creates a Kubernetes namespace with:
- Name from `values.namespace`
- Label `managed-by: argocd`
- Label `cluster: <cluster-name>`

### appproject.yaml

Creates an ArgoCD AppProject with:
- Name from `values.appProject.name`
- Description, source repos, destinations from values
- Resource whitelists for cluster and namespace resources
- Orphaned resources warning enabled

## Values

The chart expects the following values structure:

```yaml
namespace: <namespace-name>
clusterName: <cluster-name>

appProject:
  name: <project-name>
  description: <project-description>
  sourceRepos:
    - '*'
  destinations:
    - namespace: <namespace>
      server: <server-url>
  clusterResourceWhitelist:
    - group: '*'
      kind: '*'
  namespaceResourceWhitelist:
    - group: '*'
      kind: '*'
  orphanedResources:
    warn: true
```

## Usage

This chart is not meant to be installed directly. It's used by the `appprojects` ApplicationSet which:

1. Reads cluster configuration files from `apps/values/clusters/*.yaml`
2. For each cluster, creates an Application
3. The Application uses this Helm chart with values from the cluster config
4. Helm renders the templates with the provided values
5. ArgoCD deploys the namespace and AppProject

## Example

For a cluster configuration file `apps/values/clusters/trino-ipc1.yaml`:

```yaml
cluster:
  name: trino-ipc1
  
appProject:
  name: trino-ipc1
  description: AppProject for trino-ipc1 cluster
  sourceRepos: ['*']
  destinations:
    - namespace: trino-ipc1
      server: https://kubernetes.default.svc
```

The ApplicationSet will create an Application that renders this chart with:

```yaml
namespace: trino-ipc1
clusterName: trino-ipc1
appProject:
  name: trino-ipc1
  description: AppProject for trino-ipc1 cluster
  # ... rest of the configuration
```

This results in:
- Namespace: `trino-ipc1`
- AppProject: `trino-ipc1` (in argocd namespace)

## Customization

To modify the base structure of namespaces or AppProjects:

1. Edit the templates in `templates/`
2. Update the values structure if needed
3. Commit and push changes
4. ArgoCD will automatically sync the changes

## Notes

- The `values.yaml` file contains placeholder values for validation
- Actual values are provided by the ApplicationSet at runtime
- All resources are created in the cluster specified in the cluster config
- AppProjects are always created in the `argocd` namespace

# Made with Bob