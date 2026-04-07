# Helm Templates — User Guide

A ready-to-use application Helm chart that generates Kubernetes resources from a structured `values.yaml` — no template writing required.

---

## Quick start

```bash
helm dependency update
helm template my-service . -f values.yaml
helm install my-service . -f values.yaml
```

For rendering without a cluster (CI, local), add `global.ignoreLookup: "true"` to your values.

---

## Workload types

Select the workload kind with `workload.type`. Exactly one workload resource is rendered per release.

| Type | Notes |
|------|-------|
| `Deployment` | Stateless services (default) |
| `StatefulSet` | Requires `statefulSet.headlessService` |
| `DaemonSet` | `replicas` not required |

---

## Resources

All resources except the workload are opt-in and produce no output when disabled.

| Resource | Enabled by |
|----------|------------|
| Deployment / StatefulSet / DaemonSet | `workload.type` |
| ConfigMap | `configMap.enabled: true` |
| Service | `service.enabled: true` |
| CronJob | `cronJob.enabled: true` |
| HPA | `hpa.enabled: true` |
| PDB | `pdb.enabled: true` |
| PersistentVolumeClaims | `persistentVolumeClaims.*` (one PVC per entry) |
| ServiceAccount | `serviceAccount.create: true` |

---

## Required values

```yaml
global:
  region: us-east-1

port: 8080
replicas: 2           # not required for DaemonSet

resources:
  cpu: 250m
  memory: 512Mi

workload:
  type: Deployment
```

---

## Values reference

### Global scope

Shared across all sub-charts in an umbrella. Local values take priority.

```yaml
global:
  region: us-east-1            # required
  ignoreLookup: "true"         # bypass cluster lookups (CI / local rendering)
  commit: "abc123"             # optional — adds a commit label to workloads
  image:
    url: myrepo/base
    tag: latest
    pullPolicy: IfNotPresent   # Always | IfNotPresent | Never
    pullSecrets:
      - registry-credentials
  envFrom:
    secrets:
      platform-secret: {}
    configMaps:
      platform-config: {}
  volumes:
    secrets:
      platform-secret:
        mountPath: /run/secrets
    empty:
      tmp-scratch: {}
  env:
    secrets:
      platform-secret:
        - key: API_KEY
          variable: PLATFORM_API_KEY
```

### Image

```yaml
image:
  url: myrepo/myapp         # required if global.image.url not set
  tag: "1.2.3"              # default: "latest"
  pullPolicy: IfNotPresent
  pullSecrets:
    - my-registry-secret
```

### Regions

Override any value for a specific region. Active region is determined by `global.region`.

```yaml
regions:
  us-east-1:
    configMap:
      data:
        DB_HOST: "db.us-east-1.example.com"
  us-west-1:
    configMap:
      data:
        DB_HOST: "db.us-west-1.example.com"
```

### Active region (blue/green)

```yaml
activeRegion: us-east-1   # other regions get 0 replicas; workload still exists for rollback
replicas: 3
```

Mutually exclusive with `hpa.enabled`.

### Resources

```yaml
resources:
  cpu: 250m           # request; limit = request × limitMultiplier
  memory: 512Mi       # request and limit
  limitMultiplier: 4  # default: 4, minimum: 1
```

### Volumes

```yaml
volumes:
  secrets:
    my-secret:
      mountPath: /run/secrets
      defaultMode: 420          # optional, default 420 (0644)
      files:                    # optional: mount individual keys as files
        - tls.crt
  configMaps:
    my-config:
      mountPath: /etc/config
  empty:
    scratch:
      mountPath: /tmp/scratch
  persistentVolumeClaims:
    my-pvc:
      mountPath: /data
      readOnly: false
```

### Environment

```yaml
# Bulk — injects all keys from a ConfigMap or Secret
envFrom:
  configMaps:
    shared-config: {}
  secrets:
    my-secret: {}

# Individual keys — validated against the cluster during real deployments
env:
  secrets:
    my-secret:
      - key: DB_PASSWORD
        variable: DATABASE_PASSWORD
  configMaps:
    my-config:
      - key: LOG_LEVEL
        variable: LOG_LEVEL
```

The chart's own ConfigMap is always appended to `envFrom` automatically when `configMap.enabled: true`.

### ConfigMap

```yaml
configMap:
  enabled: true
  data:
    LOG_LEVEL: info
    PORT: "{{ .Values.port }}"   # Go template expressions supported
```

`configMap.data` is required when `configMap.enabled: true`.

### Probes

Each probe must have exactly one of `httpGet` or `exec`.

