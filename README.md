# ArgoCD Trino Deployment

Template-based ArgoCD ApplicationSets for managing Trino and Vault deployments across multiple clusters.

## Architecture

This repository uses a **values-driven template approach** where each cluster's configuration is defined in a single YAML file. ApplicationSets automatically discover cluster configurations and generate the necessary ArgoCD resources.

### Flow

1. **Root Application** (`trino-root-app`) is deployed to ArgoCD namespace using the `default` project
2. Root app deploys all ApplicationSets to the ArgoCD namespace
3. **AppProjects ApplicationSet** creates:
   - Namespace for each cluster (based on `appProject.name`)
   - AppProject for each cluster (with permissions and destinations)
4. **Trino ApplicationSet** creates Trino applications in cluster namespaces
5. **Vault ApplicationSet** creates Vault applications in vault namespace

## Directory Structure

```
apps/
├── bootstrap/                 # Helm chart for namespace/AppProject creation
│   ├── Chart.yaml
│   ├── values.yaml
│   ├── README.md
│   └── templates/
│       ├── namespace.yaml     # Namespace template
│       └── appproject.yaml    # AppProject template
├── values/
│   ├── clusters/              # Cluster configuration files (drives everything)
│   │   ├── README.md         # Detailed cluster config documentation
│   │   ├── trino-ipc1.yaml   # Cluster 1 configuration
│   │   └── trino-ipc2.yaml   # Cluster 2 configuration
│   ├── trino/                 # Trino Helm values per cluster
│   │   ├── trino-ipc1-values.yaml
│   │   └── trino-ipc2-values.yaml
│   └── vault/                 # Vault Helm values per cluster
│       ├── trino-ipc1-values.yaml
│       └── trino-ipc2-values.yaml
├── applicationsets/
│   ├── appprojects.yaml       # Generates Namespaces & AppProjects
│   ├── trino/
│   │   └── applicationset.yaml  # Generates Trino Applications
│   └── vault/
│       └── applicationset.yaml  # Generates Vault Applications
└── root-app/
    └── application.yaml       # Root Application (App of Apps)
```

## How It Works

1. **Cluster Configuration**: Each cluster has a configuration file in `apps/values/clusters/`
2. **Git File Generator**: ApplicationSets use Git file generators to discover cluster configs
3. **Automatic Generation**: ArgoCD automatically creates:
   - Namespace for each cluster (named after `appProject.name`)
   - AppProject for each cluster (with proper permissions)
   - Trino Application for each cluster
   - Vault Application for each cluster

## Quick Start

### Prerequisites

- ArgoCD installed in your Kubernetes cluster
- Git repository access configured in ArgoCD
- Kubectl access to the cluster

### Setup

1. **Get cluster names from ArgoCD:**
   
   Before configuring deployments, get the cluster names as registered in ArgoCD:
   ```bash
   # List all clusters registered in ArgoCD
   kubectl get secrets -n argocd -l argocd.argoproj.io/secret-type=cluster -o jsonpath='{range .items[*]}{.metadata.annotations.argocd\.argoproj\.io/cluster-name}{"\n"}{end}'
   
   # Or using ArgoCD CLI
   argocd cluster list
   ```
   
   Use these cluster names in your deployment configuration files.

2. **Clone the repository:**
   ```bash
   git clone <your-repo-url>
   cd argo-trino
   ```

3. **Pull the Trino Helm chart:**
   
   The Trino chart is stored as `.tgz` files in the `charts/` directory:
   ```bash
   # Add Trino Helm repository
   helm repo add trinodb https://trinodb.github.io/charts
   helm repo update
   
   # Pull the chart as .tgz file (version 0.19.0 or your desired version)
   helm pull trinodb/trino --version 0.19.0 --destination charts/
   
   # Verify the chart file
   ls -lh charts/trino-*.tgz
   ```
   
   **Note**: The `.tgz` files are committed to Git, allowing version control and multiple chart versions.

4. **Update Git repository URLs:**
   
   Edit these files and replace `https://github.com/your-org/your-repo.git` with your actual repository URL:
   - `apps/applicationsets/appprojects.yaml`
   - `apps/applicationsets/trino/applicationset.yaml`
   - `apps/applicationsets/vault/applicationset.yaml`
   - `apps/values/clusters/trino-ipc1.yaml`
   - `apps/values/clusters/trino-ipc2.yaml`

5. **Review cluster configurations:**
   ```bash
   cat apps/values/clusters/trino-ipc1.yaml
   cat apps/values/clusters/trino-ipc2.yaml
   ```

6. **Apply the root application:**
   ```bash
   kubectl apply -f apps/root-app/application.yaml
   ```
   
   Note: The root app uses the `default` ArgoCD project, so no separate AppProject is needed.

7. **Verify deployment:**
   ```bash
   # Check ApplicationSets
   kubectl get applicationsets -n argocd
   
   # Check AppProjects
   kubectl get appprojects -n argocd
   
   # Check Applications
   kubectl get applications -n argocd
   ```

## Adding a New Cluster

