# Pipelines — Tekton CI/CD

Guidelines for authoring and reviewing Tekton Pipeline resources in this directory.

## API Group

All pipeline resources use `tekton.dev/v1` (v1beta1 is deprecated — do not use it for new resources).

## Resource Types

| Kind | Purpose |
|------|---------|
| `Task` | Single unit of work with steps |
| `Pipeline` | Ordered composition of Tasks |
| `PipelineRun` | Execution instance of a Pipeline |
| `TaskRun` | Execution instance of a Task |
| `Trigger` / `EventListener` | Webhook-driven pipeline execution |

## Parameter Conventions

- Every `Task` and `Pipeline` parameter MUST have a `default` value, unless the parameter is truly required at runtime (e.g., git revision, image name).
- Use descriptive parameter names in kebab-case (e.g., `git-url`, `image-registry`).
- Document each parameter with a `description` field.

```yaml
params:
  - name: git-url
    type: string
    description: "Git repository URL to clone"
  - name: image-tag
    type: string
    description: "Tag for the built container image"
    default: "latest"
```

## Workspaces and Secrets

- Use `workspace` bindings to pass data between tasks — never use `emptyDir` for shared state that must persist across tasks.
- Secrets (registry credentials, git tokens, API keys) must be mounted via `Secret` workspace bindings or `SecretKeyRef` environment variables — never hardcoded in step scripts.
- Name workspaces by their purpose: `source`, `dockerconfig`, `kubeconfig`, `maven-settings`.

```yaml
workspaces:
  - name: source
    description: "Workspace containing the cloned source code"
  - name: dockerconfig
    description: "Docker config.json for registry authentication"
    optional: true
```

## Task Step Rules

- Use minimal base images for steps (prefer `registry.redhat.io/ubi9/ubi-minimal`).
- Each step should have explicit `resources` requests.
- Set `securityContext.runAsNonRoot: true` on steps unless root is strictly required.
- Never use `script` blocks that pull binaries from the internet at runtime — bake tools into the step image.

## Pipeline Structure

- Tasks within a pipeline should declare explicit `runAfter` dependencies — do not rely on definition order.
- Use `finally` tasks for cleanup (e.g., notification, artifact upload) regardless of pipeline success/failure.
- Set `timeouts` on PipelineRuns to prevent runaway executions.

## Naming

- Tasks: `<verb>-<noun>` (e.g., `build-image`, `deploy-manifests`, `scan-vulnerabilities`).
- Pipelines: `<project>-<workflow>` (e.g., `app-build-deploy`, `infra-validate`).
- Files: one resource per file, named to match the resource (e.g., `build-image.yaml`).
