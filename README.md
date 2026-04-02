# Helm Templates

A ready-to-use application Helm chart that wraps `helm-core`. It generates Kubernetes resources from a structured `values.yaml` without writing any templates.

---

## Quick start

```bash
helm dependency update
helm template my-service . -f values.yaml
helm install my-service . -f values.yaml
```

For testing without a cluster, add `global.ignoreLookup: "true"` to your values.

---

## Resources

Each resource is **opt-in** via an `enabled` flag. Resources with `enabled: false` produce no output.

| Resource | Key | Notes |
|----------|-----|-------|
| ConfigMap | `configMap.enabled` | Chart's own config, auto-mounted as envFrom |
| Deployment | `deployment.enabled` | Main workload |
| Service | `service.enabled` | ClusterIP, targets the main port |
| CronJob | `cronJob.enabled` | Shares all env/volume config with deployment |
| HPA | `hpa.enabled` | Disabled automatically when `activeRegion` is set |

---

## Full values reference

### Required

```yaml
global:
  region: us-east-1   # must match a key in regions (or be the only region)

port: 8080            # main container and service port
replicas: 2           # desired replica count
resources:
  cpu: 250m
  memory: 512Mi
```

### Global scope

Values under `global` are merged into every chart in an umbrella. Local values take priority over global ones.

```yaml
global:
  region: us-east-1          # required
  ignoreLookup: "true"       # set to bypass cluster lookups (CI / local rendering)
  commit: "abc123"           # optional — adds a 'commit' label to deployments
  image:
    url: myrepo/base
    tag: latest
    pullPolicy: IfNotPresent
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
  url: myrepo/myapp       # required if global.image.url not set
  tag: "1.2.3"            # default: "latest"
  pullPolicy: IfNotPresent # Always | IfNotPresent | Never
  pullSecrets:
    - my-registry-secret
```

### Regions

Override any value for a specific region. The active region is determined by `global.region`.

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

Set `activeRegion` to deploy with 0 replicas in all other regions. The Deployment still exists (preserving rollback history) but runs nothing.

```yaml
activeRegion: us-east-1   # only this region gets replicas > 0
replicas: 3
```

### Resources

```yaml
resources:
  cpu: 250m          # request (e.g. 250m, 1, 1.5)
  memory: 512Mi      # request and limit
  limitMultiplier: 4 # CPU limit = request × multiplier (default: 4)
```

### Volumes

```yaml
volumes:
  secrets:
    my-secret:
      mountPath: /run/secrets
      defaultMode: 420      # optional, default 0644
      files:                # optional: mount specific keys as files
        - tls.crt
        - tls.key
  configMaps:
    my-config:
      mountPath: /etc/config
  empty:
    scratch:
      mountPath: /tmp/scratch  # optional for emptyDir
```

### Environment — envFrom (bulk)

Injects all keys from a ConfigMap or Secret as environment variables.

```yaml
envFrom:
  configMaps:
    shared-config: {}
  secrets:
    my-secret: {}
```

The chart's own ConfigMap (if `configMap.enabled: true`) is always appended automatically.

### Environment — env (individual keys)

Maps specific keys to named environment variables. Keys are validated against the cluster during real deployments.

```yaml
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

### ConfigMap

```yaml
configMap:
  enabled: true
  data:
    LOG_LEVEL: info
    MAX_CONNECTIONS: "100"
    # Go template expressions are supported:
    PORT: "{{ .Values.port }}"
```

Required when `enabled: true`. Supports region overrides via `regions.<region>.configMap.data`.

### Probes

Each probe must have either `httpGet` or `exec`, not both.

```yaml
probes:
  readiness:
    httpGet:
      path: /health/ready
      scheme: HTTP        # optional, default HTTP
    failureThreshold: 3   # default 3
    initialDelaySeconds: 40
    periodSeconds: 30
    successThreshold: 1
    timeoutSeconds: 20
  liveness:
    httpGet:
      path: /health/live
  startup:
    exec:
      command: ["sh", "-c", "test -f /tmp/ready"]
    failureThreshold: 30
    periodSeconds: 10
```

### Deployment options

```yaml
deployment:
  enabled: true

portName: http                      # named port on container and service (default: "http")
serviceAccount: my-sa               # default: "default"
terminationGracePeriodSeconds: 60   # default: 30
revisionHistoryLimit: 3             # default: 1
labels:                             # extra labels on Deployment and pod
  team: platform
annotations:
  deployment:
    my-annotation: value
  configMap:
    my-annotation: value
  service:
    my-annotation: value
```

### Service

```yaml
service:
  enabled: true
# port and portName are shared with the deployment
```

### CronJob

```yaml
cronJob:
  enabled: true
  schedule: "0 * * * *"         # required when enabled
  concurrencyPolicy: Forbid     # Allow | Forbid | Replace (default: Forbid)
  successfulJobsHistory: 3      # default: 3
  failedJobsHistory: 1          # default: 1
  entrypoint:
    command: ["/bin/sh"]
    args: ["-c", "echo hello"]
```

The CronJob shares `envFrom`, `env`, `volumes`, `resources`, `initContainers`, and `image` with the Deployment.

### HPA

```yaml
hpa:
  enabled: true
  maxReplicas: 10    # required when enabled; minReplicas comes from root `replicas`
  resources:
    cpu:
      averageUtilization: 70
    memory:
      averageUtilization: 80
```

The HPA is suppressed automatically when `activeRegion` is set (region-based scaling and HPA are mutually exclusive).

### Init containers

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
    envFrom:
      secrets:
        db-credentials: {}
```

Init containers share the pod's volumes. Declare any volumes they need in the root `volumes` config.

### Sidecars

```yaml
sidecars:
  - name: log-shipper
    image:
      url: fluent/fluent-bit
      tag: "3.0"
    port: 2020
    portName: metrics
    resources:
      cpu: 50m
      memory: 64Mi
    volumes:
      empty:
        log-buffer:
          mountPath: /var/log/buffer
```

### Full example

```yaml
global:
  region: us-east-1

port: 8080
portName: http
replicas: 2

image:
  url: myrepo/my-service
  tag: "2.1.0"
  pullSecrets:
    - registry-credentials

resources:
  cpu: 500m
  memory: 1Gi
  limitMultiplier: 2

deployment:
  enabled: true

service:
  enabled: true

configMap:
  enabled: true
  data:
    LOG_LEVEL: info
    FEATURE_FLAGS: "true"

regions:
  us-east-1:
    configMap:
      data:
        API_ENDPOINT: "https://api.us-east-1.example.com"
  us-west-1:
    configMap:
      data:
        API_ENDPOINT: "https://api.us-west-1.example.com"

probes:
  readiness:
    httpGet:
      path: /health/ready
  liveness:
    httpGet:
      path: /health/live

hpa:
  enabled: true
  maxReplicas: 10
  resources:
    cpu:
      averageUtilization: 70
```

---

## Schema validation

`values.schema.json` is validated by Helm before rendering. Unknown keys at any level are rejected. If you add a new top-level key and Helm rejects it, check the schema.

---

## Running tests

```bash
cd helm-templates
helm dependency update
helm template my-service . -f test/common.values.yaml -f test/prod.values.yaml
```