Adding a new cluster is simple - just create a new configuration file:

1. **Create cluster configuration:**
   ```bash
   cp apps/values/clusters/trino-ipc1.yaml apps/values/clusters/trino-ipc3.yaml
   ```

2. **Update cluster-specific values:**
   Edit `apps/values/clusters/trino-ipc3.yaml`:
   - Change `cluster.name` to `trino-ipc3`
   - Update `appProject.name` and description
   - Update namespace references
   - Update values file paths

3. **Create Helm values files:**
   ```bash
   cp apps/values/trino/trino-ipc1-values.yaml apps/values/trino/trino-ipc3-values.yaml
   cp apps/values/vault/trino-ipc1-values.yaml apps/values/vault/trino-ipc3-values.yaml
   ```

4. **Customize Helm values** for the new cluster

5. **Commit and push:**
   ```bash
   git add apps/values/clusters/trino-ipc3.yaml
   git add apps/values/trino/trino-ipc3-values.yaml
   git add apps/values/vault/trino-ipc3-values.yaml
   git commit -m "Add trino-ipc3 cluster"
   git push
   ```

6. **ArgoCD automatically creates** the resources within minutes!

## Cluster Configuration

Each cluster configuration file contains:

- **Cluster identification** (name, server)
- **AppProject definition** (permissions, destinations)
- **Application configurations** (Trino, Vault)
- **Sync policies** (automated sync, retry logic)
- **Chart versions** and repository URLs
- **Values file paths**

See `apps/values/clusters/README.md` for detailed documentation.

## Customization

### Disable an Application

To disable Trino or Vault for a specific cluster:

```yaml
applications:
  trino:
    enabled: false  # Don't deploy Trino to this cluster
```

### Use Different Chart Versions

Each cluster can use different chart versions:

```yaml
applications:
  trino:
    chart:
      version: 0.20.0  # Use newer version for this cluster
```

### Customize Sync Policies

Control sync behavior per cluster:

```yaml
applications:
  trino:
    syncPolicy:
      automated:
        prune: false      # Don't auto-prune
        selfHeal: false   # Manual sync only
```

## Components

### Trino
- **Chart**: [trinodb/trino](https://trinodb.github.io/charts)
- **Version**: Configurable per cluster
- **Purpose**: Distributed SQL query engine

### Vault
- **Chart**: [hashicorp/vault](https://helm.releases.hashicorp.com)
- **Version**: Configurable per cluster
- **Purpose**: Secrets management

## Monitoring

### Check ApplicationSet Status
```bash
kubectl get applicationsets -n argocd
kubectl describe applicationset trino-applicationset -n argocd
```

### Check Application Status
```bash
argocd app list
argocd app get trino-trino-ipc1
argocd app sync trino-trino-ipc1  # Manual sync if needed
```

### View Logs
```bash
# ApplicationSet controller logs
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-applicationset-controller

# ArgoCD application controller logs
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-application-controller
```

## Troubleshooting

### ApplicationSet Not Generating Resources

1. Check ApplicationSet controller logs
2. Verify Git repository URL is correct
3. Ensure cluster config file is in `apps/values/clusters/*.yaml`
4. Validate YAML syntax

### Applications Not Syncing

1. Check AppProject permissions
2. Verify values file paths are correct
3. Review Application sync status in ArgoCD UI
4. Check Application events

### Git Repository Access Issues

1. Verify repository credentials in ArgoCD:
   ```bash
   argocd repo list
   ```

2. Add repository if needed:
   ```bash
   argocd repo add <repo-url> --username <user> --password <token>
   ```

## Best Practices

1. **Version Control**: All configurations in Git
2. **Descriptive Names**: Use clear cluster names
3. **Documentation**: Comment cluster-specific customizations
4. **Testing**: Test in non-production first
5. **Consistency**: Follow naming conventions
6. **Monitoring**: Set up alerts for sync failures
7. **Backups**: Regular backups of cluster configurations

## Benefits

- ✅ **Single Source of Truth**: One file per cluster
- ✅ **Automatic Discovery**: No manual ApplicationSet updates
- ✅ **Scalable**: Easy to add/remove clusters
- ✅ **GitOps Native**: Everything in version control
- ✅ **Flexible**: Per-cluster customization
- ✅ **Maintainable**: Clear structure and documentation

## Documentation

- [Cluster Configuration Guide](apps/values/clusters/README.md) - Detailed cluster config documentation
- [Deployment Guide](DEPLOYMENT-GUIDE.md) - Step-by-step deployment instructions
- [Architecture Guide](ARCHITECTURE.md) - Detailed architecture documentation
- [Cluster Onboarding Guide](CLUSTER-ONBOARDING.md) - Adding new clusters (production or test-first)
- [ArgoCD ApplicationSets](https://argo-cd.readthedocs.io/en/stable/user-guide/application-set/) - Official documentation

## Support

For issues or questions:
1. Check the cluster configuration README
2. Review ApplicationSet controller logs
3. Consult ArgoCD documentation

## License

[Your License Here]

# Made with Bob