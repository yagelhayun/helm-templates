# AI Skills: Deploying Services with helm-templates

This document is a complete reference for generating and modifying Helm chart values files that target the `helm-templates` chart. Read it fully before writing any YAML.

---

## What this system is

`helm-templates` is a platform Helm chart. Developers do **not** write Kubernetes manifests or Helm templates. Instead, they write a `values.yaml` file that the platform chart renders into Kubernetes resources. The chart handles Deployments, Services, ConfigMaps, CronJobs, and HPAs — all configured through values.

The chart is backed by a library chart called `helm-core` that performs region-aware config merging, cluster validation, and all rendering logic.

---

## Values file structure

A values file has one required section (`global`) and several optional sections. Most deployments need 6-8 top-level keys.

```
global.*           — shared platform settings (region, image fallback, etc.)
port               — main container and service port (required)
replicas           — desired replica count (required)
resources.*        — CPU and memory (required)
image.*            — container image
portName           — named port (default: "http")
activeRegion       — enables blue/green deployment
regions.*          — per-region value overrides
configMap.*        — this service's ConfigMap
deployment.*       — enable/disable Deployment
service.*          — enable/disable Service
cronJob.*          — enable/disable and configure CronJob
hpa.*              — enable/disable and configure HPA
probes.*           — readiness / liveness / startup probes
volumes.*          — volumes to mount
envFrom.*          — bulk environment from ConfigMaps/Secrets
env.*              — individual env vars from ConfigMap/Secret keys
initContainers     — init container list
sidecars           — sidecar container list
labels             — extra labels
annotations.*      — per-resource annotation blocks
serviceAccount     — service account name
revisionHistoryLimit
terminationGracePeriodSeconds
```

---

## Rules you must follow

1. **`global.region` is always required.** Every values file must include it. Its value must match one of the keys in the `regions` block (or be the only region used).

2. **`port`, `replicas`, and `resources` are always required** at the top level.

3. **Resources for `resources.cpu` and `resources.memory` are always required** when `resources` is present. `resources.limitMultiplier` is optional (default 4).

4. **Every resource (Deployment, Service, ConfigMap, CronJob, HPA) requires `enabled: true/false`.** If you omit the block entirely, the resource is not rendered. Always explicitly set `enabled`.

5. **`configMap.data` is required when `configMap.enabled: true`.** Values must be strings, numbers, or booleans.

6. **`cronJob.schedule` is required when `cronJob.enabled: true`.**

7. **`hpa.maxReplicas` is required when `hpa.enabled: true`.** `minReplicas` is taken from the top-level `replicas` key.

8. **Probes must have exactly one of `httpGet` or `exec`.** Never both.

9. **`additionalProperties: false` is enforced across the entire schema.** Do not invent new top-level keys or keys inside known objects — they will be rejected by Helm at install time. Only use keys documented here.

10. **`hpa.enabled` and `activeRegion` are mutually exclusive.** When `activeRegion` is set, the HPA is suppressed regardless of `hpa.enabled`. Do not set both.

11. **Init containers and sidecars share the pod's volumes.** Any volume an init container or sidecar needs must also be declared in the top-level `volumes` config.

12. **`global.ignoreLookup: "true"` must be set when rendering without a cluster** (CI, local testing, dry-run). Without it, `core.container.env` and volume helpers will fail trying to look up secrets/configmaps from the cluster.

---

## Annotated minimal example

```yaml
# Always required
global:
  region: us-east-1

port: 8080
replicas: 2
resources:
  cpu: 250m
  memory: 512Mi

image:
  url: myrepo/my-service
  tag: "1.0.0"

deployment:
  enabled: true

service:
  enabled: true

configMap:
  enabled: true
  data:
    LOG_LEVEL: info
```

---

## Annotated full example

