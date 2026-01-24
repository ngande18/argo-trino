# ArgoCD Trino Architecture

## Overview

This repository implements a GitOps-based deployment system for Trino and Vault across multiple clusters using ArgoCD ApplicationSets.

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                         ArgoCD Namespace                         │
│                                                                   │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │              trino-root-app (Application)                   │ │
│  │              Project: default                               │ │
│  │              Source: apps/applicationsets/                  │ │
│  └────────────────────────────────────────────────────────────┘ │
│                              │                                   │
│                              ├─────────────────┬────────────────┤
│                              ▼                 ▼                ▼
│  ┌──────────────────────┐  ┌──────────────────┐  ┌────────────┐ │
│  │  appprojects         │  │ trino            │  │ vault      │ │
│  │  ApplicationSet      │  │ ApplicationSet   │  │ AppSet     │ │
│  └──────────────────────┘  └──────────────────┘  └────────────┘ │
│            │                        │                    │        │
└────────────┼────────────────────────┼────────────────────┼────────┘
             │                        │                    │
             │ Reads cluster configs  │                    │
             │ from Git               │                    │
             ▼                        │                    │
    ┌────────────────┐               │                    │
    │ Git Repository │               │                    │
    │ apps/values/   │               │                    │
    │   clusters/    │               │                    │
    │   *.yaml       │               │                    │
    └────────────────┘               │                    │
             │                        │                    │
             │ Generates per cluster: │                    │
             ▼                        ▼                    ▼
    ┌─────────────────┐    ┌──────────────────┐  ┌──────────────┐
    │ setup-trino-ipc1│    │ trino-trino-ipc1 │  │vault-trino-  │
    │ (Application)   │    │ (Application)    │  │ipc1 (App)    │
    │                 │    │                  │  │              │
    │ Creates:        │    │ Deploys:         │  │Deploys:      │
    │ - Namespace     │    │ - Trino Helm     │  │- Vault Helm  │
    │ - AppProject    │    │   Chart          │  │  Chart       │
    └─────────────────┘    └──────────────────┘  └──────────────┘
             │                        │                    │
             ▼                        ▼                    ▼
    ┌─────────────────┐    ┌──────────────────┐  ┌──────────────┐
    │ Namespace:      │    │ Namespace:       │  │Namespace:    │
    │ trino-ipc1      │    │ trino-ipc1       │  │vault         │
    │                 │    │                  │  │              │
    │ AppProject:     │    │ Trino Pods       │  │Vault Pods    │
    │ trino-ipc1      │    │ Trino Services   │  │Vault Svcs    │
    └─────────────────┘    └──────────────────┘  └──────────────┘
