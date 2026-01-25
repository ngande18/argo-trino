# Trino Chart Storage: .tgz vs Untar Comparison

## Overview

ArgoCD supports both approaches for storing Helm charts in Git. Here's a detailed comparison to help you choose.

## Approach 1: .tgz Files (Current Implementation)

### Structure
```
charts/
├── trino-0.19.0.tgz
├── trino-0.20.0.tgz
└── trino-0.21.0.tgz
```

### Advantages
✅ **Smaller Git Repository** - Binary .tgz files are compressed  
✅ **Multiple Versions** - Easy to maintain multiple versions side-by-side  
✅ **Atomic Updates** - Single file to add/remove  
✅ **Official Format** - Same format as Helm repositories  
✅ **Integrity** - Checksums built into .tgz format  
✅ **Simple Cleanup** - Just delete old .tgz files

### Disadvantages
❌ **No Diff Visibility** - Can't see what changed between versions in Git  
❌ **Binary Files** - Git doesn't handle binary files efficiently  
❌ **No Direct Editing** - Can't modify templates directly  
❌ **Merge Conflicts** - Binary files can't be merged

### Best For
- Production environments where chart stability is critical
- Teams that don't need to customize upstream charts
- When you want to maintain multiple chart versions
- When Git repository size is a concern

## Approach 2: Untar (Extracted Charts)

### Structure
```
charts/
└── trino/
    ├── Chart.yaml
    ├── values.yaml
    ├── templates/
    │   ├── deployment.yaml
    │   ├── service.yaml
    │   └── ...
    └── README.md
```

### Advantages
✅ **Full Visibility** - See all chart files and changes in Git diffs  
✅ **Direct Customization** - Modify templates, values, or Chart.yaml directly  
✅ **Better Code Review** - Review template changes line-by-line  
✅ **Easy Debugging** - Inspect templates without extracting  
✅ **Git-Friendly** - Text files work better with Git  
✅ **Patch Friendly** - Can apply patches or overlays easily

### Disadvantages
❌ **Larger Git Repository** - Many text files take more space  
❌ **Single Version** - Harder to maintain multiple versions  
❌ **More Files to Manage** - Hundreds of files vs one .tgz  
❌ **Drift Risk** - Easy to accidentally modify upstream files  
❌ **Complex Updates** - Need to replace entire directory structure

### Best For
- Development environments where you need to customize charts
- Teams that maintain chart forks or patches
- When you need to review every template change
- When you only need one chart version at a time

## Hybrid Approach (Recommended for Many Teams)

### Structure
```
charts/
├── upstream/                    # Original .tgz files
│   ├── trino-0.19.0.tgz
│   └── trino-0.20.0.tgz
└── trino/                       # Extracted and customized
    ├── Chart.yaml
    ├── values.yaml
    ├── templates/
    └── patches/                 # Your customizations
        └── custom-config.yaml
```

### How It Works
1. Store original .tgz files in `charts/upstream/` for reference
2. Extract and customize in `charts/trino/`
3. Document customizations in `charts/trino/patches/`
4. Use extracted version for deployments

### Advantages
✅ **Best of Both Worlds** - Visibility + version tracking  
✅ **Clear Customizations** - Easy to see what you changed  
✅ **Upgrade Path** - Compare new .tgz with your customizations  
✅ **Rollback Safety** - Keep original .tgz files

## Recommendation by Use Case

### Use .tgz Files If:
- ✅ You use charts as-is without modifications
- ✅ You need multiple versions simultaneously
- ✅ Git repository size is a concern
- ✅ You want simple version management
- ✅ You trust upstream chart releases

### Use Untar If:
- ✅ You need to customize chart templates
- ✅ You want full visibility in code reviews
- ✅ You maintain chart forks or patches
- ✅ You need to debug template issues frequently
- ✅ You only need one version at a time

### Use Hybrid If:
- ✅ You customize charts but want to track upstream
- ✅ You need clear upgrade paths
- ✅ You want to document all changes
- ✅ You need both flexibility and traceability

## Implementation Examples

### Current (.tgz) Implementation
```yaml
# ApplicationSet
source:
  repoURL: '{{.git.repoURL}}'
  chart: '{{.applications.trino.chart.path}}'  # charts/trino-0.19.0.tgz

# Deployment Config
applications:
  trino:
    chart:
      path: charts/trino-0.19.0.tgz
```

### Untar Implementation
```yaml
# ApplicationSet
source:
  repoURL: '{{.git.repoURL}}'
  path: charts/trino  # Directory path

# Deployment Config
applications:
  trino:
    chart:
      path: charts/trino  # No version in path
```

### Hybrid Implementation
```yaml
# ApplicationSet
source:
  repoURL: '{{.git.repoURL}}'
  path: charts/trino  # Use customized version

# Deployment Config
applications:
  trino:
    chart:
      path: charts/trino
      # Reference original: charts/upstream/trino-0.19.0.tgz
```

## Migration Between Approaches

### From .tgz to Untar
```bash
# Extract the .tgz file
tar -xzf charts/trino-0.19.0.tgz -C charts/

# Update deployment configs to use directory path
# Remove .tgz files if desired
```

### From Untar to .tgz
```bash
# Package the directory
helm package charts/trino -d charts/

# Update deployment configs to use .tgz path
# Remove extracted directory if desired
```

## Conclusion

**Current Implementation (.tgz)** is recommended for most production use cases because:
- Simple version management
- Multiple versions support
- Smaller Git repository
- Clear separation between versions

**Switch to Untar** if you need to:
- Customize chart templates
- Review changes in detail
- Debug template issues
- Maintain chart forks

Both approaches are valid and supported by ArgoCD. Choose based on your team's needs and workflow.