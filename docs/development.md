# Developing the charts

Almost every template lives in `charts/base`. The other charts are thin wrappers that
include the named templates they need, so a change to `base` reaches all of them.

## Layout

```
charts/base/templates/_deployment.tpl     named templates, one file per resource
charts/app/templates/base.yaml            the includes this chart exposes
charts/app/values.yaml                    defaults, annotated for helm-docs
charts/app/values.schema.json             rejects unknown keys and wrong types
charts/app/tests/*_test.yaml              helm-unittest suites
charts/app/ci/*-values.yaml               realistic configs for CI install and render
```

## Making a change

```bash
# rebuild the vendored base after any version or Chart.yaml change
rm -f charts/*/Chart.lock && rm -rf charts/*/charts
for c in charts/*/; do helm dependency update "$c"; done

# the full suite - this is what CI runs
docker run --rm --user "$(id -u):$(id -g)" --env HOME=/tmp \
  -v "$PWD:/apps" helmunittest/helm-unittest:latest ./charts/*

# a single suite while iterating
docker run --rm --user "$(id -u):$(id -g)" --env HOME=/tmp \
  -v "$PWD:/apps" helmunittest/helm-unittest:latest \
  -f 'tests/deployment_test.yaml' charts/app

helm lint charts/*
```

A `base` change is not verified until the whole suite passes, not just the chart you
were looking at.

## Writing tests

Tests assert what the chart **should** do. When one fails, the chart is usually wrong —
but not always, so check the Kubernetes API reference or the Helm docs before deciding.
Never rewrite an assertion to match broken output.

Every chart renders all its resources from one `templates/base.yaml`, so assertions need
a `documentSelector`:

```yaml
- it: should set replicas
  set:
    replicas: 3
  asserts:
    - equal:
        path: spec.replicas
        value: 3
      documentSelector:
        path: kind
        value: Deployment
```

Select by `kind` when it is unique in the render, by `metadata.name` when it is not.
Note a Deployment and its Service share a name by default, which yields
`multiple indexes found` — give one a distinct name in that test.

Pin `chart.version` in the suite header, or every version bump breaks the assertions
that check the `helm.sh/chart` label.

## Adding a resource

1. Add `charts/base/templates/_<resource>.tpl`, following a sibling for conventions —
   list vs map, name suffixing, how labels and namespaces are applied.
2. Default any selector from `base.selectorLabels` and any service reference from
   `base.fullname`, so the resource stays aligned with the workload automatically.
3. Include it from the charts that should expose it, and only those.
4. Add the key to `values.yaml` with a `# --` description.
5. Add the key to `values.schema.json`, or `additionalProperties: false` rejects it.
6. Write the suite: defaults, each option, multiple instances.

## Adding a value

`values.schema.json` has `additionalProperties: false`, so an undeclared key is an
install-time error rather than a silently ignored setting. That is deliberate — it
catches typos like `podAnotations` — but it means new keys must be declared in both
`values.yaml` and the schema.

## CI

`.github/workflows/test.yml` runs on every pull request and is called by the release
workflow, so the same checks gate a merge and a publish:

- `helm-unittest` over every chart
- `helm template` of each `ci/*-values.yaml` through `kubeconform`
- `ct lint` and `ct install` on a kind cluster, for changed charts only
