# Adding Custom Templates on Top of .tgz Charts

## Problem

You want to:
- Keep upstream charts as `.tgz` files (for version control)
- Add custom Kubernetes resources on top
- Avoid extracting the `.tgz` file

## Solution: Wrapper Chart Approach

Create a wrapper chart that depends on the upstream `.tgz` chart and adds custom templates.

### Directory Structure

```
charts/
├── trino-0.19.0.tgz                 # Upstream chart (dependency)
├── trino-0.20.0.tgz                 # Another version
└── trino-wrapper/                   # Wrapper chart
    ├── Chart.yaml                   # Wrapper chart metadata
    ├── values.yaml                  # Default values
    ├── charts/                      # Dependencies (symlink or copy)
    │   └── trino-0.19.0.tgz        # Symlink to ../trino-0.19.0.tgz
    └── templates/                   # Your custom templates
        ├── NOTES.txt
        ├── custom-configmap.yaml
        ├── custom-service-monitor.yaml
        ├── custom-network-policy.yaml
        └── custom-pdb.yaml
```

## Implementation

### Step 1: Create Wrapper Chart

```bash
# Create wrapper chart structure
mkdir -p charts/trino-wrapper/templates
mkdir -p charts/trino-wrapper/charts

# Create Chart.yaml
cat > charts/trino-wrapper/Chart.yaml << 'EOF'
apiVersion: v2
name: trino-wrapper
description: Trino with custom templates
type: application
version: 1.0.0
appVersion: "0.19.0"

dependencies:
  - name: trino
    version: "0.19.0"
    repository: "file://../trino-0.19.0.tgz"
EOF

# Create values.yaml
cat > charts/trino-wrapper/values.yaml << 'EOF'
# Upstream Trino values (passed through to dependency)
trino:
  coordinator:
    resources:
      requests:
        memory: "2Gi"
        cpu: "1"

# Custom template values
customConfig:
  enabled: true
  properties:
    query.max-memory: "50GB"

monitoring:
  serviceMonitor:
    enabled: true
    interval: "30s"

networkPolicy:
  enabled: true

podDisruptionBudget:
  enabled: true
  minAvailable: 1
EOF
```

### Step 2: Add Custom Templates

```yaml
# charts/trino-wrapper/templates/custom-configmap.yaml
{{- if .Values.customConfig.enabled }}
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "trino.fullname" .Subcharts.trino }}-custom
  namespace: {{ .Release.Namespace }}
  labels:
    app.kubernetes.io/name: trino
    app.kubernetes.io/instance: {{ .Release.Name }}
data:
  custom-settings.properties: |
    {{- range $key, $value := .Values.customConfig.properties }}
    {{ $key }}={{ $value }}
    {{- end }}
{{- end }}
```

```yaml
# charts/trino-wrapper/templates/custom-service-monitor.yaml
{{- if .Values.monitoring.serviceMonitor.enabled }}
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: {{ .Release.Name }}-trino
  namespace: {{ .Release.Namespace }}
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: trino
      app.kubernetes.io/instance: {{ .Release.Name }}
  endpoints:
    - port: http
      interval: {{ .Values.monitoring.serviceMonitor.interval }}
{{- end }}
```

### Step 3: Build Dependencies

```bash
# Update dependencies (downloads/links the .tgz file)
cd charts/trino-wrapper
helm dependency update
cd ../..

# Or manually create symlink
ln -s ../../trino-0.19.0.tgz charts/trino-wrapper/charts/trino-0.19.0.tgz
```

### Step 4: Update Deployment Configuration

```yaml
# apps/values/trino-deployments/trino-ipc1.yaml
applications:
  trino:
    enabled: true
    namespace: trino-ipc1
    chart:
      path: charts/trino-wrapper  # Use wrapper chart
    valuesFile: apps/values/trino/trino-ipc1-values.yaml
```

### Step 5: Update ApplicationSet

```yaml
# apps/applicationsets/trino/applicationset.yaml
source:
  repoURL: '{{.git.repoURL}}'
  targetRevision: '{{.git.targetRevision}}'
  path: '{{.applications.trino.chart.path}}'  # charts/trino-wrapper
  helm:
    releaseName: trino
    valueFiles:
      - ../../{{.applications.trino.valuesFile}}
```

