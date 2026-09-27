# ACS — Advanced Cluster Security (StackRox)

Guidelines for authoring and reviewing ACS security policy resources in this directory.

## API Group

Security policies use `platform.stackrox.io/v1alpha1`.

## roxctl Usage

- Use `roxctl deployment check` to validate a deployment manifest against ACS policies locally before applying to a cluster.
- Use `roxctl image scan` for vulnerability scanning of container images.
- Use `roxctl image check` to evaluate an image against build-time policies.
- Always set `ROX_CENTRAL_ADDRESS` and `ROX_API_TOKEN` as environment variables — never inline them in commands or manifests.

```bash
# Validate a deployment manifest against ACS policies
roxctl deployment check --file <deployment.yaml>

# Scan an image for vulnerabilities
roxctl image scan --image <registry/image:tag>
```

## Security Policy Structure

Each `SecurityPolicy` must include:

| Field | Purpose |
|-------|---------|
| `policyName` | Human-readable display name |
| `description` | Clear explanation of what the policy blocks and why |
| `severity` | One of: `LOW_SEVERITY`, `MEDIUM_SEVERITY`, `HIGH_SEVERITY`, `CRITICAL_SEVERITY` |
| `categories` | Classification (e.g., `Vulnerability Management`, `DevOps Best Practices`) |
| `lifecycleStages` | When enforcement applies: `BUILD`, `DEPLOY`, `RUNTIME` |
| `enforcementActions` | What happens on violation (see below) |

## Enforcement Actions

| Action | Stage | Effect |
|--------|-------|--------|
| `FAIL_BUILD_ENFORCEMENT` | BUILD | Fails CI pipeline builds |
| `SCALE_TO_ZERO_ENFORCEMENT` | DEPLOY | Scales violating deployments to zero replicas |
| `KILL_POD_ENFORCEMENT` | RUNTIME | Terminates pods violating runtime policies |
| `UNSET_ENFORCEMENT` | Any | Inform-only, no active enforcement |

- New policies should start with `UNSET_ENFORCEMENT` for validation, then escalate.
- Combine `FAIL_BUILD_ENFORCEMENT` + `SCALE_TO_ZERO_ENFORCEMENT` for defense-in-depth on vulnerability policies.

## Central and Sensor Configuration

- Central runs in the `stackrox` namespace — never deploy security policies to other namespaces.
- Sensor is deployed per secured cluster via `SecuredCluster` CR.
- Helm annotations (e.g., `meta.helm.sh/release-name`) must be preserved on resources managed by the StackRox Helm chart.

## CVSS Thresholds

Standard severity thresholds for vulnerability policies:

| Threshold | Classification | Recommended Action |
|-----------|---------------|-------------------|
| >= 9.0 | Critical | Block build + scale to zero |
| >= 7.0 | High | Block build + scale to zero |
| >= 4.0 | Medium | Warn or inform |
| < 4.0 | Low | Inform only |
