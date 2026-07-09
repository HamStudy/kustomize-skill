# Source Notes

## Contents

- Research date
- Primary sources
- DeepWiki MCP resource
- Skills.sh references considered
- Current practice notes
- Version notes

## Research Date

These notes were compiled on July 9, 2026.

## Primary Sources

Use current docs when exact behavior matters:

- Kustomize kustomization reference: https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/
- Kustomize upstream repository and releases: https://github.com/kubernetes-sigs/kustomize
- Kustomize components example: https://github.com/kubernetes-sigs/kustomize/blob/master/examples/components.md
- Kubernetes Kustomize task guide: https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/
- Argo CD Kustomize guide: https://argo-cd.readthedocs.io/en/stable/user-guide/kustomize/
- Flux Kustomization guide: https://fluxcd.io/flux/components/kustomize/kustomizations/

## DeepWiki MCP Resource

For behavior questions that are not fully answered by the local references, use the DeepWiki MCP against repo `kubernetes-sigs/kustomize` before guessing. Start with `read_wiki_structure`, then use `ask_question` for a focused question or `read_wiki_contents` for a listed page.

Human-facing DeepWiki URL: https://deepwiki.com/kubernetes-sigs/kustomize

The DeepWiki structure reviewed for this skill included pages for core concepts, architecture, resource handling, target processing, built-in transformers, built-in generators, plugin systems, Resource API, OpenAPI integration, release process, and usage examples.

## Skills.sh References Considered

The user suggested these skills as input material:

- https://www.skills.sh/thebushidocollective/han/kustomize-generators
- https://www.skills.sh/thebushidocollective/han/kustomize-basics
- https://www.skills.sh/thebushidocollective/han/kustomize-overlays
- https://www.skills.sh/bagelhole/devops-security-agent-skills/kustomize

Useful patterns retained: generator inputs from literals/files/env files, generator `behavior: merge` and `behavior: replace`, Secret type examples, render-before-apply workflow, component reuse, remote-base caution, and security review around Secrets.

Patterns deliberately adjusted: those references commonly use a central `overlays/` directory and some examples use older fields such as `commonLabels`, `bases`, or split patch fields. This skill prefers sibling deployment target directories beside `base/`, `labels`, `resources`, and the unified `patches` field for new work.

## Current Practice Notes

The Kustomize kustomization reference describes a kustomization as resources, generators, transformers, and validators, with common fields such as `labels`, `patches`, `images`, `replacements`, and generators acting as convenience syntax over that model.

The upstream components example models components as optional feature bundles. This is the cleanest DRY pattern when several deployment targets need different combinations of resources, generators, and patches.

The current `labels` field should be preferred over deprecated `commonLabels`. `labels` can add labels without automatically changing selectors; selector mutation on live resources can fail because selectors are effectively immutable for many workloads.

The current `vars` docs mark `vars` deprecated in Kustomize v5. Use `replacements` for new field-to-field wiring.

The `patches` field is the modern unified patch entry point. It supports strategic merge and JSON6902 style patches and can target resources by GVK, name, namespace, label selector, or annotation selector.

The generator docs emphasize that ConfigMap and Secret generators append content hashes by default and Kustomize can update workload references. Keep that behavior unless a stable external name is required.

Kustomize transformer configuration examples list built-in ConfigMap and Secret reference paths for Pod, Deployment, ReplicaSet, DaemonSet, StatefulSet, Job, and CronJob. Those examples show the core pod-template path split: most workload controllers use `spec.template`, while CronJob uses `spec.jobTemplate.spec.template`.

The replacements docs show local-config resources can be replacement sources, and field paths can address nested maps, lists, and named list entries. Component docs show components can bundle resources, generators, patches, and transformation logic. Together, these enable a component-owned local `PodTemplate` source to copy structured pod-template fields into differently shaped workload resources.

Local render checks with kubectl's embedded Kustomize showed leaf `configMapGenerator behavior: merge` did not merge a generator defined only inside a component. Put generators that leaf targets must merge or replace in a base or variant resource chain; keep component generators for fixed component-owned config.

The CRD and OpenAPI docs note that Kustomize needs schema or transformer configuration to understand custom resource merge keys and object references. For CRDs, verify rendered output rather than assuming built-in behavior.

Argo CD supports Kustomize overlays and components, can apply inline Kustomize patches in an Application or ApplicationSet, and has explicit Kustomize version/build-option handling. In this skill, that conceptual overlay can be a sibling target directory such as `prod/`; it does not require an `overlays/` parent directory. Argo CD's docs note `ignoreMissingComponents` for default/override patterns.

Flux's Kustomization controller supports SOPS decryption. Flux docs explicitly warn that storing Secrets in Git as plaintext or base64 is unsafe.

## Version Notes

As of the research date, the GitHub releases page marks `kustomize/v5.8.1` as the latest standalone Kustomize release.

Kustomize `v5.8.0` release notes flagged a namespace propagation regression and told users to wait for a patch release. The `v5.8.1` release notes say it fixed the namespace propagation problem and Helm v4 support issues. Prefer `v5.8.1` or newer over `v5.8.0` if relying on that release line.

Kustomize `v5.8.0` added useful replacement behavior for editing structured YAML/JSON values stored inside YAML manifests. If using that behavior, verify that the deployment renderer is at least a fixed v5.8.x release.