## Alternative: Kustomize Post-Renderer

Use Kustomize to add resources on top of the Helm chart:

### Directory Structure

```
charts/
├── trino-0.19.0.tgz
└── trino-kustomize/
    ├── base/
    │   └── kustomization.yaml
    └── overlays/
        └── custom/
            ├── kustomization.yaml
            ├── custom-configmap.yaml
            └── custom-service-monitor.yaml
```

### Implementation

```yaml
# charts/trino-kustomize/base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

helmCharts:
  - name: trino
    repo: file://../../trino-0.19.0.tgz
    releaseName: trino
    namespace: trino-ipc1
    valuesFile: ../../../apps/values/trino/trino-ipc1-values.yaml
```

```yaml
# charts/trino-kustomize/overlays/custom/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base
  - custom-configmap.yaml
  - custom-service-monitor.yaml
```

```yaml
# ArgoCD Application with Kustomize
apiVersion: argoproj.io/v1alpha1
kind: Application
spec:
  source:
    repoURL: '{{.git.repoURL}}'
    targetRevision: '{{.git.targetRevision}}'
    path: charts/trino-kustomize/overlays/custom
```

## Comparison

### Wrapper Chart Approach

**Pros:**
✅ Pure Helm solution
✅ Helm dependency management
✅ Easy to version wrapper chart
✅ Can override upstream values
✅ Native Helm templating

**Cons:**
❌ Need to update Chart.yaml for version changes
❌ Slightly more complex structure
❌ Dependency management overhead

### Kustomize Approach

**Pros:**
✅ Simple resource addition
✅ No Chart.yaml maintenance
✅ Flexible patching
✅ ArgoCD native support

**Cons:**
❌ Mixed Helm/Kustomize
❌ Less Helm-native
❌ Harder to test locally

## Recommended: Wrapper Chart

The wrapper chart approach is recommended because:
1. Pure Helm solution
2. Better integration with Helm ecosystem
3. Easier to test with `helm template`
4. Clear dependency management

## Upgrading Upstream Chart

### With Wrapper Chart

```bash
# 1. Pull new upstream version
helm pull trinodb/trino --version 0.20.0 --destination charts/

# 2. Update wrapper Chart.yaml
cat > charts/trino-wrapper/Chart.yaml << 'EOF'
apiVersion: v2
name: trino-wrapper
version: 1.1.0  # Bump wrapper version
appVersion: "0.20.0"

dependencies:
  - name: trino
    version: "0.20.0"  # Update version
    repository: "file://../trino-0.20.0.tgz"  # Update path
EOF

# 3. Update dependencies
cd charts/trino-wrapper
helm dependency update
cd ../..

# 4. Test
helm template test charts/trino-wrapper -f apps/values/trino/trino-ipc1-values.yaml

# 5. Commit
git add charts/trino-0.20.0.tgz charts/trino-wrapper/
git commit -m "Upgrade Trino to 0.20.0"
```

## Testing

```bash
# Test wrapper chart
helm template test charts/trino-wrapper \
  -f apps/values/trino/trino-ipc1-values.yaml \
  --debug

# Check custom resources
helm template test charts/trino-wrapper \
  -f apps/values/trino/trino-ipc1-values.yaml \
  | grep -A 10 "kind: ServiceMonitor"
```

## Benefits

✅ **Keep .tgz files** - Upstream charts remain as .tgz  
✅ **Add custom resources** - Full flexibility for custom templates  
✅ **Version control** - Both upstream and custom templates tracked  
✅ **Easy upgrades** - Update Chart.yaml dependency version  
✅ **Helm native** - Pure Helm solution, no mixed tools  
✅ **Reusable** - Same wrapper for multiple deployments

## Conclusion

The wrapper chart approach allows you to:
- Keep upstream charts as `.tgz` files
- Add custom Kubernetes resources
- Maintain clean separation
- Use pure Helm tooling

This is the best solution for your requirement of using `.tgz` files with custom templates on top.