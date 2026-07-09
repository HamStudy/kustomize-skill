# Validation Checklist

## Contents

- Render commands
- Structural checks
- Security checks
- GitOps checks
- Review questions

## Render Commands

Use the same renderer as deployment when possible.

Standalone:

```bash
kustomize version
kustomize build dev > /tmp/dev.yaml
kustomize build prod > /tmp/prod.yaml
```

Kubectl-embedded renderer:

```bash
kubectl version --client
kubectl kustomize prod > /tmp/prod.yaml
```

Server-side dry run, when cluster access is intended:

```bash
kustomize build prod | kubectl apply --server-side --dry-run=server -f -
```

Diff, when cluster access is intended:

```bash
kustomize build prod | kubectl diff -f -
```

## Structural Checks

Inspect rendered output for:

- every expected resource kind and name
- duplicate resources with the same group, kind, namespace, and name
- correct namespace placement
- generated ConfigMap and Secret names wired into workloads
- image tags or digests updated everywhere expected
- patches applying to the intended target only
- no unresolved placeholder values such as `REPLACE_ME` or `$(VAR)`
- selectors stable and matching Pod template labels
- component-owned local PodTemplate sources annotated as local config and absent from rendered output
- shared pod-template replacements applied to every intended workload in the chosen variant family
- CronJob pod-template patches written against `spec.jobTemplate.spec.template`, not `spec.template`
- leaf targets render exactly the intended workload variant, not every possible kind
- fields copied by `replacements` are not also expected to be overridden by later leaf patches
- generators intended for leaf `behavior: merge` or `replace` live in a base/variant resource chain, not only inside a component
- CronJobs with intended schedule, suspend state, and image
- Certificates, Ingresses, and Gateway resources agreeing on hostnames and TLS Secret names
- CRD fields transformed as intended

## Security Checks

Confirm:

- no plaintext real secrets in `kustomization.yaml`, env files, generated manifests, or patches
- no private key material committed unless encrypted by the repository's approved workflow
- base64 Secret data is treated as sensitive
- generated Secret hash behavior is intentional
- ExternalSecret, SealedSecret, or SOPS ownership is clear
- ServiceAccount annotations and RBAC differ only where intended

## GitOps Checks

For Argo CD:

- Verify the Application path points at the target directory.
- Check whether Argo's configured Kustomize version supports the features used.
- If using components from Argo Application spec, decide whether `ignoreMissingComponents` is intentional.
- If using Helm inflation through Kustomize, confirm repo-server build options include `--enable-helm` or use a config management plugin.
- Prefer `.spec.source.kustomize.namespace` over only `.spec.destination.namespace` when Kustomize should set namespace fields inside rendered manifests.

For Flux:

- Verify `.spec.path` points at the intended target directory.
- If using encrypted secrets, confirm `.spec.decryption.provider: sops` and key references.
- Check post-build substitution usage separately from Kustomize replacements.
- Confirm health checks and dependsOn relationships if controllers or CRDs must be applied before app resources.

## Review Questions

Ask these before handing off:

- Could this target kustomization be shorter without losing clarity?
- Is an optional feature copied into multiple targets instead of modeled as a component?
- Is a patch doing what `images`, `replicas`, `labels`, `namespace`, or generator behavior already does?
- Is repeated pod-template YAML copied from a component-owned local `PodTemplate` source instead of repeated in full resources?
- If a value must be overridden, is it owned by `images`, generator merge data, a variant wrapper, or a separate component instead of a patch that fights a replacement?
- If config should be overridden with generator behavior, is the original generator reachable through `resources` rather than only through `components`?
- Does a base resource depend on a resource generated only in a target kustomization?
- Is a CRD field relying on a built-in transformer that does not know the CRD schema?
- Are remote bases pinned?
- Will the exact production renderer support this syntax?
- Does the rendered output, not just the source YAML, match the user's intent?
