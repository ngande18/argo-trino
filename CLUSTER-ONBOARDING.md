# Cluster Onboarding Guide

This guide explains how to onboard a new Trino cluster to the ArgoCD deployment system.

## Overview

Adding a new cluster is a simple process that involves creating configuration files and optionally using feature branches for testing before production deployment.

## Onboarding Options

### Option 1: Direct Production Deployment
Add cluster directly to production using the main/HEAD branch.

### Option 2: Test First, Then Promote
Test the cluster configuration in a feature branch before promoting to production.

---

## Option 1: Direct Production Deployment

Use this approach when you're confident in the configuration or deploying to a non-critical environment.

### Step 1: Create Cluster Configuration

```bash
git checkout main

# Create cluster configuration file
cat > apps/values/clusters/cdaitrino-e3h.yaml <<EOF
cluster:
  name: cdaitrino-e3h
  server: https://kubernetes.default.svc

# Use main branch for production
git:
  repoURL: https://github.com/your-org/your-repo.git
  targetRevision: HEAD

appProject:
  name: cdaitrino-e3h
  description: AppProject for cdaitrino-e3h cluster
  sourceRepos:
    - '*'
  destinations:
    - namespace: cdaitrino-e3h
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

applications:
  trino:
    enabled: true
    namespace: cdaitrino-e3h
    chart:
      repoURL: https://trinodb.github.io/charts
      name: trino
      version: 0.19.0
    valuesFile: apps/values/trino/cdaitrino-e3h-values.yaml
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

  vault:
    enabled: true
    namespace: vault
    chart:
      repoURL: https://helm.releases.hashicorp.com
      name: vault
      version: 0.28.0
    valuesFile: apps/values/vault/cdaitrino-e3h-values.yaml
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

valuesRepo:
  url: https://github.com/your-org/your-repo.git
  targetRevision: HEAD
EOF
```

### Step 2: Create Helm Values Files

```bash
# Create Trino values
cat > apps/values/trino/cdaitrino-e3h-values.yaml <<EOF
# Trino configuration for cdaitrino-e3h
coordinator:
  resources:
    requests:
      memory: "2Gi"
      cpu: "1"
# ... your configuration
EOF

# Create Vault values
cat > apps/values/vault/cdaitrino-e3h-values.yaml <<EOF
# Vault configuration for cdaitrino-e3h
server:
  ha:
    enabled: true
# ... your configuration
EOF
```

### Step 3: Commit and Deploy

```bash
git add apps/values/clusters/cdaitrino-e3h.yaml
git add apps/values/trino/cdaitrino-e3h-values.yaml
git add apps/values/vault/cdaitrino-e3h-values.yaml
git commit -m "Add cdaitrino-e3h cluster"
git push origin main
```

### Step 4: Verify Deployment

```bash
# Wait 3 minutes for ArgoCD to sync
sleep 180

# Check applications
kubectl get applications -n argocd | grep cdaitrino-e3h

# Check resources
kubectl get namespace cdaitrino-e3h
kubectl get appproject cdaitrino-e3h -n argocd
kubectl get pods -n cdaitrino-e3h
```

---

## Option 2: Test First, Then Promote

Use this approach for production clusters or when testing new configurations.

### Step 1: Create Cluster Configuration on Main (Points to Feature Branch)

**Important**: The cluster configuration file MUST be on the main/HEAD branch.

```bash
git checkout main

# Create cluster configuration that POINTS to feature branch
cat > apps/values/clusters/cdaitrino-e3h.yaml <<EOF
cluster:
  name: cdaitrino-e3h
  server: https://kubernetes.default.svc

# Point to feature branch for testing
git:
  repoURL: https://github.com/your-org/your-repo.git
  targetRevision: feature/cdaitrino-e3h-testing  # Feature branch!

appProject:
  name: cdaitrino-e3h
  description: AppProject for cdaitrino-e3h cluster (TEST)
  # ... rest of configuration
EOF

# Commit and push to main
git add apps/values/clusters/cdaitrino-e3h-test.yaml
git commit -m "Add cdaitrino-e3h test configuration pointing to feature branch"
git push origin main
```

### 2. Create Feature Branch with Test Resources

Now create the feature branch with your test configurations:

