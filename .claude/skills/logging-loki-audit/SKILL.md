---
name: logging-loki-audit
description: Validate LokiStack storage configurations and ClusterLogging schema correctness
auto-trigger:
  - on-file-change: "logging/**/*.yaml"
  - on-file-change: "logging/**/*.yml"
allowed-tools:
  - Bash
  - Read
---

# logging-loki-audit

Automatically validates logging configurations when files in the `logging/` directory are modified.

## Checks Performed

### 1. ClusterLogging Validation
For any `ClusterLogging` resource:

- Verify `spec.collection.type` is `vector` (not `fluentd`).
- Verify `spec.logStore.type` is `lokistack` (not `elasticsearch`).
- Verify `spec.logStore.lokistack.name` references a LokiStack by name.
- Verify `metadata.namespace` is `openshift-logging`.

### 2. LokiStack Storage Validation
For any `LokiStack` resource:

- Verify `spec.size` is explicitly set (one of: `1x.extra-small`, `1x.small`, `1x.medium`, `1x.large`).
- Verify `spec.storage.secret.name` is set and does not contain inline credentials.
- Verify `spec.storage.schemas` has at least one schema entry with a `version` and `effectiveDate`.
- Verify `spec.storageClassName` is set.
- Flag any hardcoded S3/GCS bucket credentials in the manifest.

### 3. ClusterLogForwarder Validation
For any `ClusterLogForwarder` resource:

- Verify each pipeline has explicit `inputRefs` (one or more of: `application`, `infrastructure`, `audit`).
- Verify each output has a `type` and `url` (where applicable).
- Verify TLS configuration is present for external outputs.

### 4. Dry-Run Validation
```bash
oc apply --dry-run=client -f <file> 2>&1
```

## Output

Report each check with PASS/FAIL status. For failures, include the specific field and recommended correction.
