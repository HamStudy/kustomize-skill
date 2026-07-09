# Generators And Secrets

## Contents

- ConfigMap generator patterns
- Secret generator patterns
- Name suffix hashes
- Merge and replace behavior
- Immutable generated resources
- Git-safe secret handling
- Common mistakes

## ConfigMap Generator Patterns

Prefer generators for application configuration:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
configMapGenerator:
  - name: app-config
    envs:
      - config/app.env
  - name: app-files
    files:
      - config/logging.properties
      - config/application.yaml
```

Reference generated ConfigMaps by the logical generator name:

```yaml
volumes:
  - name: app-config
    configMap:
      name: app-config
```

Kustomize updates supported references to the hashed name during build.

Use file key aliases when the key should not match the source filename:

```yaml
configMapGenerator:
  - name: app-files
    files:
      - application.yaml=config/base-application.yaml
```

## Secret Generator Patterns

Use `secretGenerator` for generated Secret manifests from local inputs:

```yaml
secretGenerator:
  - name: app-bootstrap
    envs:
      - secrets/bootstrap.env
    type: Opaque
  - name: app-tls
    files:
      - tls.crt=certs/tls.crt
      - tls.key=certs/tls.key
    type: kubernetes.io/tls
```

Use the generated logical name in workloads:

```yaml
envFrom:
  - secretRef:
      name: app-bootstrap
```

Do not commit `secrets/bootstrap.env`, private keys, or generated Secret YAML unless the contents are encrypted by the repository's accepted mechanism.

## Name Suffix Hashes

By default, generated ConfigMaps and Secrets get a hash suffix based on content. Keep that behavior on for workload-mounted config and secrets because it lets content changes produce new names and update workload references.

Disable suffix hashes only for stable-name integrations:

```yaml
generatorOptions:
  disableNameSuffixHash: true
```

or per generated resource:

```yaml
secretGenerator:
  - name: webhook-token
    literals:
      - token=replace-me
    options:
      disableNameSuffixHash: true
```

Stable-name cases include:

- a controller expects a fixed Secret name
- a non-Kustomize consumer references the resource
- the object is not mounted into a workload and rollout-by-name is not desired

Avoid global `disableNameSuffixHash: true` as a convenience. It removes one of Kustomize's strongest safety features.

## Merge And Replace Behavior

Use generator behavior in target kustomizations to modify generated config from a base:

```yaml
resources:
  - ../base
configMapGenerator:
  - name: app-config
    behavior: merge
    envs:
      - config.env
```

Behavior choices:

- `create`: default; error if the generated resource already exists.
- `merge`: add or update keys on an existing generated resource.
- `replace`: replace the existing generated resource.

When merging or replacing, the generator `name` and `namespace` must match the target generated resource as it exists before namespace transformation. If the base generator has no namespace, omit namespace in the target generator even if the target sets `namespace`.

For hash propagation, define the generator in the base when workloads reference it. Target generator entries then change the generated content and Kustomize computes the final suffix.

## Immutable Generated Resources

For clusters that enforce immutable ConfigMaps/Secrets or for applications that benefit from immutable config, use:

```yaml
generatorOptions:
  immutable: true
```

Combine immutable generated resources with hash suffixes. A content change produces a new generated object instead of mutating the old one.

## Git-Safe Secret Handling

Base64 is encoding, not encryption. A `Secret` manifest with base64 data is still plaintext for repository security purposes.

Choose one secret ownership model:

- Runtime controller model: use External Secrets Operator, Secrets Store CSI Driver, Vault operator, or another controller. Kustomize manages the CRD instance, not the secret value.
- Encrypted Git model: commit SOPS-encrypted Secret or values files, and let Flux, Argo CD plugins, or CI decrypt before apply.
- Local-only model: keep secret generator input files out of Git and inject them in CI or local release automation.
- Certificate controller model: use cert-manager `Certificate`; the TLS Secret is created by cert-manager and should usually have a stable name.

Kustomize alone is not a secret manager. It can render Secret manifests, but it does not protect secret material.

## Common Mistakes

Avoid:

- committing `.env` files that contain real passwords
- putting secret literals directly in `kustomization.yaml`
- disabling hash suffixes because the final names look cleaner
- generating a Secret only in a target kustomization when a base workload references it and expecting name hash propagation to work the same way
- mixing a generated TLS Secret and a cert-manager-managed TLS Secret with the same name
- relying on Kustomize to update CRD references to generated Secrets unless a transformer configuration or replacement handles that field