```bash
# Create and checkout feature branch
git checkout -b feature/cdaitrino-e3h-testing

# Create test values files
mkdir -p apps/values/trino apps/values/vault

# Create Trino values for cdaitrino-e3h
cat > apps/values/trino/cdaitrino-e3h-values.yaml <<EOF
# Trino configuration for cdaitrino-e3h testing
# ... your test configuration ...
EOF

# Create Vault values for cdaitrino-e3h
cat > apps/values/vault/cdaitrino-e3h-values.yaml <<EOF
# Vault configuration for cdaitrino-e3h testing
# ... your test configuration ...
EOF

# Make any other changes you want to test
# (bootstrap templates, etc.)

# Commit and push feature branch
git add .
git commit -m "Add cdaitrino-e3h test resources"
git push origin feature/cdaitrino-e3h-testing
```

### 3. How It Works

```
┌──────────────────────────────────────────────────────────────────┐
│                         Git Repository                            │
│                                                                   │
│  main branch (HEAD) ← Root-app reads from here                   │
│  ├── apps/root-app/application.yaml                              │
│  ├── apps/applicationsets/                                       │
│  │   ├── appprojects.yaml                                        │
│  │   ├── trino/applicationset.yaml                               │
│  │   └── vault/applicationset.yaml                               │
│  └── apps/values/clusters/  ← Cluster configs MUST be here       │
│      ├── trino-ipc1.yaml  → git.targetRevision: HEAD             │
│      ├── trino-ipc2.yaml  → git.targetRevision: HEAD             │
│      └── cdaitrino-e3h-test.yaml → git.targetRevision:              │
│                                  feature/cdaitrino-e3h-testing         │
│                                                                   │
│  feature/cdaitrino-e3h-testing branch ← Test resources here           │
│  ├── apps/bootstrap/  (test changes)                             │
│  ├── apps/values/trino/cdaitrino-e3h-values.yaml (test)            │
│  └── apps/values/vault/cdaitrino-e3h-values.yaml (test)            │
└──────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌──────────────────────────────────────────────────────────────────┐
│              ArgoCD Root App (reads main/HEAD)                    │
│                                                                   │
│  1. Reads ApplicationSets from main branch                       │
│  2. ApplicationSets read cluster configs from main branch:       │
│     ├── trino-ipc1.yaml  → Creates apps using HEAD              │
│     ├── trino-ipc2.yaml  → Creates apps using HEAD              │
│     └── cdaitrino-e3h-test.yaml → Creates apps using               │
│                                 feature/cdaitrino-e3h-testing         │
└──────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌──────────────────────────────────────────────────────────────────┐
│                   Generated Applications                          │
│                                                                   │
│  setup-trino-ipc1  → Reads bootstrap from HEAD                   │
│  setup-trino-ipc2  → Reads bootstrap from HEAD                   │
│  setup-cdaitrino-e3h  → Reads bootstrap from feature branch         │
│                                                                   │
│  trino-trino-ipc1  → Reads values from HEAD                      │
│  trino-trino-ipc2  → Reads values from HEAD                      │
│  trino-cdaitrino-e3h  → Reads values from feature branch            │
└──────────────────────────────────────────────────────────────────┘

KEY POINT: Cluster config files are ALWAYS read from main/HEAD branch.
           Only the resources they point to can be in feature branches.
```

### 4. Understanding the Flow

**Critical Understanding**:
1. Root-app ALWAYS reads from main/HEAD branch
2. ApplicationSets ALWAYS read cluster configs from main/HEAD branch
3. Cluster configs (in main) specify which branch to use for resources
4. Only the resources (bootstrap, values) can be in feature branches

**What's in main branch**:
- Root-app configuration
- ApplicationSets
- Cluster configuration files (including cdaitrino-e3h-test.yaml)

**What's in feature branch**:
- Bootstrap templates (if testing changes)
- Trino values files
- Vault values files
- Any other resources you're testing

### 5. Testing Workflow

#### Make Changes in Feature Branch

```bash
git checkout feature/cdaitrino-e3h-testing

# Update configurations
vim apps/values/trino/cdaitrino-e3h-values.yaml
vim apps/bootstrap/templates/namespace.yaml  # if testing bootstrap changes

git add .
git commit -m "Test new Trino configuration"
git push origin feature/cdaitrino-e3h-testing
```

#### ArgoCD Automatically Syncs

Within 3 minutes, ArgoCD will:
1. Detect the change in the feature branch
2. Sync cdaitrino-e3h applications with the new configuration
3. Leave ipc1 and ipc2 unchanged (they use HEAD)

#### Verify Changes

```bash
# Check cdaitrino-e3h applications
kubectl get applications -n argocd | grep cdaitrino-e3h

# Check cdaitrino-e3h resources
kubectl get pods -n cdaitrino-e3h

# View application details
argocd app get trino-cdaitrino-e3h
```

### 6. Promoting to Production

Once testing is successful:

