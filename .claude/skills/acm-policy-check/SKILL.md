---
name: acm-policy-check
description: Audit ACM Policy CRDs, PlacementBindings, and Placements for correctness and consistency
disable-model-invocation: true
allowed-tools:
  - Bash
  - Read
---

# acm-policy-check

Audits ACM governance resources for structural correctness and cross-reference integrity.

## Checks Performed

Run the following checks against all YAML files in the `acm/` directory:

### 1. Policy Structure Validation
```bash
# Verify every Policy has at least one policy-template
grep -l "kind: Policy" acm/*.yaml | while read f; do
  echo "=== $f ==="
  # Check for policy-templates key
  grep -c "policy-templates:" "$f" || echo "FAIL: Missing policy-templates in $f"
  # Check remediationAction is set
  grep "remediationAction:" "$f" || echo "FAIL: Missing remediationAction in $f"
done
```

### 2. Placement Binding Integrity
```bash
# For each PlacementBinding, verify the referenced Placement and Policy exist in the same file or directory
grep -A5 "kind: PlacementBinding" acm/*.yaml
```

Verify:
- `placementRef.name` matches an existing `Placement` resource name.
- `subjects[].name` matches an existing `Policy` resource name.
- All three share the same `namespace`.

### 3. ConfigurationPolicy Uniqueness
```bash
# Check for duplicate ConfigurationPolicy names across files
grep -h "name:" acm/*.yaml | grep -A1 "kind: ConfigurationPolicy" | sort | uniq -d
```

Flag any `ConfigurationPolicy` name that appears in multiple parent Policies targeting the same clusters.

### 4. Namespace Consistency
Verify all resources in a policy set use the same namespace (default: `open-cluster-management-global-set`).

### 5. Remediation Action Hierarchy
Verify no `ConfigurationPolicy.spec.remediationAction` exceeds its parent `Policy.spec.remediationAction` (e.g., child `enforce` under parent `inform` is invalid).

## Output

Print a summary of all checks with PASS/FAIL status and details for any failures.
