# ACM Placement Configuration and Hub-and-Spoke Architecture

## Hub-and-Spoke Architecture

Red Hat Advanced Cluster Management uses a hub-and-spoke model where a central hub cluster governs one or more managed (spoke) clusters. Each layer has distinct responsibilities and components.

### Component Distribution

| Component | Hub | Spoke | Installed By |
|-----------|:---:|:-----:|--------------|
| ACM Operator (MultiClusterHub) | Yes | No | Cluster admin via OperatorHub |
| Klusterlet Agent | No | Yes | ACM automatically on cluster import |
| Governance Policy Framework Agent | No | Yes | ACM automatically via klusterlet |
| Gatekeeper Operator | Optional | Yes (if using admission policies) | Manually or via ACM enforce policy |
| StackRox Sensor (ACS) | No | Yes | Central or via ACM policy |
| StackRox Central (ACS) | Yes | No | Cluster admin via OperatorHub |

### Hub Cluster

The hub runs the full ACM operator (`MultiClusterHub` CR) and is responsible for:

- Defining and distributing governance policies.
- Managing the lifecycle of managed clusters (import, detach, destroy).
- Aggregating compliance status from all spokes.
- Hosting the ACM console for multi-cluster visibility.

The hub does **not** need Gatekeeper installed unless it is also a target of its own policies (e.g., `local-cluster=true`).

### Spoke Clusters

Spoke clusters are lightweight — they only run agents, not the full ACM operator. When a cluster is imported into ACM:

1. **Klusterlet** is deployed automatically. It establishes a secure channel back to the hub and receives policy definitions.
2. **Governance Policy Framework** agents are installed by the klusterlet. They evaluate `ConfigurationPolicy` templates locally and report compliance status to the hub.
3. **Gatekeeper** (if needed) must be installed separately on each spoke. This can be automated by creating an ACM policy with `remediationAction: enforce` that deploys the Gatekeeper operator — the same approach used in this lab's `policy-gatekeeper-setup.yaml`.

No ACM operator license or installation is required on spoke clusters.

### Communication Flow

```
┌──────────────────────────────────────────────────┐
│                   Hub Cluster                     │
│                                                   │
│   ACM Operator ──► Policy Templates               │
│        │              │                            │
│        │              ▼                            │
│        │         PlacementBinding                  │
│        │              │                            │
│        ▼              ▼                            │
│   Compliance    Placement ──► selects clusters     │
│   Dashboard          │       by label match        │
│        ▲             │                             │
└────────┼─────────────┼─────────────────────────────┘
         │             │
    status reports     policy distribution
         │             │
┌────────┼─────────────┼─────────────────────────────┐
│        │             ▼          Spoke Cluster       │
│        │        Klusterlet                          │
│        │             │                              │
│        │             ▼                              │
│        └──── Policy Framework Agent                 │
│                      │                              │
│                      ▼                              │
│              ConfigurationPolicy                    │
│              (evaluate & enforce)                    │
│                      │                              │
│                      ▼                              │
│              Gatekeeper (admission control)          │
└─────────────────────────────────────────────────────┘
```

---

## Placement Configuration

The `Placement` resource determines which managed clusters receive a given policy. Clusters are selected by matching labels set on the `ManagedCluster` resource on the hub.

### Labeling Managed Clusters

Before configuring placements, label your managed clusters on the hub:

```bash
# Add environment and region labels
oc label managedcluster <cluster-name> env=production
oc label managedcluster <cluster-name> region=us-east-1

# Verify labels
oc get managedcluster <cluster-name> --show-labels
```

### Placement Patterns

#### Target a Single Cluster (Current Lab Default)

Selects only the hub's own local cluster:

```yaml
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: Placement
metadata:
  name: policy-placement
  namespace: open-cluster-management-global-set
spec:
  predicates:
    - requiredClusterSelector:
        labelSelector:
          matchExpressions:
            - key: local-cluster
              operator: In
              values:
                - "true"
```

#### Target by Environment Label

Selects clusters labeled with a specific environment:

```yaml
spec:
  predicates:
    - requiredClusterSelector:
        labelSelector:
          matchLabels:
            env: production
```

#### Target Multiple Environments

Selects clusters matching any of several values:

```yaml
spec:
  predicates:
    - requiredClusterSelector:
        labelSelector:
          matchExpressions:
            - key: env
              operator: In
              values:
                - "production"
                - "staging"
```

#### Target All Managed Clusters

Removes selection criteria to match every cluster. Tolerations ensure policies reach clusters even if they are temporarily unreachable:

```yaml
spec:
  predicates: []
  tolerations:
    - key: cluster.open-cluster-management.io/unreachable
      operator: Exists
    - key: cluster.open-cluster-management.io/unavailable
      operator: Exists
```

#### Exclude Specific Clusters

Uses `NotIn` to skip clusters with certain labels:

```yaml
spec:
  predicates:
    - requiredClusterSelector:
        labelSelector:
          matchExpressions:
            - key: env
              operator: NotIn
              values:
                - "sandbox"
```

#### Combine Multiple Criteria (AND Logic)

Multiple expressions within the same predicate are ANDed — a cluster must match all of them:

```yaml
spec:
  predicates:
    - requiredClusterSelector:
        labelSelector:
          matchExpressions:
            - key: env
              operator: In
              values:
                - "production"
            - key: region
              operator: In
              values:
                - "us-east-1"
                - "eu-west-1"
```

#### Multiple Predicates (OR Logic)

Multiple predicates are ORed — a cluster matching any predicate is selected:

```yaml
spec:
  predicates:
    - requiredClusterSelector:
        labelSelector:
          matchLabels:
            env: production
    - requiredClusterSelector:
        labelSelector:
          matchLabels:
            env: dr-site
```

### Supported Operators

| Operator | Behavior |
|----------|----------|
| `In` | Label value must be one of the listed values |
| `NotIn` | Label value must not be any of the listed values |
| `Exists` | Label key must be present (value ignored) |
| `DoesNotExist` | Label key must not be present |

### Verifying Placement Decisions

After applying a placement, verify which clusters were selected:

```bash
# List all placement decisions
oc get placementdecisions -n open-cluster-management-global-set

# Describe a specific placement to see matched clusters
oc describe placement <placement-name> -n open-cluster-management-global-set

# Check the decision details
oc get placementdecisions -n open-cluster-management-global-set -o yaml
```

### Tolerations

Tolerations ensure policies continue to apply to clusters that become temporarily unreachable or unavailable. Always include them for production placements:

```yaml
spec:
  tolerations:
    - key: cluster.open-cluster-management.io/unreachable
      operator: Exists
    - key: cluster.open-cluster-management.io/unavailable
      operator: Exists
```

Without tolerations, policies are withdrawn from clusters that lose connectivity, which may not be the desired behavior for security policies.
