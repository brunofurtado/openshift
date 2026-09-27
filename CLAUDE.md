# OpenShift Multi-Project Repository

## Workspace Tooling

Available CLI tools assumed present on the operator workstation:

- `oc` — OpenShift CLI for cluster operations and resource management
- `kustomize` — Kubernetes-native configuration management
- `helm` — Chart-based package management
- `tkn` — Tekton Pipelines CLI
- `roxctl` — Red Hat ACS / StackRox CLI for security scanning

## Safety Guardrails

### Dry-Run First Policy

**Every** `oc apply` or `oc create` command MUST be validated with `--dry-run=client`
before applying to any cluster. Generate the dry-run command, show the output, and
only proceed to the real apply when explicitly confirmed.

```bash
# Always run this first:
oc apply --dry-run=client -f <manifest.yaml>

# Only after successful dry-run and user confirmation:
oc apply -f <manifest.yaml>
```

### Resource Modifications

- Never delete or replace resources without confirming existing state via `oc get`.
- Prefer `oc apply` over `oc create` to allow idempotent updates.
- Never use `--force` on cluster operations unless explicitly requested.

### Secrets and Credentials

- Never embed credentials, tokens, or passwords in manifest files.
- Reference secrets via `SecretKeyRef`, `Secret` volumes, or external secret operators.
- Never commit `.env` files or `kubeconfig` files.

## Directory Map

| Directory | Domain | Local Guidelines |
|-----------|--------|------------------|
| [`acm/`](acm/CLAUDE.md) | Advanced Cluster Management — governance policies, placements, Gatekeeper constraints | ACM policy CRD conventions, inform vs enforce rules |
| [`acs/`](acs/CLAUDE.md) | Advanced Cluster Security — StackRox policies, vulnerability management, compliance | roxctl usage, security policy severity levels |
| [`logging/`](logging/CLAUDE.md) | Cluster Logging — LokiStack, Vector, ClusterLogging operator | Log collection pipelines, storage configuration |
| [`pipelines/`](pipelines/CLAUDE.md) | Tekton Pipelines — CI/CD tasks, pipelines, triggers | Task/Pipeline conventions, parameter standards |

## Conventions

### YAML Style

- Use `---` document separators between resources in the same file.
- Always include `apiVersion`, `kind`, `metadata.name`, and `metadata.namespace` where applicable.
- Sort resource fields in standard Kubernetes order: apiVersion, kind, metadata, spec, status.

### Naming

- File names: lowercase kebab-case describing the primary resource (e.g., `policy-gatekeeper-setup.yaml`).
- Resource names: lowercase kebab-case, prefixed with the domain when helpful (e.g., `policy-gk-constraints`).
- Namespaces: use the operator-standard namespace for each tool (e.g., `open-cluster-management-global-set`, `stackrox`, `openshift-logging`).

### Validation

- All manifests must pass `oc apply --dry-run=client` before being committed.
- Use the `/oc-validate` skill for batch validation of any modified YAML files.
