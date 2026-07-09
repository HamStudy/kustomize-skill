---
name: kustomize-skill
description: Build, review, refactor, or explain Kubernetes manifests that use Kustomize. Use for designing DRY base/target/component layouts, creating minimal reusable deployment targets without a central overlays directory, using configMapGenerator and secretGenerator, writing patches and replacements, handling cert-manager Certificates, CronJobs, ConfigMaps, Secrets, CRDs, GitOps rendering, and validating multi-file Kustomize output.
---

# Kustomize Skill

Use this skill to create or improve Kustomize layouts that keep deployment target kustomizations small while still rendering complete Kubernetes applications.

Default posture: put complete common resources in `base/`, package optional features as `components/`, put deployment targets such as `dev/`, `staging/`, and `prod/` beside `base/`, and verify the rendered YAML with the same Kustomize version the deployment system uses. Do not introduce a central `overlays/` directory unless the existing repository already uses it.

## Quick Workflow

1. Identify the deployment matrix: apps, environments, clusters, optional features, and secrets/certificate ownership.
2. Choose a layout using `references/decision-guide.md`.
3. Put full Kubernetes resources in `base/` and reusable optional capabilities in `components/`.
4. Keep each target kustomization limited to `resources`, `components`, `namespace`, `labels`, `images`, `replicas`, generator behavior overrides, and tiny targeted patches.
5. Use generators for ConfigMaps and Secrets, replacements for field-to-field wiring, and the unified `patches` field for small deltas.
6. Render every target and inspect the output for duplicate resources, wrong namespaces, selector changes, unexpanded names, and leaked secrets.

## Table Of Contents

Read only the references needed for the current task:

- `references/decision-guide.md`: choose bases, deployment target kustomizations, components, patches, replacements, generators, Helm, and CRD handling.
- `references/layout-patterns.md`: directory structures for sibling target directories, app/env/cluster matrices, and component libraries.
- `references/generators-and-secrets.md`: ConfigMap/Secret generators, hash suffixes, merge/replace behavior, immutable generated resources, and Git-safe secret patterns.
- `references/patches-replacements-crds.md`: unified patches, replacements instead of vars, CRD transformer gaps, OpenAPI, and name/image reference wiring.
- `references/resource-recipes.md`: practical recipes for Deployments, Services, CronJobs, cert-manager Certificates, Ingress, NetworkPolicy, RBAC, and ExternalSecret-style resources.
- `references/full-example.md`: a compact multi-file app that renders Deployments, Services, ConfigMaps, Secrets, CronJobs, Certificates, Ingress, and environment targets with little target YAML.
- `references/validation-checklist.md`: commands and review checks before handing off a Kustomize change.
- `references/source-notes.md`: current source notes and links that informed this skill.

## Current Defaults

Prefer these patterns unless the repository already has a stronger local convention:

- Use `apiVersion: kustomize.config.k8s.io/v1beta1` and `kind: Kustomization`.
- Use `kind: Component` with `apiVersion: kustomize.config.k8s.io/v1alpha1` for reusable optional feature bundles.
- Use the unified `patches` field instead of adding new `patchesStrategicMerge` or `patchesJson6902` entries.
- Use `labels` instead of deprecated `commonLabels`; set `includeSelectors: true` only for newly created resources where selector labels are intentional and stable.
- Use `replacements` instead of `vars`.
- Let generated ConfigMap and Secret name hashes stay enabled unless a controller or external consumer truly requires a stable name.
- Pin remote bases or vendor them. Do not point production targets at a moving branch.
- Treat Kustomize as composition and transformation, not as a general template language.

## Research Resources

Use https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/ as the primary Kustomize kustomization reference for exact field names, current syntax, deprecations, and behavior of built-in Kustomize primitives.

When the user asks a Kustomize behavior question and local references are not enough, use DeepWiki MCP for `kubernetes-sigs/kustomize` before guessing. Start with `read_wiki_structure` or ask a focused question with `ask_question` against repo `kubernetes-sigs/kustomize`. The human-facing DeepWiki URL is https://deepwiki.com/kubernetes-sigs/kustomize.

## Agent Rules

- Preserve existing repository conventions when editing an established Kustomize tree.
- Avoid adding a new target layer when a component or generator override would express the change more simply.
- Avoid copying full base manifests into target directories. Patch or compose instead.
- Avoid committing plaintext secret material. Base64 in a Kubernetes Secret is not encryption.
- Check the renderer version used by GitOps tooling, CI, and local commands before relying on newer features.
- Prefer `kustomize build <target-dir>` for standalone Kustomize and `kubectl kustomize <target-dir>` only when matching kubectl's embedded Kustomize behavior is intentional.
