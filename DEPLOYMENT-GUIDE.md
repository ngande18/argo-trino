# ArgoCD Trino Deployment Guide

This guide walks you through deploying the complete ArgoCD-based Trino infrastructure.

## Architecture Overview

The deployment follows this hierarchy:

```
trino-root-app (ArgoCD Application in default project)
    │
    ├── appprojects ApplicationSet
    │   └── Creates for each cluster:
    │       ├── Namespace (e.g., trino-ipc1)
    │       └── AppProject (e.g., trino-ipc1)
    │
    ├── trino ApplicationSet
    │   └── Creates Trino Application in each cluster namespace
    │
    └── vault ApplicationSet
        └── Creates Vault Application in vault namespace
```

## Prerequisites

1. **ArgoCD Installed**: ArgoCD must be running in your Kubernetes cluster
2. **Git Repository Access**: ArgoCD needs access to your Git repository
3. **Kubectl Access**: You need cluster admin access
4. **Repository Configured**: Your Git repository URL must be configured in all files

## Step-by-Step Deployment

### Step 1: Configure Git Repository

Update the repository URL in the following files:

```bash
# Update these files with your actual Git repository URL
# Replace: https://github.com/your-org/your-repo.git

- apps/root-app/application.yaml
- apps/applicationsets/appprojects.yaml
- apps/applicationsets/trino/applicationset.yaml
- apps/applicationsets/vault/applicationset.yaml
- apps/values/clusters/trino-ipc1.yaml
- apps/values/clusters/trino-ipc2.yaml
```

### Step 2: Review Cluster Configurations

Each cluster is defined in `apps/values/clusters/`:

```bash
# Review cluster configurations
cat apps/values/clusters/trino-ipc1.yaml
cat apps/values/clusters/trino-ipc2.yaml
```

Key configuration elements:
- `cluster.name`: Unique cluster identifier
- `appProject.name`: Name for the namespace and AppProject (typically same as cluster name)
- `applications.trino`: Trino deployment configuration
- `applications.vault`: Vault deployment configuration

### Step 3: Customize Helm Values

Update Helm values for each cluster:

```bash
# Trino values
apps/values/trino/trino-ipc1-values.yaml
apps/values/trino/trino-ipc2-values.yaml

# Vault values
apps/values/vault/trino-ipc1-values.yaml
apps/values/vault/trino-ipc2-values.yaml
```

### Step 4: Commit and Push Changes

```bash
git add .
git commit -m "Configure ArgoCD Trino deployment"
git push origin main
```

### Step 5: Deploy Root Application

Apply the root application to ArgoCD:

```bash
kubectl apply -f apps/root-app/application.yaml
```

This creates the `trino-root-app` Application in the ArgoCD namespace using the `default` project.

### Step 6: Verify Deployment

#### Check ApplicationSets

```bash
kubectl get applicationsets -n argocd

# Expected output:
# NAME            AGE
# appprojects     Xs
# trino-applicationset    Xs
# vault-applicationset    Xs
```

#### Check Generated Applications

```bash
kubectl get applications -n argocd

# Expected output:
# NAME                    SYNC STATUS   HEALTH STATUS
# trino-root-app         Synced        Healthy
# setup-trino-ipc1       Synced        Healthy
# setup-trino-ipc2       Synced        Healthy
# trino-trino-ipc1       Synced        Healthy
# trino-trino-ipc2       Synced        Healthy
# vault-trino-ipc1       Synced        Healthy
# vault-trino-ipc2       Synced        Healthy
```

#### Check AppProjects

```bash
kubectl get appprojects -n argocd

# Expected output:
# NAME           AGE
# default        Xd
# trino-ipc1     Xs
# trino-ipc2     Xs
```

#### Check Namespaces

```bash
kubectl get namespaces | grep trino

# Expected output:
# trino-ipc1    Active   Xs
# trino-ipc2    Active   Xs
```

#### Check Trino Deployments

```bash
# Check Trino pods in each namespace
kubectl get pods -n trino-ipc1
kubectl get pods -n trino-ipc2
```

#### Check Vault Deployments

```bash
# Check Vault pods
kubectl get pods -n vault
```

### Step 7: Access ArgoCD UI

View the deployment in ArgoCD UI:

