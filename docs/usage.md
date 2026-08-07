# How these charts work

Things that differ from a typical chart, and will otherwise cost you time.

## The templates are not in the chart you install

Every chart's `templates/` directory holds a single `base.yaml` that includes named
templates from the `base` library chart:

```
{{ include "base.deployment" . }}
{{ include "base.service" . }}
```

To read how a Deployment is rendered, open `charts/base/templates/_deployment.tpl`, not
`charts/app/templates/`.

A chart only includes what it declares, so setting `ingress` values on the `service`
chart does nothing — it has no Ingress template. That is deliberate: a narrow chart
cannot quietly create a workload.

## Values mean the same thing across charts

`image`, `environment`, `resources`, `volumes`, `securityPreset` and the rest behave
identically in `app`, `job` and `cronjob`, because all three render through the same
library. Learn them once.

## Resources are named after the release

A release called `my-service` produces a Deployment called `my-service` — not
`my-service-app`. `app.kubernetes.io/name` is the release name too, and the default
selector is `app: <release>` rather than the standard label pair.

That is deliberate for a generic chart: the release identifies the application, the
chart does not. It also means **two charts installed under the same release name in one
namespace will collide.**

- `fullnameOverride` renames the workload and everything derived from it
- `nameOverride` renames only the main container

## Settings that conflict

| Combination | What happens |
|---|---|
| `autoscaling` + `keda` both enabled | renders an HPA **and** a ScaledObject, which fight over replicas |
| `argo.rollouts.enabled: true` | the Deployment is replaced by a Rollout |
| `argo.rollouts.type: workloadRef` | Deployment stays at 0 replicas, the Rollout drives it |
| `statefulSet: true` | switches the workload kind; there is no separate StatefulSet chart |
| `podDisruptionBudget.maxUnavailable` | takes precedence over any `minAvailable` |

## Container order

`initContainers` and `extraContainers` accept either a list or a map. **A map is sorted
by name**, so declaration order is lost — use a list whenever order matters:

```yaml
initContainers:
  - name: wait-for-db
    command: ["sh", "-c", "until pg_isready; do sleep 1; done"]
  - name: migrate
    command: ["bin/migrate"]
```

Containers inherit the main image unless they set their own.

## Things that are wired up for you

- The Service targets the container ports you declare
- The Ingress backend points at the Service and its first port
- The autoscaler, disruption budget and network policy all select the workload's pods
- ConfigMap and Secret content is hashed into the pod template, so editing config rolls
  the pods. Set `rollOnConfigChange: false` to stop that

Override any of them and the rest follow, because they derive from the same helpers.

## Unknown values are an error

The schema rejects keys it does not recognise, so a typo fails the install by name
rather than being silently dropped:

```
Error: values don't meet the specifications of the schema(s)
- at '': additional properties 'podAnotations' not allowed
```

Multi-line strings in `configmaps[].valuesMultiLine` are rendered through `tpl`, so
`{{ .Release.Name }}` works inside them.

## When you need something the chart does not render

`rawYamlList` takes arbitrary manifests and adds the chart's common labels.
`rawTemplateList` does the same but renders through `tpl` first.
