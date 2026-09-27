---
name: oc-validate
description: Lint and dry-run OpenShift/Kubernetes YAML manifests using oc apply --dry-run=client
allowed-tools:
  - Bash
  - Read
---

# oc-validate

Validates OpenShift and Kubernetes YAML manifests for correctness before cluster application.

## When to Use

Run this skill on any YAML manifest before applying it to a cluster, or when reviewing changes to resource definitions.

## Steps

1. Identify all YAML files to validate. If a specific file was provided, use that. Otherwise, find all modified YAML files via `git diff --name-only` filtered to `.yaml`/`.yml` extensions.

2. For each YAML file, run syntax validation:
   ```bash
   oc apply --dry-run=client -f <file> 2>&1
   ```

3. If `oc` is not available, fall back to `kubectl apply --dry-run=client -f <file>`.

4. Report results as a summary table:
   | File | Status | Details |
   |------|--------|---------|

5. If any file fails validation, show the full error output and suggest a fix.

## Notes

- This skill only performs client-side validation. It does not connect to a cluster.
- Multi-document YAML files (with `---` separators) are validated as a whole.
- CRDs that are not present locally may cause false negatives — note this in the output if CRD-based resources are detected.