```yaml
probes:
  readiness:
    httpGet:
      path: /health/ready
    initialDelaySeconds: 40
    periodSeconds: 30
  liveness:
    httpGet:
      path: /health/live
  startup:
    exec:
      command: ["sh", "-c", "test -f /tmp/ready"]
    failureThreshold: 30
    periodSeconds: 10
```

### Strategy

```yaml
strategy:
  type: RollingUpdate     # default; or Recreate / OnDelete
  maxUnavailable: "25%"   # default
  maxSurge: "25%"         # default (Deployment only)
  partition: 2            # StatefulSet canary rollouts (replaces maxUnavailable/maxSurge)
```

### Service

```yaml
service:
  enabled: true
# port and portName are shared with the workload container
```

### PDB

```yaml
pdb:
  enabled: true
  minAvailable: 1       # use minAvailable or maxUnavailable, not both
```

### CronJob

```yaml
cronJob:
  enabled: true
  schedule: "0 * * * *"         # required when enabled
  concurrencyPolicy: Forbid      # Allow | Forbid | Replace (default: Forbid)
  successfulJobsHistory: 3       # default: 3
  failedJobsHistory: 1           # default: 1
  entrypoint:
    command: ["/bin/sh"]
    args: ["-c", "echo hello"]
```

Shares `envFrom`, `env`, `volumes`, `resources`, `initContainers`, and `image` with the workload.

### HPA

Cannot be combined with `activeRegion`.

```yaml
hpa:
  enabled: true
  maxReplicas: 10    # required when enabled; minReplicas comes from root replicas
  resources:
    cpu:
      averageUtilization: 70
    memory:
      averageUtilization: 80
```

### StatefulSet

```yaml
workload:
  type: StatefulSet

statefulSet:
  headlessService: my-headless-svc   # required — must exist in cluster before deploy
  podManagementPolicy: OrderedReady  # OrderedReady | Parallel
  volumeClaimTemplates:
    - name: data
      size: 10Gi
      storageClass: fast
      accessMode: ReadWriteOnce      # default
```

### PersistentVolumeClaims

Standalone PVCs — one per entry, not per pod replica.

```yaml
persistentVolumeClaims:
  shared-storage:
    size: 50Gi
    storageClass: standard
    accessMode: ReadWriteMany
```

### Init containers and sidecars

```yaml
initContainers:
  - name: db-migrate
    image:
      url: myrepo/migrator
      tag: "1.0"
    command: ["/bin/sh"]
    args: ["-c", "run-migrations.sh"]
    resources:
      cpu: 100m
      memory: 128Mi

sidecars:
  - name: log-shipper
    image:
      url: fluent/fluent-bit
      tag: "3.0"
    resources:
      cpu: 50m
      memory: 64Mi
```

Both share the pod's volumes.

### Scheduling

```yaml
# Pin pods to nodes that match all of the given labels.
nodeSelector:
  kubernetes.io/arch: amd64
  node-role.kubernetes.io/worker: "true"

# Allow pods to be scheduled on nodes with matching taints.
tolerations:
  - key: nvidia.com/gpu
    operator: Exists          # Exists | Equal
    effect: NoSchedule        # NoSchedule | PreferNoSchedule | NoExecute
  - key: node.kubernetes.io/not-ready
    operator: Exists
    effect: NoExecute
    tolerationSeconds: 300    # evict after 5 minutes on not-ready nodes

# Simplified affinity API — labelSelector is auto-injected for podAntiAffinity.
affinity:
  nodeAffinity:
    required:                             # AND-ed inside one nodeSelectorTerm
      - key: topology.kubernetes.io/zone
        operator: In                      # In | NotIn | Exists | DoesNotExist | Gt | Lt
        values: [us-east-1a, us-east-1b]
    preferred:
      - weight: 50
        key: kubernetes.io/arch
        operator: In
        values: [amd64]
  podAntiAffinity:
    preferred:                            # labelSelector auto-injected from chart name
      - weight: 100
        topologyKey: kubernetes.io/hostname
    required:
      - topologyKey: topology.kubernetes.io/zone

# Schedule at higher or lower priority relative to other pods.
# The named PriorityClass must exist in the cluster.
priorityClassName: high-priority
```

### Security

`podSecurityContext` applies to all containers in the pod. `containerSecurityContext` applies to the main container only. To set a security context on a sidecar or init container, add `containerSecurityContext` inside that container's spec.

```yaml
podSecurityContext:
  runAsNonRoot: true
  runAsUser: 1000
  runAsGroup: 3000
  fsGroup: 2000
  fsGroupChangePolicy: OnRootMismatch   # Always | OnRootMismatch
  seccompProfile:
    type: RuntimeDefault                # RuntimeDefault | Localhost | Unconfined

containerSecurityContext:
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  capabilities:
    drop: [ALL]
    add: [NET_BIND_SERVICE]
```

