# Pod Template DRY Patterns

## Contents

- Mental model
- Pod template path families
- Choosing the right pattern
- Pod profile components
- Job and CronJob variants
- Replacements for scalar fan-out
- CRD workload controllers
- When Kustomize is the wrong abstraction
- Review checklist

## Mental Model

Many Kubernetes resources own a Pod template, but Kustomize does not provide a native "PodSpec fragment include" feature. Kustomize composes complete resources, then transforms fields.

The Kustomize-native DRY pattern is:

1. Keep each workload resource as a small, valid wrapper for its kind.
2. Put common pod behavior into a component that applies targeted patches.
3. Let built-in transformers handle images, labels, generated ConfigMap names, and generated Secret names where they already know the workload kind.
4. Use replacements for repeated scalar values or explicit field-to-field wiring.
5. Stop and use a generator or template language if one source PodSpec must generate many resource kinds exactly.

## Pod Template Path Families

Use this table before writing patches or replacements:

| Kind | API group | Pod template path | Container path |
| --- | --- | --- | --- |
| `Pod` | core/v1 | `spec` | `spec.containers` |
| `Deployment` | apps/v1 | `spec.template` | `spec.template.spec.containers` |
| `StatefulSet` | apps/v1 | `spec.template` | `spec.template.spec.containers` |
| `DaemonSet` | apps/v1 | `spec.template` | `spec.template.spec.containers` |
| `ReplicaSet` | apps/v1 | `spec.template` | `spec.template.spec.containers` |
| `Job` | batch/v1 | `spec.template` | `spec.template.spec.containers` |
| `CronJob` | batch/v1 | `spec.jobTemplate.spec.template` | `spec.jobTemplate.spec.template.spec.containers` |

Deployment, StatefulSet, DaemonSet, ReplicaSet, and Job share the `spec.template` path family. CronJob does not. It wraps the Pod template in `spec.jobTemplate.spec.template`.

## Choosing The Right Pattern

Use plain Kustomize primitives first:

- Use `images` for image changes across supported workload kinds.
- Use `labels` with `includeTemplates: true` for labels that belong on Pods.
- Use ConfigMap and Secret generators in the base or component, then refer to generated resources by logical name in pod fields.
- Use `replicas` only for resources that have a replica count. Do not try to force Jobs or CronJobs into that model.
- Use a component when the same pod behavior is optional or shared across more than one target or workload kind.

Use a pod profile component for shared pod concerns:

- `serviceAccountName`
- pod `securityContext`
- container `securityContext`
- `env`, `envFrom`, and generated ConfigMap/Secret references
- resource requests and limits
- probes
- volumes and volume mounts
- node selectors, affinity, tolerations, topology spread, priority class
- sidecars or init containers

Keep kind-specific behavior outside the shared pod profile:

- Deployment rollout strategy
- StatefulSet `serviceName`, update strategy, ordinals, and `volumeClaimTemplates`
- DaemonSet update strategy
- Job backoff, completions, parallelism, and TTL
- CronJob schedule, concurrency policy, suspend state, history limits, and timezone

## Pod Profile Components

Use top-level metadata labels or annotations to opt workload resources into a profile. Do not use selector labels for this. A metadata label selected by Kustomize patches does not need to appear on Pods.

Example workload opt-in:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: worker
  labels:
    pod-profile.acme.io/worker: "true"
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: worker
  template:
    metadata:
      labels:
        app.kubernetes.io/name: worker
    spec:
      containers:
        - name: worker
          image: ghcr.io/acme/worker:latest
```

Example component:

```yaml
apiVersion: kustomize.config.k8s.io/v1alpha1
kind: Component
configMapGenerator:
  - name: worker-config
    envs:
      - config/worker.env
patches:
  - path: deployment.pod-template.patch.yaml
    target:
      group: apps
      version: v1
      kind: Deployment
      labelSelector: pod-profile.acme.io/worker=true
  - path: statefulset.pod-template.patch.yaml
    target:
      group: apps
      version: v1
      kind: StatefulSet
      labelSelector: pod-profile.acme.io/worker=true
  - path: daemonset.pod-template.patch.yaml
    target:
      group: apps
      version: v1
      kind: DaemonSet
      labelSelector: pod-profile.acme.io/worker=true
  - path: job.pod-template.patch.yaml
    target:
      group: batch
      version: v1
      kind: Job
      labelSelector: pod-profile.acme.io/worker=true
  - path: cronjob.pod-template.patch.yaml
    target:
      group: batch
      version: v1
      kind: CronJob
      labelSelector: pod-profile.acme.io/worker=true