```bash
# Port forward to ArgoCD server
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Access UI at: https://localhost:8080
# Username: admin
# Password: (get with command below)
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

## Adding a New Cluster

To add a new cluster (e.g., `trino-ipc3`):

### 1. Create Cluster Configuration

```bash
cp apps/values/clusters/trino-ipc1.yaml apps/values/clusters/trino-ipc3.yaml
```

Edit `apps/values/clusters/trino-ipc3.yaml`:
- Change `cluster.name` to `trino-ipc3`
- Change `appProject.name` to `trino-ipc3`
- Update `appProject.description`
- Update namespace references
- Update values file paths

### 2. Create Helm Values Files

```bash
cp apps/values/trino/trino-ipc1-values.yaml apps/values/trino/trino-ipc3-values.yaml
cp apps/values/vault/trino-ipc1-values.yaml apps/values/vault/trino-ipc3-values.yaml
```

Customize the values for the new cluster.

### 3. Commit and Push

```bash
git add apps/values/clusters/trino-ipc3.yaml
git add apps/values/trino/trino-ipc3-values.yaml
git add apps/values/vault/trino-ipc3-values.yaml
git commit -m "Add trino-ipc3 cluster"
git push
```

### 4. Verify Auto-Generation

Within 3 minutes, ArgoCD will automatically create:
- Namespace: `trino-ipc3`
- AppProject: `trino-ipc3`
- Application: `setup-trino-ipc3`
- Application: `trino-trino-ipc3`
- Application: `vault-trino-ipc3`

## Troubleshooting

### ApplicationSet Not Generating Resources

**Problem**: ApplicationSet exists but no Applications are created

**Solution**:
1. Check ApplicationSet controller logs:
   ```bash
   kubectl logs -n argocd -l app.kubernetes.io/name=argocd-applicationset-controller --tail=100
   ```

2. Verify Git repository access:
   ```bash
   argocd repo list
   ```

3. Check cluster configuration file syntax:
   ```bash
   yamllint apps/values/clusters/*.yaml
   ```

### Application Stuck in Progressing State

**Problem**: Application shows "Progressing" status indefinitely

**Solution**:
1. Check Application details:
   ```bash
   argocd app get <app-name>
   ```

2. View sync status:
   ```bash
   kubectl describe application <app-name> -n argocd
   ```

3. Check for resource conflicts or permission issues

### Namespace Not Created

**Problem**: AppProject created but namespace missing

**Solution**:
1. Check the setup Application:
   ```bash
   kubectl get application setup-<cluster-name> -n argocd -o yaml
   ```

2. Verify kustomize patches in appprojects.yaml

3. Check Application controller logs:
   ```bash
   kubectl logs -n argocd -l app.kubernetes.io/name=argocd-application-controller --tail=100
   ```

### Sync Failures

**Problem**: Applications fail to sync

**Solution**:
1. Check AppProject permissions:
   ```bash
   kubectl get appproject <project-name> -n argocd -o yaml
   ```

2. Verify destination namespace exists

3. Check Helm values file paths are correct

4. Review Application events:
   ```bash
   kubectl describe application <app-name> -n argocd
   ```

## Maintenance

### Updating Chart Versions

To update Trino or Vault chart versions:

1. Edit cluster configuration file:
   ```yaml
   applications:
     trino:
       chart:
         version: 0.20.0  # Update version
   ```

2. Commit and push changes

3. ArgoCD will automatically sync the new version

### Disabling an Application

To disable Trino or Vault for a specific cluster:

```yaml
applications:
  trino:
    enabled: false  # Disable Trino
```

### Removing a Cluster

1. Delete the cluster configuration file:
   ```bash
   git rm apps/values/clusters/trino-ipc3.yaml
   git rm apps/values/trino/trino-ipc3-values.yaml
   git rm apps/values/vault/trino-ipc3-values.yaml
   ```

2. Commit and push:
   ```bash
   git commit -m "Remove trino-ipc3 cluster"
   git push
   ```

3. ArgoCD will automatically prune the resources

## Best Practices

1. **Always test in non-production first**
2. **Use Git branches for major changes**
3. **Monitor ArgoCD sync status regularly**
4. **Keep Helm values files well-documented**
5. **Use consistent naming conventions**
6. **Regular backups of cluster configurations**
7. **Review ApplicationSet controller logs periodically**

## Security Considerations

1. **Repository Access**: Use SSH keys or tokens with minimal required permissions
2. **AppProject Permissions**: Review and restrict as needed
3. **Secrets Management**: Use Vault or sealed-secrets for sensitive data
4. **RBAC**: Configure appropriate ArgoCD RBAC policies
5. **Network Policies**: Implement network policies for Trino and Vault

## Support

For issues or questions:
1. Check ApplicationSet and Application controller logs
2. Review cluster configuration files
3. Consult ArgoCD documentation: https://argo-cd.readthedocs.io/

# Made with Bob