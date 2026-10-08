# Namespace Cleanup — must-gather and debug orphans

A CronJob that deletes leftover `openshift-must-gather-*` and `openshift-debug-*`
namespaces once they are older than a configurable age.

## Why this is needed

`oc adm must-gather` and `oc debug` each create a temporary namespace and remove it
when they finish. Interrupted runs — a cancelled command, a dropped connection, a
collection that times out — leave the namespace behind. Over time these accumulate
and clutter the cluster.

## What gets deleted

A namespace is deleted only when **both** conditions hold:

1. Its name matches `^openshift-(must-gather|debug|debug-node)-[a-z0-9]{5}$`
2. Its `creationTimestamp` is older than `MAX_AGE_DAYS` (default: **7**)

Both tools create their namespace with a `generateName` prefix, so Kubernetes
appends exactly five random characters. Anchoring the pattern on that suffix is
what keeps permanent namespaces out of scope:

| Namespace | Deleted? | Reason |
|-----------|----------|--------|
| `openshift-must-gather-x7k2p` | yes (if aged) | generated suffix |
| `openshift-debug-ab12c` | yes (if aged) | generated suffix |
| `openshift-debug-node-9zqrt` | yes (if aged) | generated suffix, pre-4.12 form |
| `openshift-must-gather-operator` | **no** | not a 5-character suffix |
| `openshift-debug` | **no** | no suffix |
| `openshift-monitoring` | **no** | prefix does not match |

Namespaces already in `Terminating` phase are skipped.

## Resources

| File | Contents |
|------|----------|
| `rbac.yaml` | `Namespace`, `ServiceAccount`, `ClusterRole`, `ClusterRoleBinding` |
| `cleanup-cronjob.yaml` | `ConfigMap` (the cleanup script) and the `CronJob` |

The ClusterRole grants only `get`, `list`, and `delete` on `namespaces` — no
create, no patch, no other resource types.

## Deploy

```bash
# Validate first (repo policy).
oc apply --dry-run=client -f rbac.yaml
oc apply --dry-run=client -f cleanup-cronjob.yaml

# Apply.
oc apply -f rbac.yaml
oc apply -f cleanup-cronjob.yaml
```

### Pin the image to your cluster version

`cleanup-cronjob.yaml` references `registry.redhat.io/openshift4/ose-cli:v4.16`.
Change the tag to match your cluster's minor version:

```bash
oc version -o json | jq -r '.openshiftVersion'
```

An explicit tag is required — the ACM Gatekeeper constraint and the ACS severity
policies in this repo both reject `:latest`.

## Verify before trusting it

The CronJob deletes from its first scheduled run. To see what it would select
without waiting for the schedule, run the selection half by hand:

```bash
oc get ns -o go-template='{{range .items}}{{.metadata.name}} {{.metadata.creationTimestamp}}{{"\n"}}{{end}}' \
  | grep -E '^openshift-(must-gather|debug|debug-node)-[a-z0-9]{5} '
```

To trigger an immediate run:

```bash
oc create job --from=cronjob/ns-cleanup ns-cleanup-manual -n ns-cleanup
oc logs -n ns-cleanup job/ns-cleanup-manual -f
```

## Tuning

| Setting | Where | Default |
|---------|-------|---------|
| Age threshold | `MAX_AGE_DAYS` env var on the container | `7` |
| Schedule | `spec.schedule` | `0 3 * * *` (daily, 03:00 UTC) |
| Run timeout | `spec.jobTemplate.spec.activeDeadlineSeconds` | `900` |

```bash
# Change the age threshold without re-applying the whole manifest:
oc set env cronjob/ns-cleanup -n ns-cleanup MAX_AGE_DAYS=14
```

## Notes

- `oc delete namespace` uses `--wait=false` so a namespace stuck in finalization
  does not block the rest of the sweep.
- `concurrencyPolicy: Forbid` prevents overlapping runs.
- The pod runs under `restricted-v2`: non-root, read-only root filesystem, all
  capabilities dropped. `HOME` is set to a writable `emptyDir` because `oc`
  writes its discovery cache there.
