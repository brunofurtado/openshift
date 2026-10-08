# ACS — Advanced Cluster Security (StackRox)

Declarative vulnerability management policies for Red Hat Advanced Cluster Security,
blocking high-risk container images at build time and deploy time.

## Table of Contents

- [Architecture](#architecture)
  - [Component Distribution](#component-distribution)
  - [Central Cluster](#central-cluster)
  - [Secured Clusters](#secured-clusters)
  - [Enforcement Flow](#enforcement-flow)
- [Prerequisites](#prerequisites)
- [Files](#files)
- [Apply](#apply)
- [Verify](#verify)
- [What Gets Blocked](#what-gets-blocked)
  - [Why Two Overlapping Policies](#why-two-overlapping-policies)
  - [Enforcement Scope and Limits](#enforcement-scope-and-limits)
- [Enforcement Actions](#enforcement-actions)
- [Severity Thresholds](#severity-thresholds)
- [roxctl Usage](#roxctl-usage)
  - [Authentication](#authentication)
  - [Local Validation](#local-validation)
  - [CI Integration](#ci-integration)
- [Rollout Procedure](#rollout-procedure)
- [Policy Authoring Reference](#policy-authoring-reference)

---

## Architecture

ACS separates a central control plane from per-cluster agents. Policies are defined
once in Central and evaluated by agents running on every secured cluster.

### Component Distribution

| Component | Central Cluster | Secured Cluster | Deployed By |
|-----------|:---------------:|:---------------:|-------------|
| Central (API, UI, policy engine) | Yes | No | `Central` CR via ACS operator |
| Central DB (PostgreSQL) | Yes | No | `Central` CR |
| Scanner (image vulnerability analysis) | Yes | Optional | `Central` CR |
| Sensor | No | Yes | `SecuredCluster` CR |
| Collector (DaemonSet) | No | Yes | `SecuredCluster` CR |
| Admission Controller (webhook) | No | Yes | `SecuredCluster` CR |

A cluster can be both — the cluster running Central is usually also secured by its
own `SecuredCluster` CR.

### Central Cluster

Runs in the `stackrox` namespace and is responsible for:

- Storing policy definitions and evaluating image scan results.
- Aggregating violations and risk scores across all secured clusters.
- Serving the ACS console and the API that `roxctl` talks to.

### Secured Clusters

- **Sensor** connects outbound to Central, receives policy definitions, and
  applies deploy-time enforcement such as scaling a violating workload to zero.
- **Collector** runs on every node and feeds runtime process and network data.
- **Admission Controller** is a validating webhook that can reject a workload
  before it is admitted, rather than remediating it after the fact.

### Enforcement Flow

```
┌─────────────────────────────────────────────────────────┐
│                    Central Cluster                       │
│                                                          │
│   SecurityPolicy CR ──► Central ──► Policy Engine        │
│                            │              │               │
│                            │              ▼               │
│                            │          Scanner             │
│                            │      (CVE + severity)        │
│                            ▼                              │
│                      Violations DB                        │
│                            ▲                              │
└────────────────────────────┼──────────────────────────────┘
                             │
                    policy sync / violation reports
                             │
┌────────────────────────────┼──────────────────────────────┐
│                            ▼        Secured Cluster        │
│                        Sensor                              │
│                       │      │                             │
│          ┌────────────┘      └────────────┐                │
│          ▼                                ▼                │
│  Admission Controller              Collector               │
│  (deny on admission)               (runtime signals)       │
│          │                                                 │
│          ▼                                                 │
│  Deployment admitted / denied / scaled to zero             │
└────────────────────────────────────────────────────────────┘
```

Build-time enforcement sits outside this loop entirely — it happens in CI, when
`roxctl` asks Central to evaluate an image. See [CI Integration](#ci-integration).

---

## Prerequisites

- Red Hat Advanced Cluster Security operator installed
- A `Central` instance running in the `stackrox` namespace
- At least one `SecuredCluster` registered with Central
- Scanner enabled — both policies depend on image scan results
- `roxctl` on the workstation for local validation

---

## Files

### 1. `policy-cvss-score-block.yaml`

Blocks images containing any individual CVE scored CVSS 7.0 or higher.

| Field | Value |
|-------|-------|
| **Resource** | `SecurityPolicy` |
| `metadata.name` | `block-cvss-7-or-higher` |
| `spec.policyName` | Block Images with CVSS >= 7 |
| `severity` | `HIGH_SEVERITY` |
| `categories` | Vulnerability Management |
| `lifecycleStages` | `BUILD`, `DEPLOY` |
| `enforcementActions` | `FAIL_BUILD_ENFORCEMENT`, `SCALE_TO_ZERO_ENFORCEMENT` |
| Criterion | `CVSS >= 7.0` |

### 2. `policy-image-severity-block.yaml`

Blocks images whose aggregated severity rating is Important or Critical.

| Field | Value |
|-------|-------|
| **Resource** | `SecurityPolicy` |
| `metadata.name` | `block-high-severity-images` |
| `spec.policyName` | Block Deployment of High/Critical Severity Images |
| `severity` | `HIGH_SEVERITY` |
| `categories` | Vulnerability Management |
| `lifecycleStages` | `DEPLOY` |
| `enforcementActions` | `SCALE_TO_ZERO_ENFORCEMENT` |
| Criterion | `Severity >= IMPORTANT` |

Both carry the annotation `meta.helm.sh/release-name: stackrox-secured-cluster-services`.
Preserve it — removing it breaks ownership tracking for resources managed by the
StackRox Helm chart.

---

## Apply

```bash
# Validate first (repo policy).
oc apply --dry-run=client -f policy-cvss-score-block.yaml
oc apply --dry-run=client -f policy-image-severity-block.yaml

# Apply.
oc apply -f policy-cvss-score-block.yaml
oc apply -f policy-image-severity-block.yaml
```

Order does not matter — the two policies are independent and have no dependency
on one another.

## Verify

```bash
# Confirm the custom resources exist
oc get securitypolicy -n stackrox

# Inspect reconciliation status for one policy
oc describe securitypolicy block-cvss-7-or-higher -n stackrox

# Check the operator reconciled them without error
oc logs -n rhacs-operator deployment/rhacs-operator-controller-manager --tail=50
```

Then confirm in the ACS console under **Platform Configuration → Policy Management**
that both policies appear and show the expected enforcement state.

---

## What Gets Blocked

| Policy | Checks | Build | Deploy |
|--------|--------|:-----:|:------:|
| `block-cvss-7-or-higher` | Any single CVE with CVSS >= 7.0 | Fail build | Scale to zero |
| `block-high-severity-images` | Aggregated image severity >= Important | — | Scale to zero |

### Why Two Overlapping Policies

They measure different things and catch different images:

- **CVSS** is a per-CVE numeric score. A single High-scoring CVE trips this policy
  even if the image is otherwise clean.
- **Severity** is the vendor-assigned rating ACS aggregates for the image as a whole.
  Red Hat's rating for a CVE can differ from its upstream CVSS score — a CVE scored
  7.5 upstream may be rated Moderate by Red Hat when the vulnerable code path is not
  reachable in their build.

Running both means neither a high raw score nor a high vendor rating slips through.
The cost is duplicate violations for images that trip both.

### Enforcement Scope and Limits

Three things worth knowing before you rely on these policies:

1. **Enforcement applies to new and updated workloads, not existing ones.**
   Applying a policy does not retroactively scale down already-running deployments.
   It flags them as violations; enforcement triggers on the next create or update.

2. **`SCALE_TO_ZERO_ENFORCEMENT` is remediation, not prevention.** The workload is
   admitted, then Sensor scales it to zero replicas. There is a brief window where
   the pod exists. To reject at admission instead, enable the admission controller
   on the `SecuredCluster` CR.

3. **`FAIL_BUILD_ENFORCEMENT` does nothing on its own.** ACS cannot fail a build it
   is not part of. The flag only takes effect when a CI step calls
   `roxctl image check`, which returns a non-zero exit code on violation.

---

## Enforcement Actions

| Action | Stage | Effect |
|--------|-------|--------|
| `FAIL_BUILD_ENFORCEMENT` | BUILD | Non-zero exit from `roxctl image check`, failing the CI step |
| `SCALE_TO_ZERO_ENFORCEMENT` | DEPLOY | Sensor scales the violating deployment to zero replicas |
| `KILL_POD_ENFORCEMENT` | RUNTIME | Terminates pods violating runtime policies |
| `UNSET_ENFORCEMENT` | Any | Inform only — violations are recorded, nothing is blocked |

An enforcement action only works if the matching stage is listed in
`lifecycleStages`. `FAIL_BUILD_ENFORCEMENT` on a policy without `BUILD` has no effect.

---

## Severity Thresholds

Standard thresholds used when authoring new vulnerability policies in this repo:

| Threshold | Classification | Recommended Action |
|-----------|----------------|--------------------|
| >= 9.0 | Critical | Block build + scale to zero |
| >= 7.0 | High | Block build + scale to zero |
| >= 4.0 | Medium | Warn or inform |
| < 4.0 | Low | Inform only |

---

## roxctl Usage

### Authentication

Never inline the endpoint or token in a command or manifest. Export them:

```bash
export ROX_CENTRAL_ADDRESS="central-stackrox.apps.<cluster-domain>:443"
export ROX_API_TOKEN="$(oc get secret <token-secret> -n stackrox -o jsonpath='{.data.token}' | base64 -d)"
```

In CI, supply `ROX_API_TOKEN` from a `Secret` via `secretKeyRef` — never as a
literal in a pipeline definition.

### Local Validation

```bash
# Evaluate an image against build-time policies
roxctl image check --image <registry/image:tag>

# Full vulnerability report for an image
roxctl image scan --image <registry/image:tag>

# Evaluate a deployment manifest against deploy-time policies
roxctl deployment check --file <deployment.yaml>
```

### CI Integration

`roxctl image check` exits non-zero when an image trips a policy carrying
`FAIL_BUILD_ENFORCEMENT`, which is what actually fails the pipeline:

```bash
roxctl image check --image "${IMAGE}" || exit 1
```

Useful flags:

| Flag | Purpose |
|------|---------|
| `--output json` | Machine-readable result for pipeline parsing |
| `--severity` | Restrict output to a minimum severity |
| `--retries` | Retry on transient Central connectivity failures |

---

## Rollout Procedure

Both policies in this directory are already at full enforcement. When adding a
**new** policy, escalate rather than starting at enforce:

```
1. Create with enforcementActions: [UNSET_ENFORCEMENT]
   └── Policy evaluates and records violations, blocks nothing

2. Review violations in the ACS console for a representative period
   └── Confirm the match set is what you intended, not a wall of false positives

3. Add FAIL_BUILD_ENFORCEMENT
   └── Catches new images in CI before they ever reach a cluster

4. Add SCALE_TO_ZERO_ENFORCEMENT
   └── Closes the gap for images deployed outside the pipeline
```

Jumping straight to step 4 on a cluster with existing workloads tends to scale down
something unexpected the first time anyone redeploys it.

---

## Policy Authoring Reference

Every `SecurityPolicy` in this directory must set:

| Field | Purpose |
|-------|---------|
| `policyName` | Human-readable display name shown in the ACS console |
| `description` | What the policy blocks and why |
| `severity` | `LOW_SEVERITY`, `MEDIUM_SEVERITY`, `HIGH_SEVERITY`, `CRITICAL_SEVERITY` |
| `categories` | Classification, e.g. Vulnerability Management, DevOps Best Practices |
| `lifecycleStages` | `BUILD`, `DEPLOY`, `RUNTIME` |
| `enforcementActions` | What happens on violation |
| `policySections` | The actual match criteria |

Conventions:

- All resources use `platform.stackrox.io/v1alpha1` and live in the `stackrox`
  namespace — never deploy security policies elsewhere.
- File names are kebab-case and describe the check: `policy-<what-it-blocks>.yaml`.
- `metadata.name` is kebab-case; `spec.policyName` is the readable title.
- Preserve Helm annotations on resources managed by the StackRox chart.

See [`CLAUDE.md`](CLAUDE.md) in this directory for the full authoring guidelines.
