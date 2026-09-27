---
name: pipelines-tekton-dryrun
description: Lint Tekton Tasks and Pipelines for missing parameter defaults and hardcoded credentials
auto-trigger:
  - on-file-change: "pipelines/**/*.yaml"
  - on-file-change: "pipelines/**/*.yml"
allowed-tools:
  - Bash
  - Read
---

# pipelines-tekton-dryrun

Automatically lints Tekton resources when files in the `pipelines/` directory are modified.

## Checks Performed

### 1. API Version Check
- Verify resources use `tekton.dev/v1` (flag any using deprecated `tekton.dev/v1beta1`).

### 2. Parameter Defaults
For each `Task` and `Pipeline`:

- List all declared `params`.
- Flag any parameter without a `default` value that is not clearly a required runtime input.
- Verify every parameter has a `description` field.

### 3. Hardcoded Credential Detection
Scan all step `script` blocks and `env` values for:

- Inline passwords or tokens (patterns: `password=`, `token=`, `apikey=`, `secret=`).
- Hardcoded registry credentials.
- Base64-encoded secret-like strings.

Flag any match as a security violation.

### 4. Workspace Binding Audit
- Verify secrets are passed via `workspace` bindings or `SecretKeyRef`, not inline.
- Flag any `emptyDir` workspace used for cross-task data sharing (should use a PVC-backed workspace).

### 5. Step Security
For each step in a Task:

- Flag missing `securityContext.runAsNonRoot`.
- Flag steps that download binaries at runtime (`curl`, `wget` in script blocks pulling executables).
- Verify base images use a pinned tag, not `latest`.

### 6. Pipeline Structure
For each Pipeline:

- Verify tasks declare explicit `runAfter` for ordering (not relying on definition order).
- Check for a `finally` block for cleanup tasks.

### 7. Dry-Run Validation
```bash
oc apply --dry-run=client -f <file> 2>&1
```

## Output

Report each check with PASS/FAIL/WARN status per file, followed by a summary table.