```bash
# Merge feature branch to main
git checkout main
git merge feature/cdaitrino-e3h-testing
git push origin main

# Update cdaitrino-e3h config to use HEAD
vim apps/values/clusters/cdaitrino-e3h-test.yaml
# Change: targetRevision: HEAD

git add apps/values/clusters/cdaitrino-e3h-test.yaml
git commit -m "Promote cdaitrino-e3h to production (use HEAD)"
git push origin main

# Optional: Delete feature branch
git branch -d feature/cdaitrino-e3h-testing
git push origin --delete feature/cdaitrino-e3h-testing
```

## Use Cases

### 1. Testing New Helm Chart Versions

```yaml
# In feature branch: apps/values/clusters/cdaitrino-e3h-test.yaml
git:
  targetRevision: feature/test-trino-v0.20

applications:
  trino:
    chart:
      version: 0.20.0  # New version
```

### 2. Testing Bootstrap Changes

```yaml
# Test changes to namespace or AppProject templates
git:
  targetRevision: feature/new-bootstrap-labels
```

### 3. Testing Configuration Changes

```yaml
# Test new Trino or Vault configurations
git:
  targetRevision: feature/high-memory-config
```

### 4. Canary Deployments

```yaml
# Cluster3 tests new config
git:
  targetRevision: feature/canary-deployment

# After validation, update ipc1
# Then update ipc2
# Progressive rollout
```

## Best Practices

### 1. Clear Branch Naming

Use descriptive branch names:
- `feature/cdaitrino-e3h-testing`
- `test/new-trino-version`
- `experiment/custom-bootstrap`

### 2. Document Test Clusters

Add comments in cluster config:
```yaml
# TEST CLUSTER - Uses feature branch
# Purpose: Testing Trino 0.20.0 upgrade
# Owner: team@example.com
# Created: 2024-01-24
git:
  targetRevision: feature/trino-upgrade
```

### 3. Separate Test Clusters

Use dedicated test clusters:
- `trino-ipc1` (production)
- `trino-ipc2` (production)
- `cdaitrino-e3h-test` (testing)
- `trino-ipc4-dev` (development)

### 4. Monitor Test Deployments

```bash
# Watch for sync issues
kubectl get applications -n argocd -w

# Check application health
argocd app list | grep cdaitrino-e3h

# View sync status
argocd app get setup-cdaitrino-e3h
```

### 5. Clean Up After Testing

```bash
# Remove test cluster config
git rm apps/values/clusters/cdaitrino-e3h-test.yaml

# Delete test resources
kubectl delete namespace cdaitrino-e3h

# Delete feature branch
git branch -d feature/cdaitrino-e3h-testing
git push origin --delete feature/cdaitrino-e3h-testing
```

## Advanced Scenarios

### Multiple Test Clusters on Different Branches

```yaml
# apps/values/clusters/trino-test1.yaml
git:
  targetRevision: feature/test-scenario-1

# apps/values/clusters/trino-test2.yaml
git:
  targetRevision: feature/test-scenario-2

# apps/values/clusters/trino-test3.yaml
git:
  targetRevision: feature/test-scenario-3
```

### Testing with Different Git Repositories

```yaml
# Test from a fork or different repo
git:
  repoURL: https://github.com/your-fork/your-repo.git
  targetRevision: experimental-features
```

### Pinning to Specific Commits

```yaml
# Pin to a specific commit for reproducibility
git:
  targetRevision: abc123def456  # Git commit SHA
```

## Troubleshooting

### Application Not Syncing

```bash
# Check ApplicationSet controller logs
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-applicationset-controller

# Verify branch exists
git ls-remote origin feature/cdaitrino-e3h-testing

# Check Application status
kubectl describe application setup-cdaitrino-e3h -n argocd
```

### Wrong Branch Being Used

```bash
# Verify cluster config
cat apps/values/clusters/cdaitrino-e3h-test.yaml | grep targetRevision

# Check Application source
argocd app get setup-cdaitrino-e3h -o yaml | grep targetRevision
```

### Changes Not Appearing

```bash
# Force refresh
argocd app get setup-cdaitrino-e3h --refresh

# Check sync status
argocd app sync setup-cdaitrino-e3h --dry-run
```

## Summary

✅ **Isolated Testing**: Test changes without affecting production
✅ **Per-Cluster Control**: Each cluster can use different branches
✅ **Safe Experimentation**: Easy to rollback by changing targetRevision
✅ **Progressive Rollout**: Test → Validate → Promote to production
✅ **Flexible**: Support multiple test scenarios simultaneously

# Made with Bob