# ACM — Advanced Cluster Management

Guidelines for authoring and reviewing ACM governance resources in this directory.

## API Group

All policy resources use `policy.open-cluster-management.io/v1`.
Placement resources use `cluster.open-cluster-management.io/v1beta1`.

## Resource Structure

Every ACM policy deployment requires three resources:

1. **Policy** — wraps one or more `ConfigurationPolicy` templates.
2. **Placement** — selects target clusters via label selectors.
3. **PlacementBinding** — binds a Placement to a Policy.

All three must share the same namespace (default: `open-cluster-management-global-set`).

## Naming Conventions

- Policy: `policy-<domain>-<purpose>` (e.g., `policy-gatekeeper-setup`).
- Placement: `<policy-name>-placement`.
- PlacementBinding: `<policy-name>-placement-binding`.
- ConfigurationPolicy: descriptive slug of what it manages (e.g., `disallow-latest-tag-logic`).

## Remediation Action Rules

| Action | When to Use |
|--------|-------------|
| `inform` | Audit-only: report compliance without modifying cluster state. Use during initial rollout or for visibility policies. |
| `enforce` | Active remediation: create or update resources to match desired state. Use once the policy is validated and approved. |

- The parent `Policy.spec.remediationAction` sets the ceiling; a `ConfigurationPolicy` cannot escalate beyond it.
- Never set a new policy to `enforce` without a prior `inform` validation pass or an explicit dry-run.

## ConfigurationPolicy Rules

- Each ConfigurationPolicy should manage a single concern.
- Use `complianceType: musthave` for resources that must exist.
- Use `complianceType: mustnothave` for resources that must be absent.
- Never duplicate a ConfigurationPolicy name across multiple parent Policies targeting the same cluster — this creates competing ownership.

## Gatekeeper Integration

- ConstraintTemplates go in a dedicated ConfigurationPolicy (e.g., `disallow-latest-tag-logic`).
- Constraints go in a separate ConfigurationPolicy with `extraDependencies` on the template policy, ensuring the CRD exists before the constraint is applied.
- Gatekeeper `enforcementAction` values: `deny` (block admission), `dryrun` (audit only), `warn` (admit with warning).

## Namespace Exclusions

System namespaces must be excluded from Gatekeeper constraints:

```yaml
excludedNamespaces:
  - kube-*
  - openshift-*
  - stackrox
  - open-cluster-management*
```
