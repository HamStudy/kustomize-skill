# Kustomize Skill

This repository contains an agentic skill for building, reviewing, and improving Kubernetes manifests that use Kustomize.

The skill is designed to help agents produce Kustomize layouts that are DRY, configurable, readable, and accurate. It favors small deployment target kustomizations, reusable components, explicit variant bases, and rendered-output validation over broad template systems or copied YAML.

## Install

Recommended:

```bash
npx skills install HamStudy/kustomize-skill
```

The skill lives in the `kustomize-skill/` subdirectory so skill installers and agentic platforms can discover it as a reusable package.

## Goals

- Keep common Kubernetes resources in one place, usually `base/`.
- Keep target directories such as `dev/`, `staging/`, and `prod/` short and easy to scan.
- Use Kustomize primitives directly: `resources`, `components`, generators, `images`, `replicas`, `labels`, `patches`, and `replacements`.
- Prefer reusable components for optional features such as ingress/certificates, CronJobs, observability, external secrets, and pod-template fragments.
- Share pod-template configuration across compatible workload families without inventing a separate templating language.
- Make image tags, generated config, secrets, schedules, replicas, and resource profiles easy to override in one clear place.
- Preserve readable Kubernetes YAML whenever possible.
- Validate rendered output before handoff.

## What It Covers

The skill includes a compact `SKILL.md` table of contents plus reference files for:

- choosing a Kustomize layout
- avoiding a central `overlays/` directory when a sibling target layout is clearer
- using components for reusable capabilities
- using ConfigMap and Secret generators safely
- replacing deprecated `vars` with `replacements`
- sharing pod template fields between Job/CronJob or Deployment/StatefulSet/DaemonSet variants
- applying opt-in shared env, volume, mount, resource, and security-context fragments
- handling cert-manager, Ingress, CronJobs, CRDs, GitOps renderers, and validation checks

## Design Notes

This is documentation and procedural guidance for agents, not a Kustomize plugin. It does not add new Kustomize syntax. Instead, it teaches agents to compose current Kustomize features in predictable ways and to verify the rendered manifests with the same renderer used by CI or GitOps.

The content is intentionally platform-neutral. Any agentic system that can load a `SKILL.md` file and its adjacent reference files should be able to use the skill.

