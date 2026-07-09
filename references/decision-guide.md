# Decision Guide

## Contents

- Core model
- What belongs where
- Choosing the right Kustomize primitive
- Minimal target rule
- Version and renderer choices
- Anti-patterns

## Core Model

Think in three directory roles:

- `base/`: a complete, boring application shape that can render by itself.
- `components/`: optional or reusable feature bundles that may contain resources, generators, patches, and replacements.
- target directories such as `dev/`, `staging/`, `prod/`, or `clusters/<cluster>/<app>/`: short manifests of choices for deployment targets.

The base should contain the common Deployment, Service, ServiceAccount, RBAC, ConfigMap generator, stable CronJob definitions, NetworkPolicy, and any CRD instances that every target needs. The base should not know whether it is dev, staging, or prod unless the repository has only one environment.

Components should model feature switches or cross-cutting bundles:

- ingress and certificate exposure
- external database wiring
- observability sidecars or ServiceMonitor resources
- optional CronJobs
- cloud-specific identity annotations
- ExternalSecret or SealedSecret integration
- PodDisruptionBudget, HPA, or NetworkPolicy bundles if not universal

Kustomize may call the target kustomization an overlay, but do not default to a central `overlays/` directory. Prefer sibling target directories beside `base/` unless an existing repository already uses another convention.

Targets should choose and tune, not redefine. A healthy target `prod/kustomization.yaml` often has only:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../base
components:
  - ../components/ingress-cert
  - ../components/maintenance-cron
namespace: payments-prod
labels:
  - pairs:
      app.kubernetes.io/part-of: payments
      environment: prod
    includeTemplates: true
images:
  - name: ghcr.io/acme/payments-api
    newTag: "2026-07-09.1"
replicas:
  - name: api
    count: 4
configMapGenerator:
  - name: app-config
    behavior: merge
    envs:
      - config.env
patches:
  - path: certificate.patch.yaml
  - path: deployment-resources.patch.yaml
```

## What Belongs Where

Put in `base/`:

- full resources shared by nearly every target
- selector labels and stable object names
- generated ConfigMaps and Secrets that workloads reference by logical name
- references to generated ConfigMaps/Secrets using the unsuffixed generator name
- default resource requests, probes, ports, service names, and volumes

Put in `components/`:

- optional features that need several changes together
- resources plus patches that should be reused by multiple targets
- feature-specific generators, such as a config fragment and its Deployment volume mount
- controller-specific integrations such as cert-manager, External Secrets Operator, Prometheus Operator, or cloud identity, when not universal

Put in target directories such as `dev/`, `staging/`, `prod/`, or `clusters/<cluster>/<app>/`:

- namespace, labels, annotations, replica counts, and image tags
- small environment config overrides using generator `behavior: merge` or `replace`
- target-specific values such as hostname, issuer name, schedule, retention, resource limits, or service account annotations
- tiny patches that are genuinely unique to that target

## Choosing The Right Kustomize Primitive

Use `resources` when adding complete YAML documents or another Kustomization directory.

Use `components` when the same optional capability is selected by more than one target, especially when the capability needs both resources and patches.

Use `configMapGenerator` for non-secret config from files, env files, or literals. Prefer a generator in the base when a workload references the ConfigMap, then merge or replace values from targets.

Use `secretGenerator` for generated Secret manifests from local files, env files, or literals. Do not confuse this with secret storage safety; generator inputs are still secret material.

Use `generatorOptions` to label or annotate all generated ConfigMaps/Secrets, mark them immutable, or deliberately disable name suffix hashes.

Use `images` for image name, tag, or digest changes. Do not patch container image fields unless the image field lives in a CRD Kustomize does not know how to transform.

Use `replicas` for simple replica count changes.

Use `labels` for cross-cutting labels. Use `includeTemplates: true` when Pods should get the labels. Use `includeSelectors: true` only when creating new resources or when selector mutation is explicitly intended.

Use `patches` for small structural changes. Prefer file patches over long inline patches when the patch is more than a few lines.

Use `replacements` when one rendered field must be copied into one or more target fields. This is the replacement for `vars` and is useful for generated names, CRD fields, and command/env fields.

Use `configurations`, `crds`, or `openapi` when custom resources need name reference, image reference, or strategic merge semantics that built-in transformers do not know.

Use Helm inflation sparingly. If Helm is the source package format, render it deliberately and pin versions. Do not hide broad Helm behavior inside Kustomize unless the deployment system supports the required `--enable-helm` behavior.

## Minimal Target Rule

When a target starts copying whole manifests, stop and ask:

- Can the common shape move to `base/`?
- Is this optional capability a component?
- Can a generated ConfigMap/Secret be merged instead of replaced as YAML?
- Can `images`, `replicas`, `labels`, or `namespace` express this without a patch?
- Can a replacement wire this value instead of duplicating names?
- Is the difference real environment behavior, or just a parameter that belongs in a small patch?

The ideal target kustomization is a readable bill of materials for a deployment target.

## Version And Renderer Choices

Record which renderer is authoritative:

- standalone `kustomize build`
- `kubectl kustomize`
- Argo CD repo-server
- Flux kustomize-controller
- another CI or GitOps renderer

Pin behavior in CI. Newer Kustomize versions add useful behavior, but GitOps controllers often bundle their own versions or allow version overrides. Avoid relying on a feature until the real renderer supports it.

As of July 9, 2026, upstream standalone Kustomize releases show `kustomize/v5.8.1` as latest. Avoid `v5.8.0` specifically for production baselines because its own release notes flagged a namespace propagation regression that was fixed in `v5.8.1`.

## Anti-Patterns

Avoid these unless a local constraint makes them unavoidable:

- a central `overlays/` directory introduced by habit rather than by repository convention
- target directories that repeat full Deployments, CronJobs, Services, or Certificates
- many environment targets that differ only by a single optional feature
- unpinned remote bases
- global `disableNameSuffixHash: true` used for convenience
- selector label changes on already-applied live resources
- adding new `vars`
- treating base64 Secret YAML as secure
- mixing Helm, Kustomize, and ad hoc templating without a clear owner for each transformation
- deep inheritance chains such as `base -> shared -> region -> env -> cluster -> app`
- patches that depend on fragile list indexes when a strategic merge by name or a replacement fieldPath would be clearer
