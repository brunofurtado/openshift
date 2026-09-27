# ACM Gatekeeper - Disallow Latest Image Tag

Enforces a policy across managed clusters that blocks Deployments and Pods using the `latest` image tag (or no tag at all), using Red Hat Advanced Cluster Management (ACM) and OPA Gatekeeper.

## Prerequisites

- Red Hat ACM hub cluster with the Governance framework enabled
- OPA Gatekeeper operator already installed on the managed clusters
- The `open-cluster-management-global-set` namespace exists on the hub

## Files

### 1. `policy-gatekeeper-setup.yaml`

Sets up the Gatekeeper ConstraintTemplate. Contains:

| Resource | Name | Purpose |
|----------|------|---------|
| **Policy** | `policy-gatekeeper-setup` | Wrapper policy (remediationAction: `enforce`) |
| ConfigurationPolicy | `disallow-latest-tag-logic` | Creates the `ConstraintTemplate` with Rego logic that detects `latest` tags |
| Placement | `policy-gatekeeper-setup-placement` | Targets clusters with label `local-cluster=true` |
| PlacementBinding | `policy-gatekeeper-setup-placement-binding` | Binds the placement to the policy |

### 2. `policy-gk-constraint.yaml`

Deploys the Gatekeeper constraint with dependency ordering. Contains:

| Resource | Name | Purpose |
|----------|------|---------|
| **Policy** | `policy-gk-constraints` | Wrapper policy (remediationAction: `enforce`) |
| ConfigurationPolicy | `disallow-latest-tag-constraint` | Creates the constraint after the ConstraintTemplate is compliant (via `extraDependencies`) |
| Placement | `policy-gk-constraints-placement` | Targets clusters with label `local-cluster=true` |
| PlacementBinding | `policy-gk-constraints-placement-binding` | Binds the placement to the policy |

## Execution Order

```
1. Apply policy-gatekeeper-setup.yaml
   └── Creates the ConstraintTemplate

2. Apply policy-gk-constraint.yaml
   └── Waits for ConstraintTemplate to be Compliant (extraDependencies)
   └── Then creates the Constraint instance
```

### Apply

```bash
oc apply -f policy-gatekeeper-setup.yaml
oc apply -f policy-gk-constraint.yaml
```

### Verify

```bash
# Check policy compliance on the hub
oc get policy -n open-cluster-management-global-set

# Check the ConstraintTemplate exists on the managed cluster
oc get constrainttemplate k8sdisallowlatesttag

# Check the Constraint exists on the managed cluster
oc get k8sdisallowlatesttag no-latest-tags-in-deployments
```

## What Gets Blocked

Once active, the admission webhook will **deny** any Deployment or Pod where a container image:
- Uses the `:latest` tag explicitly (e.g., `nginx:latest`)
- Omits a tag entirely (e.g., `nginx`)

### Excluded Namespaces

System namespaces are excluded from enforcement:
- `kube-*`
- `openshift-*`
- `stackrox`
- `open-cluster-management*`
