# Helm Templates — Best Practices

Advanced usage patterns for `helm-templates`. These are opt-in techniques for non-trivial setups — standard single-service deployments don't need any of this.

---

## Umbrella charts

### Basic setup

Depend on `helm-templates` multiple times using aliases — one per service. `global` values are shared automatically across all instances.

`nameOverride` is **required** per sub-chart — all aliases share the same release name, so there is no safe automatic default. A clean convention is to mirror the chart name:

```yaml
# umbrella/Chart.yaml
dependencies:
  - name: helm-templates
    alias: api
    version: 0.8.0
    repository: "oci://ghcr.io/yagelhayun/helm-charts"
  - name: helm-templates
    alias: worker
    version: 0.8.0
    repository: "oci://ghcr.io/yagelhayun/helm-charts"
```

```yaml
# umbrella/values.yaml
global:
  region: us-east-1
  image:
    pullSecrets: [registry-credentials]

api:
  nameOverride: "{{ .Chart.Name }}"   # resolves to "api" at render time
  replicas: 2
  resources:
    cpu: 500m
    memory: 512Mi
  image:
    url: myrepo/api
    tag: "1.0.0"
  workload:
    type: Deployment
  service:
    enabled: true

worker:
  nameOverride: "{{ .Chart.Name }}"
  replicas: 5
  resources:
    cpu: 250m
    memory: 256Mi
  image:
    url: myrepo/worker
    tag: "1.0.0"
  workload:
    type: Deployment
  service:
    enabled: false
```

Schema validation applies independently per sub-chart. Each alias is validated against `values.schema.json` in isolation.

---

### Shared configuration subchart

For platform-wide config that every service needs (feature flags, platform endpoints, shared credentials), introduce a dedicated `common` alias with `workload.enabled: false`. It renders only a ConfigMap — no Deployment, no Service, no overhead. Other services consume it by name.

```yaml
# umbrella/Chart.yaml
dependencies:
  - name: helm-templates
    alias: common
    version: 0.8.0
    repository: "oci://ghcr.io/yagelhayun/helm-charts"
  - name: helm-templates
    alias: api
    version: 0.8.0
    repository: "oci://ghcr.io/yagelhayun/helm-charts"
  - name: helm-templates
    alias: worker
    version: 0.8.0
    repository: "oci://ghcr.io/yagelhayun/helm-charts"
```

```yaml
# umbrella/values.yaml
global:
  region: us-east-1

common:
  nameOverride: common
  workload:
    enabled: false
  configMap:
    enabled: true
    as: env
    data:
      PLATFORM_ENV: production
      LOG_FORMAT: json
      TRACING_ENDPOINT: "http://otel-collector:4317"

api:
  nameOverride: "{{ .Chart.Name }}"
  replicas: 2
  resources:
    cpu: 500m
    memory: 512Mi
  image:
    url: myrepo/api
    tag: "1.0.0"
  workload:
    type: Deployment
  service:
    enabled: true
  envFrom:
    configMaps:
      common: {}   # injects all keys from the 'common' ConfigMap as env vars

worker:
  nameOverride: "{{ .Chart.Name }}"
  replicas: 3
  resources:
    cpu: 250m
    memory: 256Mi
  image:
    url: myrepo/worker
    tag: "1.0.0"
  workload:
    type: Deployment
  service:
    enabled: false
  envFrom:
    configMaps:
      common: {}
```

The `common` ConfigMap name matches its `nameOverride`. Any service that needs platform config references it by that name in `envFrom.configMaps` or `volumes.configMaps`. Updating platform config means changing one block in `values.yaml` — not touching every service.

> Cluster lookups for `envFrom.configMaps` validate that the referenced ConfigMap exists. During a fresh install, render `common` first (`helm install common ./umbrella --set ...`) or use `global.ignoreLookup: "true"` for the initial rollout.

---

## Structured config serialization

Apps that accept JSON config (most Go, Node.js, and Python services) benefit from receiving a single structured env var rather than a flat list of individually named vars. Define the config as a proper YAML object in values and serialize it with `toJson`.

```yaml
# values.yaml

# Define config as structured YAML — easier to read and review in PRs
connectionPool:
  host: db.us-east-1.example.com
  port: 5432
  maxConnections: 20
  minConnections: 2
  ssl: true
  connectTimeout: 5

featureFlags:
  newCheckout: true
  darkMode: false
  maxRetries: 3

configMap:
  enabled: true
  data:
    DB_CONFIG: '{{ .Values.connectionPool | toJson }}'
    FEATURE_FLAGS: '{{ .Values.featureFlags | toJson }}'
```

The app receives:
```
DB_CONFIG={"host":"db.us-east-1.example.com","port":5432,"maxConnections":20,...}
FEATURE_FLAGS={"newCheckout":true,"darkMode":false,"maxRetries":3}
```

**Why this is better than flat vars for complex config:**
- Structured PR diffs — the YAML object reads naturally, no mental mapping between `DB_MAX_CONNECTIONS` and its purpose
- Adding a new field to the object doesn't require a new env var and schema change
- The app's config struct maps directly to the JSON shape

**Combining with regions** — override only the fields that change per region:

```yaml
connectionPool:
  host: db.default.example.com
  port: 5432
  maxConnections: 20
  ssl: true

regions:
  us-east-1:
    connectionPool:
      host: db.us-east-1.example.com
      maxConnections: 50    # higher capacity in primary region
  us-west-1:
    connectionPool:
      host: db.us-west-1.example.com

configMap:
  enabled: true
  data:
    DB_CONFIG: '{{ .Values.connectionPool | toJson }}'
```

`core.general.config` deep-merges the region override into the base object before the template expression runs — only the fields you override change, the rest inherit from the base.

---

### Lists as delimited strings

For values that need to be a comma-separated (or otherwise delimited) string — allowed origins, trusted IPs, feature flag names — define them as a proper YAML list and join at serialization time:

```yaml
allowedOrigins:
  - https://app.example.com
  - https://admin.example.com
  - https://staging.example.com

configMap:
  enabled: true
  data:
    ALLOWED_ORIGINS: '{{ .Values.allowedOrigins | join "," }}'
    # Result: https://app.example.com,https://admin.example.com,...
```

The list is easy to review and diff in Git. The app receives the format it expects.