```yaml
global:
  region: us-east-1
  commit: "abc1234"              # adds a 'commit' label to the deployment
  image:
    pullSecrets:
      - platform-registry        # registry pull secret name (must exist in cluster)
  envFrom:
    secrets:
      platform-tls: {}           # injected into every pod from the platform level
  volumes:
    secrets:
      platform-tls:
        mountPath: /run/tls
  env:
    secrets:
      platform-secret:
        - key: PLATFORM_TOKEN
          variable: PLATFORM_TOKEN

port: 8080
portName: http                   # names the port; used for service targetPort too
replicas: 3
revisionHistoryLimit: 5
terminationGracePeriodSeconds: 60
serviceAccount: my-service-sa

image:
  url: myrepo/my-service
  tag: "2.1.0"
  pullPolicy: IfNotPresent
  pullSecrets:
    - my-registry-secret         # merged with global.image.pullSecrets

resources:
  cpu: 500m
  memory: 1Gi
  limitMultiplier: 2             # CPU limit = 500m × 2 = 1000m

activeRegion: us-east-1          # set replicas=0 in all other regions

regions:
  us-east-1:
    configMap:
      data:
        DB_HOST: "db.us-east-1.example.com"
        CACHE_HOST: "redis.us-east-1.example.com"
  us-west-1:
    configMap:
      data:
        DB_HOST: "db.us-west-1.example.com"
        CACHE_HOST: "redis.us-west-1.example.com"

configMap:
  enabled: true
  data:
    LOG_LEVEL: info
    MAX_POOL_SIZE: "20"
    PORT: "{{ .Values.port }}"   # Go template expressions are supported

deployment:
  enabled: true

service:
  enabled: true

volumes:
  secrets:
    my-service-creds:
      mountPath: /run/secrets
      files:                     # mount specific keys as individual files
        - db-password
        - api-key
  empty:
    tmp-upload:
      mountPath: /tmp/uploads    # emptyDir

envFrom:
  configMaps:
    shared-platform-config: {}
  secrets:
    my-service-creds: {}

env:
  secrets:
    my-service-creds:
      - key: DB_PASSWORD
        variable: DATABASE_PASSWORD
  configMaps:
    feature-flags:
      - key: DARK_MODE
        variable: FEATURE_DARK_MODE

probes:
  readiness:
    httpGet:
      path: /health/ready
      scheme: HTTP
    failureThreshold: 3
    initialDelaySeconds: 40
    periodSeconds: 30
    successThreshold: 1
    timeoutSeconds: 20
  liveness:
    httpGet:
      path: /health/live
    failureThreshold: 3
    initialDelaySeconds: 60
    periodSeconds: 30
  startup:
    httpGet:
      path: /health/started
    failureThreshold: 30         # gives 5 minutes (30 × 10s) to start
    periodSeconds: 10

initContainers:
  - name: db-migrate
    image:
      url: myrepo/migrator
      tag: "1.0"
    command: ["/bin/sh"]
    args: ["-c", "/app/run-migrations.sh"]
    resources:
      cpu: 100m
      memory: 128Mi
    envFrom:
      secrets:
        my-service-creds: {}

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

hpa:
  enabled: false                 # disabled because activeRegion is set
  maxReplicas: 10
  resources:
    cpu:
      averageUtilization: 70
    memory:
      averageUtilization: 80

cronJob:
  enabled: true
  schedule: "0 2 * * *"         # daily at 2am
  concurrencyPolicy: Forbid
  successfulJobsHistory: 3
  failedJobsHistory: 1
  entrypoint:
    command: ["/bin/sh"]
    args: ["-c", "/app/cleanup.sh"]

labels:
  team: backend
  cost-center: "1234"

annotations:
  deployment:
    reloader.stakater.com/auto: "true"
  service:
    service.beta.kubernetes.io/aws-load-balancer-type: nlb
```

---

## Multi-chart (umbrella) pattern

When deploying multiple services together, use an umbrella chart. Each sub-chart has its own values namespace. Shared configuration lives under `global`:

```yaml
# umbrella/values.yaml
global:
  region: us-east-1
  image:
    pullSecrets:
      - platform-registry
  envFrom:
    secrets:
      platform-secret: {}

worker:                          # matches the sub-chart name
  port: 9999
  replicas: 5
  resources:
    cpu: 1
    memory: 4Gi
  image:
    url: myrepo/worker
    tag: staging
  deployment:
    enabled: true
  service:
    enabled: true
  configMap:
    enabled: true
    data:
      MODE: worker

master:
  port: 1111
  replicas: 2
  resources:
    cpu: 3
    memory: 8Gi
  image:
    url: myrepo/master
    tag: stable
  deployment:
    enabled: true
  service:
    enabled: true
  configMap:
    enabled: true
    data:
      MODE: master
```

Each sub-chart's values are merged with `global` — `global` loses to chart-specific values on conflict.

---

## Common generation tasks

### Add a new environment variable from a secret

```yaml
env:
  secrets:
    my-secret:                   # secret must exist in the cluster
      - key: SECRET_KEY_NAME     # key inside the secret
        variable: ENV_VAR_NAME   # name in the container
```

### Add a region-specific config value

```yaml
configMap:
  enabled: true
  data:
    SHARED_KEY: shared-value     # same in all regions

regions:
  us-east-1:
    configMap:
      data:
        REGIONAL_KEY: "east-value"
  us-west-1:
    configMap:
      data:
        REGIONAL_KEY: "west-value"
```

### Mount a secret as files

```yaml
volumes:
  secrets:
    my-tls-secret:
      mountPath: /run/tls
      files:
        - tls.crt
        - tls.key
# Results in:
# /run/tls/tls.crt  (subPath: tls.crt)
# /run/tls/tls.key  (subPath: tls.key)
```

### Configure a startup probe for a slow-starting service

```yaml
probes:
  startup:
    httpGet:
      path: /health
    failureThreshold: 60   # 60 × 10s = 10 minutes to start
    periodSeconds: 10
  liveness:               # liveness only runs after startup succeeds
    httpGet:
      path: /health
```

### Add an HPA (no activeRegion)

```yaml
# Do NOT set activeRegion when using HPA
replicas: 2              # becomes minReplicas

hpa:
  enabled: true
  maxReplicas: 20
  resources:
    cpu:
      averageUtilization: 70
```

### Disable a resource

```yaml
cronJob:
  enabled: false
  schedule: "@daily"     # schedule can still be present; ignored when disabled
```

---

## What NOT to do

- Do not add keys not listed in this document — they will be rejected by schema validation.
- Do not set `hpa.enabled: true` and `activeRegion` together.
- Do not use `configMap.enabled: true` without `configMap.data`.
- Do not reference a secret/configmap in `env` that doesn't exist in the target cluster (lookup will fail during real deployments).
- Do not use `exec` and `httpGet` in the same probe.
- Do not set `resources.limitMultiplier` below `1`.
- Do not set `revisionHistoryLimit` to `0` unless you explicitly want to disable rollback.