### ServiceAccount

```yaml
serviceAccount:
  create: true
  name: my-sa    # optional; defaults to chart name
```

### Miscellaneous

```yaml
portName: http                      # named port on container and service (default: "http")
revisionHistoryLimit: 1             # default: 1
terminationGracePeriodSeconds: 30   # default: 30

labels:                             # extra labels on workload and pod
  team: platform

annotations:                        # per-resource annotation blocks
  deployment:
    my-annotation: value
  configMap:
    my-annotation: value
  service:
    my-annotation: value
  cronJob:
    my-annotation: value

topologySpreadConstraints: []       # defaults to hostname + zone spreading; set to [] to disable
```

### Raw manifests

An escape hatch for Kubernetes resources not supported by the chart. Each entry is rendered as a separate YAML document.

```yaml
rawManifests:
  # Plain passthrough — content is emitted verbatim
  - content: |
      apiVersion: networking.k8s.io/v1
      kind: IngressClass
      metadata:
        name: my-ingress-class
      spec:
        controller: example.com/ingress-controller

  # Templated — content is processed as a Go template with full access to .Values, .Release, .Chart
  - content: |
      apiVersion: v1
      kind: ConfigMap
      metadata:
        name: {{ .Release.Name }}-extra
      data:
        region: {{ .Values.global.region }}
    tpl: true
```

`tpl: false` is the default. Set `tpl: true` only when the content contains `{{ }}` expressions — without it, template markers are rendered literally.

---

## Umbrella charts

Depend on `helm-templates` multiple times using aliases — one per service. `global` values are shared automatically across all instances.

**`Chart.yaml`**:
```yaml
dependencies:
  - name: helm-templates
    version: 0.3.0
    repository: "oci://ghcr.io/yagelhayun/helm-charts"
    alias: api
  - name: helm-templates
    version: 0.3.0
    repository: "oci://ghcr.io/yagelhayun/helm-charts"
    alias: worker
```

**`values.yaml`**:
```yaml
global:
  region: us-east-1
  image:
    pullSecrets: [registry-credentials]

api:
  port: 8080
  replicas: 2
  image:
    url: myrepo/api
    tag: "1.0.0"
  resources:
    cpu: 500m
    memory: 512Mi
  workload:
    type: Deployment
  service:
    enabled: true

worker:
  port: 9090
  replicas: 5
  image:
    url: myrepo/worker
    tag: "1.0.0"
  resources:
    cpu: 250m
    memory: 256Mi
  workload:
    type: Deployment
  service:
    enabled: false
```

Each alias becomes the chart name used for resource naming. Schema validation applies independently per sub-chart.

---

## Schema validation

`values.schema.json` is validated by Helm before rendering. Unknown keys at any level are rejected (`additionalProperties: false` throughout). If Helm rejects a value, check the schema for allowed keys.

---

## GitOps (ArgoCD / Flux)

### Schema validation

Works as normal — ArgoCD and Flux both invoke `helm template` before applying, so `values.schema.json` is validated every sync. Misconfigured values are caught before anything reaches the cluster.

### Cluster lookups

ArgoCD and Flux render charts via `helm template`, which causes `core.isRealDeployment` to return `false`. This means **cluster lookups are silently skipped** at render time — the library will not verify that referenced Secrets, ConfigMaps, or PVCs exist before the manifests are applied.

The failure only surfaces later, when a pod can't start because a referenced resource is missing.

### Mitigation options

**CI validation with cluster credentials (recommended):** In your CI pipeline, run `helm template` with a read-only kubeconfig pointed at staging. With real cluster access, `core.isRealDeployment` activates and all lookups run, giving you the full failure behavior before merge.

```bash
helm template my-service . -f values.yaml --kube-context staging-readonly
```

**ArgoCD server-side diff:** `argocd app diff` dry-applies rendered manifests against the Kubernetes API. This catches invalid resource specs but does not catch missing Secrets referenced in `envFrom` (Kubernetes only validates those at pod schedule time).

**PreSync hook:** A Helm hook (`helm.sh/hook: pre-sync`) running a Job that checks for required Secrets/ConfigMaps before sync proceeds. Use this only if you have strict zero-downtime requirements — it adds complexity and a sync round-trip.

**Accept the limitation:** Treat schema validation as your pre-flight gate and let ArgoCD's sync health (pod `CrashLoopBackOff` / `OOMKilled`) surface missing dependencies post-apply. Practical for teams with fast feedback loops and non-critical workloads.
