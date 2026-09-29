# OpenShift Security and Governance Lab

A collection of declarative policies and configurations for securing and governing Red Hat OpenShift clusters. This repository provides ready-to-apply manifests covering admission control, vulnerability management, centralized logging, and CI/CD pipelines.

## Purpose

This repository serves as a hands-on lab for implementing security and operational best practices across an OpenShift platform using:

- **Advanced Cluster Management (ACM)** — Governance policies distributed across managed clusters via the hub-and-spoke model.
- **Advanced Cluster Security (ACS / StackRox)** — Vulnerability scanning and deployment-time enforcement policies.
- **Cluster Logging** — Centralized log collection with LokiStack and Vector.
- **Tekton Pipelines** — CI/CD task and pipeline definitions with built-in security guardrails.

## Repository Structure

```
openshift/
├── acm/                        # ACM governance policies
│   ├── policy-gatekeeper-setup.yaml    # Gatekeeper ConstraintTemplate (Rego logic)
│   └── policy-gk-constraint.yaml       # Gatekeeper Constraint (admission enforcement)
├── acs/                        # ACS / StackRox security policies
│   ├── policy-cvss-score-block.yaml        # Block images with CVSS >= 7.0
│   └── policy-image-severity-block.yaml    # Block images with severity >= Important
├── logging/                    # Cluster logging configurations
│   └── (configurations coming soon)
├── pipelines/                  # Tekton CI/CD pipelines
│   └── (pipelines coming soon)
└── CLAUDE.md                   # AI assistant context and safety guardrails
```

## Prerequisites

- An OpenShift 4.x cluster (or set of managed clusters)
- The following operators installed where applicable:
  - Red Hat Advanced Cluster Management for Kubernetes
  - Red Hat Advanced Cluster Security for Kubernetes
  - Red Hat OpenShift Logging
  - Red Hat OpenShift Pipelines
- CLI tools: `oc`, `kustomize`, `helm`, `tkn`, `roxctl`

## Getting Started

1. Clone this repository:
   ```bash
   git clone <repository-url>
   cd openshift
   ```

2. Validate manifests before applying:
   ```bash
   oc apply --dry-run=client -f <manifest.yaml>
   ```

3. Apply configurations to your cluster:
   ```bash
   # Example: deploy ACM Gatekeeper policies
   oc apply -f acm/policy-gatekeeper-setup.yaml
   oc apply -f acm/policy-gk-constraint.yaml
   ```

## What Gets Enforced

### ACM — Disallow Latest Image Tag
Blocks any Deployment or Pod using the `:latest` image tag (or no tag at all) across managed clusters via OPA Gatekeeper admission webhooks. System namespaces (`kube-*`, `openshift-*`, `stackrox`, `open-cluster-management*`) are excluded.

### ACS — Vulnerability Policies
Two complementary policies prevent deploying vulnerable images:
| Policy | Check | Action |
|--------|-------|--------|
| `policy-cvss-score-block` | Individual CVEs with CVSS >= 7.0 | Fail build + scale to zero |
| `policy-image-severity-block` | Overall image severity >= Important | Scale to zero |

## Contributing

1. Always validate manifests with `oc apply --dry-run=client` before committing.
2. Follow the naming conventions documented in each subproject's `CLAUDE.md`.
3. New policies should start in `inform` / `UNSET_ENFORCEMENT` mode before escalating to active enforcement.

## License

This project is licensed under the Apache License 2.0 — see the [LICENSE](LICENSE) file for details.
