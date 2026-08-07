# Helm Charts

Generic charts for Kubernetes workloads. One values file per service instead of a chart
per service — `app` covers Deployments, StatefulSets and everything that surrounds them.

## Start here

```bash
helm repo add baseCharts https://ferodrinks.github.io/helm-charts
```

`values.yaml`:

```yaml
image:
  repository: nginx
  tag: "1.27"
```

```bash
helm install my-app baseCharts/app -f values.yaml
```

Those three lines render a Deployment and a Service, named after the release, with the
standard Kubernetes labels applied. `helm template` first if you want to see the
manifests before anything is installed.

## Growing from there

Everything is off until you set it, so features go in one block at a time.

**Expose a port** — the Service picks it up without further configuration.

```yaml
ports:
  - containerPort: 8080
    name: http
```

**Route traffic to it.**

```yaml
ingress:
  enabled: true
  class: traefik
  hosts:
    - host: my-app.example.com
```

**Configuration and secrets**, side by side.

```yaml
environment:
  variables:
    LOG_LEVEL: info
  secretVariables:
    DATABASE_URL:
      name: my-app-secret
      key: DATABASE_URL
```

**Scaling and availability.**

```yaml
autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  cpu:
    - averageUtilization: 70

podDisruptionBudget:
  enabled: true
```

**The restricted Pod Security Standard** — non-root, no privilege escalation, all
capabilities dropped, seccomp `RuntimeDefault`.

```yaml
securityPreset: restricted
```

Still one file. It now renders a Deployment, Service, Ingress, HorizontalPodAutoscaler
and PodDisruptionBudget whose selectors, backends and scale targets all match each
other — the chart derives them from the same source rather than making you repeat them.

## Next

- [Examples](docs/examples.md) — complete values files: web service, scheduled job,
  database with storage, sidecars, canary deploys, cert-manager TLS
- [Usage](docs/usage.md) — helmfile, ArgoCD, choosing a chart, pinning versions
- [Development](docs/development.md) — changing the charts themselves

## Charts

Most services want `app`. The rest cover things that are not a long-running service.

| Chart | Use it for |
|---|---|
| [`app`](charts/app) | Anything long-running: an API, a worker, a database |
| [`job`](charts/job) | A task that runs once |
| [`cronjob`](charts/cronjob) | A task on a schedule |
| [`service`](charts/service) | A Service on its own |
| [`ingress`](charts/ingress) | An Ingress on its own |
| [`rbac`](charts/rbac) | Roles, bindings and service accounts |
| [`prometheus-rules`](charts/prometheus-rules) | Alerting rules |
| [`default-base`](charts/default-base) | Shared namespace setup: TLS secrets, storage classes |
| [`argo-applications`](charts/argo-applications) | ArgoCD Applications and Projects |
| [`argo-workflows`](charts/argo-workflows) | Argo Workflows and templates |
| [`base`](charts/base) | The shared templates every chart above uses. Not installable |

## Everything `app` can render

Deployment · StatefulSet · Service · Ingress · HorizontalPodAutoscaler · KEDA
ScaledObject · PodDisruptionBudget · VerticalPodAutoscaler · ConfigMap · Secret ·
ExternalSecret · NetworkPolicy · ServiceMonitor · Certificate · Argo Rollout ·
AnalysisTemplate · PriorityClass · PVC · RBAC · Traefik IngressRoute and Middleware ·
Istio VirtualService, DestinationRule, Gateway and policies · AWS TargetGroupBinding,
SecurityGroupPolicy and IngressClassParams · anything else via `rawYamlList`

`helm show values baseCharts/app` lists every option with a description.

## Base library

Each chart depends on `base` and includes the templates it needs:

```yaml
dependencies:
  - name: base
    version: ~0.18.0
    repository: file://../base
```

```
{{ include "base.deployment" . }}
{{ include "base.service" . }}
{{ include "base.ingress" . }}
```