```

## Component Breakdown

### 1. Root Application (`trino-root-app`)

**Location**: `apps/root-app/application.yaml`

**Purpose**: Entry point for the entire deployment system

**Key Properties**:
- Deployed to: `argocd` namespace
- Project: `default` (uses ArgoCD's default project)
- Source: `apps/applicationsets/` directory
- Auto-sync: Enabled with prune and self-heal

**What it deploys**:
- All ApplicationSet resources in the `apps/applicationsets/` directory

### 2. AppProjects ApplicationSet

**Location**: `apps/applicationsets/appprojects.yaml`

**Purpose**: Creates namespaces and AppProjects for each cluster

**Generator**: Git file generator reading `apps/values/clusters/*.yaml`

**For each cluster, it creates**:
- An Application named `setup-{cluster.name}`
- Uses Helm chart from `apps/bootstrap/`
- Renders templates with values from cluster configuration

**Resources Created**:
- Namespace with name from `appProject.name`
- AppProject with permissions and destinations from cluster config

### 3. Trino ApplicationSet

**Location**: `apps/applicationsets/trino/applicationset.yaml`

**Purpose**: Deploys Trino to each cluster

**Generator**: Git file generator reading `apps/values/clusters/*.yaml`

**For each cluster, it creates**:
- An Application named `trino-{cluster.name}`
- Uses Helm chart from trinodb/charts
- Values from `apps/values/trino/{cluster.name}-values.yaml`
- Deploys to namespace specified in cluster config

### 4. Vault ApplicationSet

**Location**: `apps/applicationsets/vault/applicationset.yaml`

**Purpose**: Deploys Vault for secrets management

**Generator**: Git file generator reading `apps/values/clusters/*.yaml`

**For each cluster, it creates**:
- An Application named `vault-{cluster.name}`
- Uses Helm chart from hashicorp/vault
- Values from `apps/values/vault/{cluster.name}-values.yaml`
- Deploys to vault namespace

## Data Flow

### Cluster Configuration

Each cluster is defined in a single YAML file:

```yaml
# apps/values/clusters/trino-ipc1.yaml
cluster:
  name: trino-ipc1
  server: https://kubernetes.default.svc

appProject:
  name: trino-ipc1
  description: AppProject for trino-ipc1 cluster
  sourceRepos: ['*']
  destinations:
    - namespace: trino-ipc1
      server: https://kubernetes.default.svc
    - namespace: vault
      server: https://kubernetes.default.svc

applications:
  trino:
    enabled: true
    namespace: trino-ipc1
    chart:
      repoURL: https://trinodb.github.io/charts
      name: trino
      version: 0.19.0
    valuesFile: apps/values/trino/trino-ipc1-values.yaml
    
  vault:
    enabled: true
    namespace: vault
    chart:
      repoURL: https://helm.releases.hashicorp.com
      name: vault
      version: 0.28.0
    valuesFile: apps/values/vault/trino-ipc1-values.yaml
```

### Bootstrap Helm Chart

Helm chart for creating namespace and AppProject:

```
apps/bootstrap/
├── Chart.yaml              # Helm chart metadata
├── values.yaml             # Default values (placeholders)
├── templates/
│   ├── namespace.yaml      # Namespace template
│   └── appproject.yaml     # AppProject template
└── README.md              # Chart documentation
```

The appprojects ApplicationSet uses this Helm chart and provides values from cluster configurations.

## Deployment Sequence

1. **Apply Root App**: `kubectl apply -f apps/root-app/application.yaml`
2. **Root App Syncs**: Deploys all ApplicationSets to argocd namespace
3. **ApplicationSets Discover**: Read cluster configs from Git
4. **Setup Applications Created**: One per cluster (namespace + AppProject)
5. **Trino Applications Created**: One per cluster
6. **Vault Applications Created**: One per cluster
7. **Resources Deployed**: Namespaces, AppProjects, Trino, and Vault

## Key Features

### 1. Single Source of Truth
- Each cluster defined in one YAML file
- All configuration in Git
- Version controlled and auditable

### 2. Automatic Discovery
- ApplicationSets automatically detect new cluster configs
- No manual ApplicationSet updates needed
- Add cluster = add one file

### 3. Namespace Isolation
- Each cluster gets its own namespace
- Separate AppProject with specific permissions
- Clear resource boundaries

### 4. Flexible Configuration
- Per-cluster Helm values
- Different chart versions per cluster
- Enable/disable applications per cluster

### 5. GitOps Native
- All changes through Git
- Automatic sync and reconciliation
- Self-healing enabled

## Security Model

### AppProject Permissions

Each cluster's AppProject defines:
- **Source Repositories**: Which Git repos can be used
- **Destinations**: Which namespaces and clusters can be targeted
- **Resource Whitelist**: Which Kubernetes resources can be created

### Namespace Isolation

- Each cluster has its own namespace
- Resources cannot cross namespace boundaries
- Clear ownership and access control

### Secrets Management

- Vault deployed for secrets management
- Integration with Trino for credential management
- Sealed secrets or external secrets operators can be added

## Scalability

### Adding Clusters

To add a new cluster:
1. Create cluster config file: `apps/values/clusters/new-cluster.yaml`
2. Create Helm values: `apps/values/trino/new-cluster-values.yaml`
3. Create Helm values: `apps/values/vault/new-cluster-values.yaml`
4. Commit and push
5. ArgoCD automatically creates all resources

### Removing Clusters

To remove a cluster:
1. Delete cluster config file
2. Delete Helm values files
3. Commit and push
4. ArgoCD automatically prunes resources

## Monitoring and Observability

### ArgoCD UI

- View all Applications and their sync status
- See resource health and sync waves
- Access logs and events

### CLI Commands

```bash
# List all applications
argocd app list

# Get application details
argocd app get trino-trino-ipc1

# View sync status
argocd app sync trino-trino-ipc1 --dry-run

# Check ApplicationSet status
kubectl get applicationsets -n argocd
```

### Logs

```bash
# ApplicationSet controller logs
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-applicationset-controller

# Application controller logs
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-application-controller
```

## Best Practices

1. **Use Git Branches**: Test changes in feature branches
2. **Review Before Merge**: Use pull requests for changes
3. **Monitor Sync Status**: Set up alerts for sync failures
4. **Document Changes**: Use meaningful commit messages
5. **Test in Non-Prod**: Validate in test environment first
6. **Keep Values Organized**: Maintain clear structure in values files
7. **Version Control Everything**: All config in Git

## Troubleshooting

See [DEPLOYMENT-GUIDE.md](DEPLOYMENT-GUIDE.md) for detailed troubleshooting steps.

## References

- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [ApplicationSets](https://argo-cd.readthedocs.io/en/stable/user-guide/application-set/)
- [Trino Helm Chart](https://github.com/trinodb/charts)
- [Vault Helm Chart](https://github.com/hashicorp/vault-helm)

# Made with Bob