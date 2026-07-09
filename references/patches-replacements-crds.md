# Patches, Replacements, And CRDs

## Contents

- Patches
- Choosing patch styles
- Replacements
- Migrating away from vars
- CRD handling
- Custom transformer configuration
- Patch safety checklist

## Patches

Use the unified `patches` field for new work:

```yaml
patches:
  - path: deployment-resources.patch.yaml
    target:
      group: apps
      version: v1
      kind: Deployment
      name: api
  - patch: |-
      - op: replace
        path: /spec/schedule
        value: "15 */6 * * *"
    target:
      group: batch
      version: v1
      kind: CronJob
      name: cleanup
```

`patches` can use strategic merge or JSON6902 content. It can target by group, version, kind, name, namespace, label selector, or annotation selector.

Prefer a named patch file when the patch is more than a few lines. Inline patches are fine for one or two simple operations.

## Choosing Patch Styles

Strategic merge is readable for built-in Kubernetes resources:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  template:
    spec:
      containers:
        - name: api
          resources:
            requests:
              cpu: 250m
              memory: 256Mi
```

JSON6902 is precise and works the same for built-in resources and custom resources:

```yaml
- op: replace
  path: /spec/dnsNames/0
  value: api.prod.example.com
```

Use JSON6902 when:

- patching CRDs without OpenAPI merge metadata
- removing fields
- adding to exact paths
- strategic merge would replace a list unexpectedly

Avoid fragile numeric list indexes when an alternative exists. For containers, strategic merge by `name` is usually more stable than replacing `/containers/0`.

## Replacements

Use `replacements` when a rendered value should be copied from one resource field into another:

```yaml
replacements:
  - source:
      kind: Service
      name: api
      fieldPath: metadata.name
    targets:
      - select:
          kind: CronJob
          name: smoke-test
        fieldPaths:
          - spec.jobTemplate.spec.template.spec.containers.[name=check].env.[name=API_SERVICE].value
```

Use replacements for:

- putting a generated name into a field Kustomize does not transform automatically
- filling command args or env values from object names
- wiring values from a ConfigMap into CRD fields
- avoiding placeholder drift across targets

Prefer `sourceValue` for a literal that must be copied to several places:

```yaml
replacements:
  - sourceValue: api.prod.example.com
    targets:
      - select:
          kind: Certificate
          name: api-cert
        fieldPaths:
          - spec.dnsNames.0
          - spec.commonName
```

## Migrating Away From Vars

Do not add new `vars`. They are deprecated in Kustomize v5 and are not intended for the future v1 Kustomization API.

Migration pattern:

1. Replace `$(NAME)` placeholders in resources with explicit placeholder values.
2. Convert each var source object to a replacement `source`.
3. Point replacement targets to the exact fields that should receive the value.
4. Render before and after to verify output.

Use `kustomize edit fix --vars` only in a clean git worktree, then inspect the diff carefully. The automated rewrite may touch many files and may not preserve exact build output.

## CRD Handling

Kustomize knows built-in Kubernetes resource relationships better than arbitrary CRDs. For CRDs, be explicit.

Common cases:

- For simple CRD patches, prefer JSON6902 unless strategic merge behavior is backed by OpenAPI.
- For custom image fields, add transformer configuration so `images` can update the CRD.
- For custom object references, add name reference configuration or use replacements.
- For CRD strategic merge list behavior, provide OpenAPI schema data with merge keys.

Cert-manager, External Secrets Operator, Prometheus Operator, Argo Rollouts, and Gateway API resources are all CRDs. Do not assume generated names, prefixes, labels, or images will propagate into their nested fields without checking rendered output.

## Custom Transformer Configuration

When a CRD has an image field:

```yaml
configurations:
  - kustomizeconfig/images.yaml
```

```yaml
# kustomizeconfig/images.yaml
images:
  - kind: MyWorkload
    group: example.com
    path: spec/template/spec/container/image
```

When a CRD references a Secret or ConfigMap by name:

```yaml
configurations:
  - kustomizeconfig/name-reference.yaml
```

```yaml
# kustomizeconfig/name-reference.yaml
nameReference:
  - kind: Secret
    fieldSpecs:
      - kind: MyWorkload
        group: example.com
        path: spec/secretName
```

Use `crds:` or `openapi:` when the repository already has CRD schema files and needs Kustomize to understand merge keys or object references from schema metadata.

## Patch Safety Checklist

Before finishing:

- Render every affected target.
- Confirm patches match exactly the intended resources.
- Confirm generated ConfigMap and Secret references point to rendered names.
- Confirm selector labels did not change on live resources unless intentionally recreating them.
- Confirm CRD fields changed as intended.
- Confirm no plaintext secrets were introduced.
- Prefer a smaller Kustomize primitive if a patch is only changing image, replicas, namespace, labels, or generated config.
