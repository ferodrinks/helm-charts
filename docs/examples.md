# Examples

Values files for common setups.

## Web service

Deployment, Service, Ingress.

```yaml
replicas: 2

image:
  repository: ghcr.io/ferodrinks/my-service
  tag: "1.4.2"

ports:
  - containerPort: 8080
    name: http

service:
  ports:
    - port: 80
      targetPort: 8080
      name: http

ingress:
  enabled: true
  class: traefik
  hosts:
    - host: my-service.example.com

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    memory: 512Mi

livenessProbe:
  httpGet:
    path: /healthz
    port: http
readinessProbe:
  httpGet:
    path: /ready
    port: http

securityPreset: restricted
```

## Autoscaling

Adds a HorizontalPodAutoscaler and a PodDisruptionBudget.

```yaml
autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  cpu:
    - averageUtilization: 70

podDisruptionBudget:
  enabled: true

podAntiAffinity:
  enabled: true
```

`replicas` is dropped from the Deployment once autoscaling is on, so the HPA owns it.
`podDisruptionBudget` with no values derives `minAvailable` from half of `minReplicas`.

Do not enable `keda` as well — both manage replicas and you get two competing scalers.

## Configuration and secrets

```yaml
environment:
  variables:
    NODE_ENV: production
    LOG_LEVEL: info
  secretVariables:
    DATABASE_URL:
      name: my-service-secret
      key: DATABASE_URL

configmaps:
  - name: my-service-config
    values:
      FEATURE_FLAGS: "a,b,c"
    valuesMultiLine:
      settings.json: |-
        {
          "release": "{{ .Release.Name }}"
        }
```

`valuesMultiLine` is rendered through `tpl`, so template expressions inside the file
body are evaluated rather than written out literally.

Editing either rolls the pods, because their content is hashed into the pod template.
Set `rollOnConfigChange: false` to opt out.

For real secrets use `externalSecrets` rather than `secrets.decoded` — the latter puts
plaintext in git, in CI logs and in the release secret.

```yaml
externalSecrets:
  - name: my-service-secret
    secretStoreRef:
      name: aws-store
    data:
      - secretKey: DATABASE_URL
        remoteRef:
          key: prod/my-service/database-url
```

## Database

StatefulSet instead of a Deployment, plus its Service.

```yaml
statefulSet: true
replicas: 3

image:
  repository: ghcr.io/ferodrinks/my-database
  tag: "2.1.0"

ports:
  - containerPort: 5432
    name: postgres

service:
  ports:
    - port: 5432

volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ReadWriteOnce]
      resources:
        requests:
          storage: 20Gi

persistentVolumeClaimRetentionPolicy:
  whenDeleted: Retain
  whenScaled: Retain
```

`serviceName` defaults to the release name.

## Scheduled job

Chart: `cronjob`. Same values without `schedule` in chart `job` for a one-off task.

```yaml
schedule: "0 2 * * *"
concurrencyPolicy: Forbid
successfulJobsHistoryLimit: 3

image:
  repository: ghcr.io/ferodrinks/my-service
  tag: "1.4.2"

command: ["sh", "-c"]
args: ["bin/nightly-report"]

resources:
  requests:
    cpu: 500m
    memory: 1Gi
```

## Migration before the app starts

Runs in the order written. Both inherit the main container's image.

```yaml
initContainers:
  - name: wait-for-db
    command: ["sh", "-c", "until pg_isready; do sleep 1; done"]
  - name: migrate
    command: ["bin/migrate"]
```

## Sidecar

```yaml
extraContainers:
  - name: log-shipper
    image:
      repository: ghcr.io/ferodrinks/log-shipper
      tag: "0.9.0"
    volumeMounts:
      - name: logs
        mountPath: /var/log/app

volumes:
  - name: logs
    emptyDir: {}
```

The main container stays first, so `kubectl logs` without `-c` still picks it.

## TLS from cert-manager

```yaml
certificates:
  - dnsNames:
      - my-service.example.com
    issuerRef:
      name: letsencrypt-prod

ingress:
  enabled: true
  class: traefik
  hosts:
    - host: my-service.example.com
  tls:
    - secretName: my-service-tls
      hosts:
        - my-service.example.com
```

The certificate writes `<release>-tls` unless `secretName` says otherwise.

## Locking down network access

```yaml
networkPolicies:
  - ingress:
      - from:
          - podSelector:
              matchLabels:
                app.kubernetes.io/name: api-gateway
    egress:
      - to:
          - namespaceSelector:
              matchLabels:
                kubernetes.io/metadata.name: kube-system
```

`podSelector` defaults to this workload's own pods.

For a namespace-wide default-deny, set `podSelector: {}` and spell out `policyTypes` —
there are no rules for it to be derived from:

```yaml
networkPolicies:
  - name: default-deny
    podSelector: {}
    policyTypes: [Ingress, Egress]
```

That one belongs in the `default-base` chart rather than on each service.

## Canary with Argo Rollouts

```yaml
argo:
  rollouts:
    enabled: true
    strategy:
      canary:
        steps:
          - setWeight: 20
          - pause: {duration: 5m}
          - setWeight: 50
          - pause: {duration: 5m}
```

The Deployment is replaced by a Rollout. Use `type: workloadRef` instead to keep the
Deployment at zero replicas and have the Rollout reference it.

## Anything the chart does not cover

```yaml
rawYamlList:
  - apiVersion: v1
    kind: LimitRange
    metadata:
      name: limits
    spec:
      limits:
        - type: Container
          default:
            cpu: 100m
```

`rawTemplateList` does the same but renders through `tpl` first, so
`{{ .Release.Name }}` works.
