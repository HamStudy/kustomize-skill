# Layout Patterns

## Contents

- Preferred app layout
- When a repository already uses overlays
- Component library layout
- Environment and cluster matrix
- Platform and application split
- Remote base policy
- Naming conventions
- Layering limit

## Preferred App Layout

Use sibling deployment target directories beside `base/`:

```text
app/
  base/
    kustomization.yaml
    deployment.yaml
    service.yaml
    serviceaccount.yaml
    networkpolicy.yaml
    cronjob.yaml
    config/
      app.env
      logging.properties
  components/
    ingress-cert/
      kustomization.yaml
      ingress.yaml
      certificate.yaml
    external-secrets/
      kustomization.yaml
      externalsecret.yaml
      deployment.patch.yaml
    maintenance-cron/
      kustomization.yaml
      cronjob.yaml
  dev/
    kustomization.yaml
    config.env
    ingress-cert.patch.yaml
  prod/
    kustomization.yaml
    config.env
    ingress-cert.patch.yaml
    deployment-resources.patch.yaml
```

In this shape, `base/` is the directory for things that are not deployment-target-specific. `dev/`, `prod/`, and similar directories are still Kustomize overlays conceptually, but the repo does not have an `overlays/` bucket.

The target directories should be recognizable at a glance. Most YAML volume belongs in `base/` and `components/`.

## When A Repository Already Uses Overlays

If the repository already has `overlays/`, do not churn it just for style. Work with the local convention unless the user explicitly asks to restructure.

When creating a new layout or doing a meaningful restructure, prefer:

```text
app/
  base/
  components/
  dev/
  staging/
  prod/
```

over:

```text
app/
  base/
  components/
  overlays/
    dev/
    staging/
    prod/
```

## Component Library Layout

Use this when multiple apps share optional capabilities:

```text
platform-kustomize/
  components/
    cert-manager-http/
    external-secrets-vault/
    prometheus-servicemonitor/
    cloud-sql-proxy/
    redis-sidecar/
    baseline-networkpolicy/
  bases/
    webapp/
    worker/
    cron-worker/
```

Each component should have one job. A component may include:

- new resources
- ConfigMap or Secret generators
- patches against resources expected from the base
- replacements to wire names or values
- transformer configuration for CRD fields

Do not make a component that silently assumes five unrelated features. Compose several small components from the target directory instead.

## Environment And Cluster Matrix

For many clusters, avoid creating a unique full target for every cluster if most values are metadata. Split environment behavior from cluster behavior:

```text
apps/payments/
  base/
  components/
  dev/
  staging/
  prod/
clusters/
  us-east-1/prod/payments/
    kustomization.yaml
  eu-west-1/prod/payments/
    kustomization.yaml
```

The cluster target can reference the app target:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../../../apps/payments/prod
labels:
  - pairs:
      topology.kubernetes.io/region: us-east-1
    includeTemplates: true
patches:
  - path: cluster-identity.patch.yaml
```

Keep this extra layer only if cluster-specific concerns are real: region identity, storage class, service account annotations, node selectors, DNS zones, or issuer names.

## Platform And Application Split

Separate platform controllers from app instances:

```text
platform/
  cert-manager/
  external-secrets/
  ingress-nginx/
  monitoring/
apps/
  checkout/
  accounts/
  worker/
```

Applications may include CRD instances such as `Certificate`, `ExternalSecret`, or `ServiceMonitor`, but controller installation normally belongs in platform manifests.

## Remote Base Policy

Remote bases are useful for upstream packages and shared internal bases, but production usage should be pinned:

```yaml
resources:
  - https://github.com/acme/platform-kustomize//bases/webapp?ref=v1.4.2
```

Prefer release tags or immutable SHAs. Avoid `main`, `master`, or unqualified remote URLs in production targets. If reliability, reviewability, or private auth is a problem, vendor the base into the repo and update it intentionally.

## Naming Conventions

Use names that reveal intent:

- `base/`, not `common/` plus another `base/`
- `components/ingress-cert`, not `components/prod`
- `prod/`, not `overlays/prod`, for new app layouts
- `clusters/us-east-1/prod/payments/` for real cluster-specific targets
- patch files named after intent, such as `deployment-resources.patch.yaml`, `certificate-dns.patch.yaml`, or `cron-schedule.patch.yaml`

Prefer stable resource names in the base. Use `namePrefix` and `nameSuffix` when a whole rendered set must coexist in the same namespace, not as the default way to express environments.

## Layering Limit

One app base plus one target is easy. A component layer is still easy. A cluster target can be justified. Beyond that, the tree becomes harder to reason about than duplicated YAML.

When the tree grows, collapse layers by:

- promoting universal resources into the base
- extracting optional behavior into components
- replacing long target patches with generator merge files
- moving deployment orchestration differences to the GitOps layer
