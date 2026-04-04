# Helm Templates — Developer Guide

This document is for contributors to `helm-templates` itself: adding resources, writing tests, understanding how it wraps `helm-core`.

---

## What this chart is

`helm-templates` is a thin platform chart. It owns:

- A structured, schema-validated `values.yaml` API
- Template files that check feature flags and delegate rendering to `helm-core` helpers
- A `values.schema.json` that enforces the API contract

It does not contain rendering logic. All Kubernetes resource construction lives in `helm-core`.

---

## Directory structure

```
helm-templates/
├── Chart.yaml
├── values.yaml             # defaults for optional fields
├── values.schema.json      # strict API schema (additionalProperties: false)
├── Taskfile.yml            # developer commands
├── templates/
│   ├── configmap.yaml
│   ├── deployment.yaml
│   ├── daemonset.yaml
│   ├── statefulset.yaml
│   ├── service.yaml
│   ├── cronjob.yaml
│   ├── hpa.yaml
│   ├── pdb.yaml
│   ├── pvc.yaml
│   └── serviceaccount.yaml
└── unit-tests/
    ├── configmap_test.yaml
    ├── deployment_test.yaml
    ├── daemonset_test.yaml
    ├── statefulset_test.yaml
    ├── service_test.yaml
    ├── service_connectivity_test.yaml
    ├── cronjob_test.yaml
    ├── hpa_test.yaml
    ├── pdb_test.yaml
    ├── pod_features_test.yaml
    ├── pvc_test.yaml
    ├── labels_test.yaml
    ├── serviceaccount_test.yaml
    └── strategy_test.yaml
```

---

## Template pattern

Every template file follows the same structure:

```yaml
{{- if <enabled condition> }}
{{- $config := include "core.general.config" . | fromYaml }}
{{- $context := merge (dict "$" $) $config }}
kind: <ResourceKind>
...render using $context helpers...
{{- end }}
```

The enabled condition varies by resource type:

| Template | Condition |
|----------|-----------|
| `deployment.yaml` | `if eq (.Values.workload).type "Deployment"` |
| `daemonset.yaml` | `if eq (.Values.workload).type "DaemonSet"` |
| `statefulset.yaml` | `if eq (.Values.workload).type "StatefulSet"` |
| `configmap.yaml` | `if .Values.configMap.enabled` |
| `service.yaml` | `if .Values.service.enabled` |
| `cronjob.yaml` | `if .Values.cronJob.enabled` |
| `hpa.yaml` | `if and .Values.hpa.enabled (ne (.Values.workload).type "DaemonSet") (not $config.activeRegion)` |
| `pdb.yaml` | `if (.Values.pdb).enabled` |
| `pvc.yaml` | iterates `$context.persistentVolumeClaims` |
| `serviceaccount.yaml` | `if (.Values.serviceAccount).create` |

Note `cronJob` (camelCase) in both the values key and `annotations.cronJob`.

---

## Running tests locally

Requires the `helm-unittest` plugin and [Task](https://taskfile.dev/).

```bash
# Package helm-core and wire it into helm-templates/charts/ (run once, or after helm-core changes)
task setup

# Run all unit tests
task test
```

`task setup` cleans any existing `charts/`, lock files, and `.tgz` packages, then re-links helm-core from the local repo. `task test` runs `helm unittest . -f 'unit-tests/*_test.yaml'`.

To run a single test file:

```bash
helm unittest . -f 'unit-tests/deployment_test.yaml'
```

---

## Adding a new resource

1. Add a new template file in `templates/` following the pattern above.
2. Add the enabling condition and values key.
3. Add the corresponding schema entry in `values.schema.json`:
   - Add the top-level key with its type and required fields.
   - Add any conditional `if/then` validation (e.g., required sub-fields when `enabled: true`).
   - Keep `additionalProperties: false` at every object level.
4. Add defaults to `values.yaml` if the resource has opt-out defaults (like `pdb.enabled: false`).
5. Write unit tests in `unit-tests/<resource>_test.yaml`.
6. Run `task test`.

---

## Schema evolution

`values.schema.json` enforces `additionalProperties: false` at every object level. This means:

- Adding a new top-level key requires a schema entry or Helm will reject it.
- Renaming a key is a breaking change — bump the chart version.
- New optional keys are backwards compatible as long as existing required fields don't change.

The schema uses `allOf` constraints for cross-field validation:

- `replicas` is required unless `workload.type` is `DaemonSet`
- `statefulSet` block is required when `workload.type` is `StatefulSet`
- `hpa.enabled` must not be true when `workload.type` is `DaemonSet`

When adding conditional validation, use `if/then` inside `allOf` to keep conditions co-located with their context.

---

## Defaults in values.yaml

`values.yaml` only contains defaults for fields that are:
- Optional but have a sensible non-empty default (e.g. `revisionHistoryLimit: 1`)
- Disabled by default but whose schema requires an `enabled` key to be present (e.g. `pdb.enabled: false`, `hpa.enabled: false`)
- Complex objects with defaults consumers should see and override (e.g. `topologySpreadConstraints`)

Do not add defaults for required fields — those belong in the user's values file.

---

## Publishing

The chart is published as an OCI artifact via the GitHub Actions workflow at `.github/workflows/publish.yml`. Publishing is triggered on version bumps to `Chart.yaml`. Update both `helm-core` and `helm-templates` versions together when `helm-core` changes, since `helm-templates` vendors a specific version of `helm-core`.
