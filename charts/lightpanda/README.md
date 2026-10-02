# Lightpanda Browser Helm Chart

A Helm chart to deploy [Lightpanda Browser](https://lightpanda.io) — an
AI-native, headless browser written in Zig — as a CDP/WebDriver Bidi server
on Kubernetes, for web automation and scraping with Puppeteer, Playwright,
or any CDP-compatible client.

- **Upstream project:** https://github.com/lightpanda-io/browser

## Introduction

This chart deploys Lightpanda's `serve` mode, which exposes a
CDP (Chrome DevTools Protocol) and/or WebDriver Bidi endpoint over a
WebSocket connection. It includes production-ready defaults: a restricted
security context, liveness/readiness/startup probes, optional autoscaling,
`PodDisruptionBudget`, `NetworkPolicy`, and Prometheus `ServiceMonitor`
support.

## Prerequisites

- Kubernetes 1.23+
- Helm 3.8+
- (Optional) [Prometheus Operator](https://github.com/prometheus-operator/prometheus-operator)
  CRDs installed, if you enable `metrics.serviceMonitor.enabled`
- (Optional) An Ingress controller (e.g. ingress-nginx), if you enable `ingress.enabled`

## Installing the chart

From the published Helm repository (see the [repository root README](../../README.md)
for one-time setup):

```console
helm repo add lightpanda-charts https://laghoule.github.io/lightpanda_browser/
helm repo update
helm install my-lightpanda lightpanda-charts/lightpanda
```

Or directly from a local checkout of this repository:

```console
helm install my-lightpanda ./charts/lightpanda
```

Both commands deploy Lightpanda on the Kubernetes cluster with the default
configuration. See [Configuration](#configuration) below for the parameters
that can be configured during installation.

## Uninstalling the chart

```console
helm uninstall my-lightpanda
```

## Values validation

This chart ships a [`values.schema.json`](./values.schema.json). Helm
automatically validates your overrides against it on `install`, `upgrade`,
`template`, and `lint`, catching mistakes early (wrong types, invalid enum
values such as an unsupported `service.type`, out-of-range ports, etc.).

```console
helm lint ./charts/lightpanda
helm template ./charts/lightpanda --set service.type=Bogus
# Error: values don't meet the specifications of the schema(s)...
```

## Connecting to the browser

Once deployed, port-forward the service and point your CDP client at it:

```console
kubectl port-forward svc/my-lightpanda-lightpanda 9222:9222
```

```js
import puppeteer from 'puppeteer-core';

const browser = await puppeteer.connect({
  browserWSEndpoint: 'ws://127.0.0.1:9222',
});
```

See the [Notes](#post-install-notes) printed after `helm install` for
connection instructions tailored to your `service.type`/`ingress` settings.

## Configuration

The following table lists the main configurable parameters and their
defaults. See [`values.yaml`](./values.yaml) for the full, authoritative
list and inline comments.

### Global / deployment

| Parameter | Description | Default |
| --- | --- | --- |
| `replicaCount` | Number of pod replicas (ignored if `autoscaling.enabled`) | `1` |
| `image.repository` | Container image repository | `lightpanda/browser` |
| `image.tag` | Image tag (defaults to `.Chart.AppVersion` if empty) | `"1.0.0"` |
| `image.digest` | Image digest, takes precedence over `tag` if set | `""` |
| `image.pullPolicy` | Image pull policy | `IfNotPresent` |
| `imagePullSecrets` | Image pull secrets | `[]` |
| `nameOverride` | Override the chart name | `""` |
| `fullnameOverride` | Override the full release name | `""` |

### Server (`lightpanda serve` flags)

| Parameter | Description | Default |
| --- | --- | --- |
| `server.host` | Bind address inside the container | `0.0.0.0` |
| `server.port` | CDP/WebDriver listening port | `9222` |
| `server.protocols` | Protocols to enable: `cdp` and/or `webdriver` | `[cdp]` |
| `server.advertiseHost` | Host advertised to clients (behind a proxy/ingress) | `""` |
| `server.maxConnections` | Max concurrent CDP connections | `16` |
| `server.maxPendingConnections` | Max pending CDP connections | `128` |
| `server.maxMessageSize` | Max CDP message size, in bytes | `1048576` |
| `server.maxHttpMessageSize` | Max HTTP message size, in bytes | `4096` |
| `server.httpSessionTimeout` | HTTP session timeout, in seconds | `60` |

> **Note:** The bundled `livenessProbe`/`readinessProbe`/`startupProbe` use
> an HTTP `GET /json/version` on the `cdp` port, which requires `cdp` to be
> present in `server.protocols`. Probes are automatically omitted from the
> rendered manifests if you only enable `webdriver`; provide your own probes
> via a values override in that case if needed.

### Browser configuration

| Parameter | Description | Default |
| --- | --- | --- |
| `config.logLevel` | Log level: `debug`, `info`, `warn`, `error`, `fatal` | `info` |
| `config.logFormat` | Log format: `logfmt`, `json`, `pretty` | `logfmt` |
| `config.obeyRobots` | Respect `robots.txt` | `true` |
| `config.blockPrivateNetworks` | Block requests to private/internal networks | `false` |
| `config.cors` | Enforce CORS checks on cross-origin fetch/XHR (`false` passes `--disable-features cors`) | `true` |
| `config.corsStoreEntryLimit` | Max entries in the CORS store (`0` = engine default) | `1000` |
| `config.loadResources` | Extra resource types to load (e.g. `iframe`, `image`) | `[]` |
| `config.watchdogMs` | Watchdog timeout, in milliseconds | `30000` |
| `config.httpTimeout` | HTTP request timeout, in milliseconds | `15000` |
| `config.httpConnectTimeout` | HTTP connect timeout, in milliseconds | `8000` |
| `config.httpMaxConcurrent` | Max concurrent outbound HTTP requests | `40` |
| `config.timezone` | Timezone reported to pages | `UTC` |
| `config.locale` | Locale reported to pages | `en-US` |
| `config.userAgent` | Override the default User-Agent | `""` |
| `config.userAgentSuffix` | Suffix appended to the default User-Agent | `""` |
| `config.v8MaxHeapMb` | V8 max heap size, in MB (`0` = engine default) | `0` |
| `config.blockUrls` | URL patterns to block | `[]` |
| `config.blockCidrs` | CIDR ranges to block | `[]` |
| `config.adblockLists` | Adblock list URLs/paths to load | `[]` |
| `config.proxy.enabled` | Route outbound traffic through an HTTP proxy | `false` |
| `config.proxy.httpProxy` | Proxy URL | `""` |
| `config.proxy.proxyBearerToken` | Bearer token for the proxy, set as plain env var | `""` |
| `config.proxy.existingSecret` | Existing Secret name holding the proxy bearer token | `""` |
| `config.proxy.existingSecretKey` | Key within `existingSecret` (default `token`) | `""` |

### Environment, extra args/env

| Parameter | Description | Default |
| --- | --- | --- |
| `env.disableTelemetry` | Set `LIGHTPANDA_DISABLE_TELEMETRY=true` | `true` |
| `env.disableCoreDump` | Set `LIGHTPANDA_DISABLE_CORE_DUMP=true` | `true` |
| `extraArgs` | Extra CLI arguments appended to the container command | `[]` |
| `extraEnv` | Extra environment variables (list of `name`/`value(From)`) | `[]` |
| `extraEnvFrom` | Extra `envFrom` sources (ConfigMaps/Secrets) | `[]` |
| `extraVolumes` | Extra pod volumes | `[]` |
| `extraVolumeMounts` | Extra container volume mounts | `[]` |

### Security

| Parameter | Description | Default |
| --- | --- | --- |
| `podSecurityContext` | Pod-level `securityContext` | non-root, uid/gid `1000` |
| `securityContext` | Container-level `securityContext` | read-only rootfs, all capabilities dropped |
| `tmpVolume.enabled` | Mount an `emptyDir` at `/tmp` (required since the root filesystem is read-only) | `true` |
| `tmpVolume.sizeLimit` | Optional size limit for the `/tmp` `emptyDir` | `""` (unbounded) |
| `serviceAccount.create` | Create a dedicated `ServiceAccount` | `true` |
| `serviceAccount.automount` | Automount the service account token | `true` |
| `serviceAccount.name` | Existing/override `ServiceAccount` name | `""` |
| `networkPolicy.enabled` | Create a `NetworkPolicy` | `false` |
| `networkPolicy.ingress` | Ingress rules (defaults to allow-all from any source on the server port) | `[]` |
| `networkPolicy.egress` | Egress rules (defaults to allow-all — the browser needs outbound internet access) | `[]` |

### Networking

| Parameter | Description | Default |
| --- | --- | --- |
| `service.type` | Kubernetes Service type | `ClusterIP` |
| `service.port` | Service port | `9222` |
| `service.nodePort` | Fixed `NodePort` (only used when `service.type=NodePort`) | `null` |
| `service.annotations` | Extra Service annotations | `{}` |
| `ingress.enabled` | Create an `Ingress` | `false` |
| `ingress.className` | `IngressClass` name | `""` |
| `ingress.hosts` | Ingress hosts/paths | see `values.yaml` |
| `ingress.tls` | Ingress TLS configuration | `[]` |

> CDP/WebDriver are long-lived WebSocket connections. No `ingress.annotations`
> are set by default, since the required tuning is provider-specific — see
> [Ingress & WebSockets](#ingress--websockets) below.

### Scaling & availability

| Parameter | Description | Default |
| --- | --- | --- |
| `resources` | Container resource requests/limits | `100m`/`128Mi` requests, `1000m`/`512Mi` limits |
| `autoscaling.enabled` | Create a `HorizontalPodAutoscaler` | `false` |
| `autoscaling.minReplicas` / `maxReplicas` | HPA replica bounds | `1` / `5` |
| `autoscaling.targetCPUUtilizationPercentage` | Target CPU utilization | `80` |
| `autoscaling.targetMemoryUtilizationPercentage` | Target memory utilization | `80` |
| `podDisruptionBudget.enabled` | Create a `PodDisruptionBudget` | `false` |
| `podDisruptionBudget.minAvailable` | Minimum available pods | `1` |
| `nodeSelector` / `tolerations` / `affinity` / `topologySpreadConstraints` | Standard scheduling controls | `{}` / `[]` / `{}` / `[]` |

### Observability

| Parameter | Description | Default |
| --- | --- | --- |
| `metrics.enabled` | Expose Prometheus metrics (disables `--disable-metrics` flag) | `true` |
| `metrics.serviceMonitor.enabled` | Create a Prometheus Operator `ServiceMonitor` | `false` |
| `metrics.serviceMonitor.namespace` | Namespace for the `ServiceMonitor` | release namespace |
| `metrics.serviceMonitor.interval` | Scrape interval | `30s` |
| `metrics.serviceMonitor.scrapeTimeout` | Scrape timeout | `10s` |
| `metrics.serviceMonitor.labels` | Extra labels for the `ServiceMonitor` (for `Prometheus` CR selectors) | `{}` |
| `metrics.prometheusRule.enabled` | Create a Prometheus Operator `PrometheusRule` with built-in alerts | `false` |
| `metrics.prometheusRule.namespace` | Namespace for the `PrometheusRule` | release namespace |
| `metrics.prometheusRule.additionalLabels` | Extra labels on the `PrometheusRule` (for your Prometheus' `ruleSelector`) | `{}` |
| `metrics.prometheusRule.rules` | List of alerting rules — see below | see `values.yaml` |
| `metrics.prometheusRule.additionalGroups` | Extra raw rule groups (standard `PrometheusRule` `groups` schema), appended as-is | `[]` |
| `metrics.grafanaDashboard.enabled` | Create a `ConfigMap` containing the Grafana dashboard, for the Grafana sidecar | `false` |
| `metrics.grafanaDashboard.namespace` | Namespace for the dashboard `ConfigMap` | release namespace |
| `metrics.grafanaDashboard.labels` | Labels on the `ConfigMap`; must match the sidecar's label selector | `grafana_dashboard: "1"` |
| `metrics.grafanaDashboard.annotations` | Annotations on the `ConfigMap` (e.g. `grafana_folder: Lightpanda`) | `{}` |

#### Grafana dashboard

The chart bundles a Grafana dashboard built from the metrics exposed by
Lightpanda (see
[`src/Metrics.zig`](https://github.com/lightpanda-io/browser/blob/main/src/Metrics.zig)).
It is available as [`dashboards/lightpanda.json`](./dashboards/lightpanda.json)
(import it manually in Grafana), or can be provisioned automatically:

```yaml
metrics:
  serviceMonitor:
    enabled: true
  grafanaDashboard:
    enabled: true
    annotations:
      grafana_folder: Lightpanda   # optional
```

This renders a `ConfigMap` labelled `grafana_dashboard: "1"`, which is what the
Grafana dashboard sidecar (kube-prometheus-stack, the `grafana` chart, ...)
looks for by default. The dashboard has `Data source`, `Job` and `Instance`
variables, and covers server connections/commands, outbound HTTP
(rate, errors, latency, size, cache), V8 heap and JS errors, arena pool
memory, and robots.txt / CORS / adblock activity.

#### Built-in alerting rules

Enabling `metrics.prometheusRule.enabled` ships a `PrometheusRule` with alerts
based on Lightpanda's own metrics (see
[`src/Metrics.zig`](https://github.com/lightpanda-io/browser/blob/main/src/Metrics.zig)
upstream). `metrics.prometheusRule.rules` is a plain list, and the template
just `range`s over it — each item maps almost 1:1 to a native
[Prometheus alerting rule](https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/)
(`alert`/`expr`/`for`/`labels`/`annotations`), plus an `enabled` flag:

| Alert | Fires when |
| --- | --- |
| `LightpandaTargetDown` | Prometheus can't scrape the instance for 5m |
| `LightpandaJSHeapLimitHit` | Any page was killed for hitting the V8 heap limit within 15m |
| `LightpandaHighJSErrorRate` | Uncaught JS error rate exceeds 0.1/sec over 10m |
| `LightpandaHighHTTPErrorRate` | Ratio of failed outbound HTTP requests exceeds 10% over 10m |
| `LightpandaConnectionLimitReached` | New connections were deferred because `server.maxConnections` was hit within 5m |
| `LightpandaSessionTimeoutSpike` | Any WebDriver/CDP session timed out within 15m |
| `LightpandaInboxBacklogDrops` | Any websocket connection was dropped for backlog overload within 5m |

**Templating note:** `expr` is rendered through Helm's `tpl`, so it can
reference chart helpers, e.g. `{{ include "lightpanda.fullname" . }}` to
scope the `job` label to this release. `labels`/`annotations` are **not**
processed by Helm and are emitted as-is, so Prometheus' own alert templating
(`{{ $value }}`, `{{ $labels.xxx }}`) works there with zero escaping.

**Important — lists replace, they don't merge:** Helm replaces list values
wholesale on override, it doesn't merge them item-by-item like it does for
maps. So `--set metrics.prometheusRule.rules[0].enabled=false` or a values
file that only sets a couple of fields will **drop every other built-in
rule**, not just tweak one (see the example below, where only the two
listed rules survive). To tweak, disable, or remove individual rules while
keeping the rest, copy the full `rules` list from `values.yaml` into your
own values file and edit it there.

Example: disable the target-down alert and tighten the JS error threshold,
keeping every other built-in rule (copied from `values.yaml`, then edited):

```yaml
metrics:
  prometheusRule:
    enabled: true
    rules:
      - alert: LightpandaTargetDown
        enabled: false           # <- disabled, rest unchanged
        expr: 'up{job="{{ include "lightpanda.fullname" . }}"} == 0'
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: Lightpanda target down
          description: 'Prometheus has not been able to scrape this instance for over 5m (instance {{ $labels.instance }}).'
      - alert: LightpandaHighJSErrorRate
        enabled: true
        expr: 'rate(js_errors_total{job="{{ include "lightpanda.fullname" . }}"}[10m]) > 0.05'  # <- tightened from 0.1
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: High rate of uncaught JS errors
          description: '{{ $value | printf "%.2f" }} uncaught JS errors/sec ({{ $labels.kind }}) over the last 10m (threshold 0.05/sec).'
      # ... plus the other built-in rules, unchanged, copied from values.yaml
```

To add a brand new rule without touching the built-ins, append to the copied
list:

```yaml
      - alert: MyCustomExtraRule
        enabled: true
        expr: 'rate(http_redirects_total{job="{{ include "lightpanda.fullname" . }}"}[5m]) > 10'
        for: 5m
        labels:
          severity: info
        annotations:
          summary: Lots of redirects
          description: '{{ $value }} redirects/sec'
```

For alerts that don't fit the single-group model at all (e.g. a separate
rule group with its own name), use `additionalGroups`, appended as sibling
groups — this one is untouched by list-replacement concerns since it's
additive, not merged with `rules`:

```yaml
metrics:
  prometheusRule:
    enabled: true
    additionalGroups:
      - name: lightpanda-custom
        rules:
          - alert: MyCustomAlert
            expr: up{job="my-lightpanda"} == 0
            for: 2m
            labels:
              severity: page
            annotations:
              summary: Custom alert
```

## Ingress & WebSockets

CDP and WebDriver Bidi are both served over long-lived WebSocket
connections. Depending on your ingress controller, you may need to raise
proxy timeouts and/or explicitly enable WebSocket upgrades so the
connection isn't cut after a short idle period. This chart doesn't ship
default `ingress.annotations` because they are controller-specific — add
the ones matching your setup via `ingress.annotations`.

> **Note on `ingress-nginx`:** the community [`kubernetes/ingress-nginx`](https://github.com/kubernetes/ingress-nginx)
> project entered maintenance mode in 2025 with a planned retirement, in
> favor of the [Gateway API](https://gateway-api.sigs.k8s.io/). If you're
> starting a new deployment, consider a controller with a longer runway
> (Traefik, HAProxy, the Gateway API, or a cloud-managed ingress) instead of
> `ingress-nginx`.

**`ingress-nginx`** (community controller, in maintenance mode):
```yaml
ingress:
  className: nginx
  annotations:
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
```
WebSocket upgrades are auto-detected by `ingress-nginx` from the `Upgrade`
header, so no dedicated annotation is required for that part.

**Traefik**: WebSockets are supported natively, with no annotations
required — a plain `Ingress` with `className: traefik` is enough:
```yaml
ingress:
  className: traefik
  hosts:
    - host: lightpanda.example.com
      paths:
        - path: /
          pathType: Prefix
```
Unlike `ingress-nginx`, Traefik doesn't expose per-`Ingress` timeout
annotations — connection idle/read timeouts are set once, per entrypoint,
in Traefik's own static configuration (not in this chart), e.g.:
```yaml
# traefik static config (traefik.yml / Helm values of the Traefik chart itself)
entryPoints:
  websecure:
    address: ":443"
    transport:
      respondingTimeouts:
        readTimeout: 3600s
        idleTimeout: 3600s
```
If you need per-route control instead, use a Traefik `IngressRoute` CRD in
place of `ingress.enabled` (set `ingress.enabled: false` and create the
`IngressRoute` yourself, pointing at the `{{ include "lightpanda.fullname" . }}`
Service) combined with a `ServersTransport` resource for backend-side
timeouts.

**HAProxy Ingress**:
```yaml
ingress:
  className: haproxy
  annotations:
    haproxy-ingress.github.io/timeout-tunnel: "3600s"
```

**Gateway API** (recommended for new deployments): this chart only renders
a classic `networking.k8s.io/v1` `Ingress`. To use the Gateway API instead,
set `ingress.enabled: false` and create your own `Gateway`/`HTTPRoute`
resources selecting the Service created by this chart
(`{{ include "lightpanda.fullname" . }}`), which natively supports
WebSockets without special annotations.

## Examples

### Expose via Ingress

```yaml
ingress:
  enabled: true
  className: nginx
  hosts:
    - host: lightpanda.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - hosts:
        - lightpanda.example.com
      secretName: lightpanda-tls
```

See [Ingress & WebSockets](#ingress--websockets) below for the
controller-specific annotations you'll typically need alongside this.

### Route outbound traffic through an authenticated proxy

```yaml
config:
  proxy:
    enabled: true
    httpProxy: "http://proxy.internal:3128"
    existingSecret: lightpanda-proxy-token
    existingSecretKey: token
```

### Enable autoscaling

```yaml
autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
```

### Pin the image by digest

```yaml
image:
  digest: "sha256:REPLACE_WITH_DIGEST"
```

## Post-install notes

After installation, run:

```console
helm status my-lightpanda
```

to see connection instructions for your specific `service.type`/`ingress`
configuration (also shown automatically at the end of `helm install`/
`helm upgrade`), see [`templates/NOTES.txt`](./templates/NOTES.txt).

## Chart structure

```
charts/lightpanda/
├── Chart.yaml
├── values.yaml
├── values.schema.json
├── .helmignore
├── README.md
├── dashboards/
│   └── lightpanda.json
└── templates/
    ├── _helpers.tpl
    ├── deployment.yaml
    ├── service.yaml
    ├── serviceaccount.yaml
    ├── ingress.yaml
    ├── hpa.yaml
    ├── pdb.yaml
    ├── networkpolicy.yaml
    ├── servicemonitor.yaml
    ├── prometheusrule.yaml
    ├── grafana-dashboard.yaml
    └── NOTES.txt
```

## AI-assisted development

This chart and its documentation were created with the help of an AI
coding agent (Zed's agent, using Anthropic Claude models), under human
review and direction. See the [repository root README](../../README.md#ai-assisted-development)
for details.

## License

This chart is licensed under the [GNU General Public License v3.0](../../LICENSE)
(GPLv3). It is maintained independently and is not an official Lightpanda
project artifact — see [lightpanda-io/browser](https://github.com/lightpanda-io/browser)
for the upstream project's own license.