```

Strategic merge patch files still need to be parseable resources, so include `apiVersion`, `kind`, and `metadata.name`. When `target` selects by label, the patch name can be a clearly inert name such as `ignored-by-target`; render to verify it applies only where intended.

Deployment, StatefulSet, DaemonSet, and Job patches use the same pod-template nesting but different `apiVersion` or `kind`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ignored-by-target
spec:
  template:
    spec:
      serviceAccountName: worker
      securityContext:
        runAsNonRoot: true
      containers:
        - name: worker
          envFrom:
            - configMapRef:
                name: worker-config
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              memory: 256Mi
```

The CronJob patch has the same pod details under the CronJob path:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: ignored-by-target
spec:
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: worker
          securityContext:
            runAsNonRoot: true
          containers:
            - name: worker
              envFrom:
                - configMapRef:
                    name: worker-config
              resources:
                requests:
                  cpu: 100m
                  memory: 128Mi
                limits:
                  memory: 256Mi
```

This is not zero repetition. It is the Kustomize-native compromise: one component owns the shared behavior, and each patch file accounts for Kubernetes' real schema shape.

## Job And CronJob Variants

If a task is normally scheduled and only sometimes run manually, define the CronJob and create one-off Jobs from it at runtime:

```bash
kubectl create job report-manual-$(date +%s) --from=cronjob/report
```

If Git must render either a Job or a CronJob, keep two tiny wrapper resources and select the same pod profile component from each target:

```text
task/
  base/
  components/
    worker-pod-profile/
  run/
    kustomization.yaml
    job.yaml
  schedule/
    kustomization.yaml
    cronjob.yaml
```

The wrappers should contain only kind-specific fields and the opt-in label. The component should own the common pod settings.

## Replacements For Scalar Fan-Out

Use replacements when the same rendered value must be copied into several pod-template paths. This is useful for command args, env values, CRD fields, or values that come from generated resources.

```yaml
replacements:
  - source:
      kind: ConfigMap
      name: worker-settings
      fieldPath: data.queue
    targets:
      - select:
          kind: Job
          name: worker
        fieldPaths:
          - spec.template.spec.containers.[name=worker].env.[name=QUEUE].value
        options:
          create: true
      - select:
          kind: CronJob
          name: worker
        fieldPaths:
          - spec.jobTemplate.spec.template.spec.containers.[name=worker].env.[name=QUEUE].value
        options:
          create: true
```

Prefer replacements for field values, not whole pod templates. Whole-template copying is possible only in narrow cases and tends to become harder to understand than the repeated YAML.

## CRD Workload Controllers

Argo Rollouts, Tekton, Knative, custom operators, and other CRDs may also contain Pod templates. Kustomize does not automatically know every CRD field.

For CRD workload controllers:

- prefer explicit patches for CRD pod-template paths
- add transformer configuration for custom image fields or name references
- use replacements for custom fields that need generated names or shared values
- render and inspect the CRD output every time

## When Kustomize Is The Wrong Abstraction

Kustomize is a transformer, not a polymorphic workload generator. If the requirement is "write one PodSpec once and generate Deployment, Job, CronJob, StatefulSet, and DaemonSet wrappers from it," use a generator before Kustomize.

Reasonable generator layers include:

- a small checked-in script that emits plain Kubernetes YAML
- CUE, Jsonnet, or another data templating tool
- Helm, when the chart model is already accepted by the team
- a KRM function, when the deployment system supports it

Keep Kustomize after that layer for environment composition, image changes, generated ConfigMaps/Secrets, labels, namespace, and final patches.

## Review Checklist

Before handing off a pod-template DRY design, confirm:

- the shared component is optional only when it is truly optional
- target resources opt in with metadata labels or annotations, not selector labels
- CronJob patches use `spec.jobTemplate.spec.template`
- Job patches use `spec.template`
- container patches merge by container `name`, not list index
- generated ConfigMap and Secret references are updated in every workload kind
- kind-specific behavior stays in kind-specific wrappers or target patches
- rendered output is easier to read than the source abstraction
