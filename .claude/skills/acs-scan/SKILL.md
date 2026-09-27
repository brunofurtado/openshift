---
name: acs-scan
description: Run roxctl deployment check against local deployment manifests for ACS policy compliance
disable-model-invocation: true
allowed-tools:
  - Bash
  - Read
---

# acs-scan

Runs ACS/StackRox policy checks against local Kubernetes deployment manifests.

## Prerequisites

- `roxctl` CLI must be installed and available in PATH.
- Environment variables `ROX_CENTRAL_ADDRESS` and `ROX_API_TOKEN` must be set.

## Steps

### 1. Verify Environment
```bash
# Check roxctl is available
command -v roxctl >/dev/null 2>&1 || { echo "ERROR: roxctl not found in PATH"; exit 1; }

# Check required environment variables
[ -z "$ROX_CENTRAL_ADDRESS" ] && echo "ERROR: ROX_CENTRAL_ADDRESS not set" && exit 1
[ -z "$ROX_API_TOKEN" ] && echo "ERROR: ROX_API_TOKEN not set" && exit 1
```

### 2. Scan Deployment Manifests
For each deployment manifest provided (or all YAML files containing `kind: Deployment`):

```bash
# Find deployment manifests
grep -rl "kind: Deployment" . --include="*.yaml" --include="*.yml" | while read f; do
  echo "=== Scanning: $f ==="
  roxctl deployment check --file "$f" 2>&1
  echo ""
done
```

### 3. Scan Container Images
For any image references found in the manifests:

```bash
# Extract image references and scan
grep -h "image:" <manifest.yaml> | awk '{print $2}' | sort -u | while read img; do
  echo "=== Image: $img ==="
  roxctl image check --image "$img" 2>&1
done
```

## Output

Report each manifest and image scan result with:
- Policy violations found (name, severity, description)
- Recommended remediation actions
- Overall pass/fail status
