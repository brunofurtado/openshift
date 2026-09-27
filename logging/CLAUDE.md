# Logging — Cluster Logging and LokiStack

Guidelines for authoring and reviewing OpenShift cluster logging resources in this directory.

## API Groups

- ClusterLogging: `logging.openshift.io/v1`
- ClusterLogForwarder: `logging.openshift.io/v1`
- LokiStack: `loki.grafana.openshift.io/v1`

## Namespace

All logging resources are deployed to `openshift-logging`.

## ClusterLogging

- Use the `vector` collector implementation (the `fluentd` collector is deprecated).
- Always specify resource requests and limits for the collector.
- Define log retention policies per log type (`application`, `infrastructure`, `audit`).

```yaml
apiVersion: logging.openshift.io/v1
kind: ClusterLogging
metadata:
  name: instance
  namespace: openshift-logging
spec:
  collection:
    type: vector
  logStore:
    type: lokistack
    lokistack:
      name: logging-loki
```

## LokiStack

- Use `LokiStack` as the log store backend (ElasticSearch is deprecated).
- Storage must reference an existing `ObjectBucketClaim` or S3-compatible secret — never hardcode storage credentials.
- Size the stack appropriately: `1x.extra-small` for dev, `1x.small` for staging, `1x.medium`+ for production.
- Always configure `storageClassName` to a valid StorageClass on the target cluster.

| Field | Requirement |
|-------|-------------|
| `spec.size` | Must be set explicitly |
| `spec.storage.schemas` | At least one schema version defined |
| `spec.storageClassName` | Must reference an existing StorageClass |
| `spec.storage.secret.name` | Must reference an existing Secret |

## ClusterLogForwarder

- Use `ClusterLogForwarder` to route logs to external systems.
- Define explicit `inputRefs` — never rely on implicit log forwarding.
- Supported output types: `loki`, `kafka`, `syslog`, `cloudwatch`, `elasticsearch`.
- Always set `tls` configuration when forwarding to external endpoints.

## Vector Collection Pipelines

- Vector replaces Fluentd as the default collector — never create new Fluentd configurations.
- Vector pipeline filters should be defined via `ClusterLogForwarder` filters, not raw Vector config.
- Monitor collector pod health via `oc get pods -n openshift-logging -l component=collector`.
