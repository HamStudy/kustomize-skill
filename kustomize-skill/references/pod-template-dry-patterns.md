# Pod Template DRY Patterns

## Contents

- Mental model
- Choose a workload family
- Layout: variant bases plus leaf targets
- Job or CronJob pattern
- Deployment, StatefulSet, or DaemonSet pattern
- Whole-template copy vs selected-field copy
- Common container fields
- Overrides and configurability
- Component rules
- What stays in variant wrappers
- Patch fallback
- Review checklist

## Mental Model

Do not design one abstraction that supports every possible workload kind unless the user actually needs that. Most real choices are family-shaped:

- "This task can be a `Job` or a `CronJob`."
- "This service can be a `Deployment`, `StatefulSet`, or `DaemonSet`."
- "This target should use the scheduled variant; that target should use the one-off variant."

The Kustomize-native pattern is:

1. Put the common pod shape in one component.
2. Store that pod shape as a local `PodTemplate`.
3. Use the component's `replacements` to copy the common pod fields into compatible workload variants.
4. Put each workload kind in a small variant base such as `job/`, `cronjob/`, `deployment/`, or `statefulset/`.
5. Let leaf targets extend exactly the variant they want.

This gives the user one reusable pod definition without making every target render every possible workload kind.

## Choose A Workload Family

Use separate variant families when pod semantics differ.

Job family:

- `Job`
- `CronJob`
- usually safe for whole-template copy because both use Job-style Pod semantics
- keep schedule, suspend, history, backoff, TTL, completions, and parallelism in the wrappers

Long-running controller family:

- `Deployment`
- `StatefulSet`
- `DaemonSet`
- often safe for whole-template copy when the Pod template has only normal long-running Pod fields
- keep selector, replicas, rollout strategy, StatefulSet service/volume claims, and DaemonSet update strategy in wrappers

Mixed family:

- `Deployment` plus `Job` or `CronJob`
- prefer selected-field copy
- keep fields such as Job `restartPolicy` out of the shared template

CRD workload family:

- Argo Rollout, Tekton, Knative, or custom operator workloads
- use explicit replacement field paths for the CRD's pod-template shape
- verify rendered output every time

## Layout: Variant Bases Plus Leaf Targets

Use a component for the shared pod template and variant bases for the possible workload kinds:

```text
task/
  base/
    kustomization.yaml
  components/
    task-pod/
      kustomization.yaml
      pod-template.yaml
  job/
    kustomization.yaml
    job.yaml
  cronjob/
    kustomization.yaml
    cronjob.yaml
  prod/
    kustomization.yaml
  run-now/
    kustomization.yaml
```

`base/` owns mergeable defaults such as generated ConfigMaps. `job/` and `cronjob/` are variant bases, not environment targets. The leaf targets choose one:

```yaml
# prod/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../cronjob
images:
  - name: busybox
    newTag: "1.37"
configMapGenerator:
  - name: task-config
    behavior: merge
    literals:
      - LOG_LEVEL=warn
```

```yaml
# run-now/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../job
images:
  - name: busybox
    newTag: "1.37"
```

This keeps the leaf target small and clear: it extends the version it wants.

## Job Or CronJob Pattern

Use whole-template copy when a task can be either a `Job` or a `CronJob`.

`base/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
configMapGenerator:
  - name: task-config
    literals:
      - QUEUE=default
      - LOG_LEVEL=info
```

`components/task-pod/pod-template.yaml`:

```yaml
apiVersion: v1
kind: PodTemplate
metadata:
  name: task-pod
  annotations:
    config.kubernetes.io/local-config: "true"
template:
  metadata:
    labels:
      app.kubernetes.io/name: task
  spec:
    restartPolicy: Never
    serviceAccountName: task
    containers:
      - name: task
        image: busybox:1.36
        command: ["sh", "-c", "echo hello"]
        envFrom:
          - configMapRef:
              name: task-config
```

`components/task-pod/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1alpha1
kind: Component
resources:
  - pod-template.yaml
replacements:
  - source:
      kind: PodTemplate
      name: task-pod
      fieldPath: template
    targets:
      - select:
          kind: Job
          name: task
        fieldPaths:
          - spec.template
      - select:
          kind: CronJob
          name: task
        fieldPaths:
          - spec.jobTemplate.spec.template
```

`job/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../base
  - job.yaml
components:
  - ../components/task-pod
```

`job/job.yaml`:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: task
spec:
  template:
    metadata:
      labels:
        app.kubernetes.io/name: task
    spec:
      restartPolicy: Never
      containers:
        - name: task
          image: placeholder
```

`cronjob/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../base
  - cronjob.yaml
components:
  - ../components/task-pod
```

`cronjob/cronjob.yaml`:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: task
spec:
  schedule: "0 * * * *"
  jobTemplate:
    spec:
      template:
        metadata:
          labels:
            app.kubernetes.io/name: task
        spec:
          restartPolicy: Never
          containers:
            - name: task
              image: placeholder
```

The component can list replacement targets for both kinds. A variant that contains only `Job` renders the Job; a variant that contains only `CronJob` renders the CronJob. Leaf targets can override the generated ConfigMap with `behavior: merge` because the generator is in the resource chain, not hidden inside the component. Still render each variant with the real deployment renderer.

## Deployment, StatefulSet, Or DaemonSet Pattern

For long-running controllers, make the same shape:

```text
service/
  components/
    web-pod/
      kustomization.yaml
      pod-template.yaml
  deployment/
    kustomization.yaml
    deployment.yaml
  statefulset/
    kustomization.yaml
    statefulset.yaml
  daemonset/
    kustomization.yaml
    daemonset.yaml
  prod/
    kustomization.yaml
```

`components/web-pod/pod-template.yaml` should avoid Job-only fields such as `restartPolicy: Never`:

```yaml
apiVersion: v1
kind: PodTemplate
metadata:
  name: web-pod
  annotations:
    config.kubernetes.io/local-config: "true"
template:
  metadata:
    labels:
      app.kubernetes.io/name: web
  spec:
    serviceAccountName: web
    containers:
      - name: web
        image: ghcr.io/acme/web:latest
        ports:
          - name: http
            containerPort: 8080
        envFrom:
          - configMapRef:
              name: web-config
```

The component can copy the whole template into controller variants:

```yaml
apiVersion: kustomize.config.k8s.io/v1alpha1
kind: Component
resources:
  - pod-template.yaml
configMapGenerator:
  - name: web-config
    envs:
      - config/web.env
replacements:
  - source:
      kind: PodTemplate
      name: web-pod
      fieldPath: template
    targets:
      - select:
          kind: Deployment
          name: web
        fieldPaths:
          - spec.template
      - select:
          kind: StatefulSet
          name: web
        fieldPaths:
          - spec.template
      - select:
          kind: DaemonSet
          name: web
        fieldPaths:
          - spec.template
```

The `deployment/`, `statefulset/`, and `daemonset/` directories keep controller-specific fields. For example, StatefulSet keeps `serviceName` and `volumeClaimTemplates`; Deployment keeps `replicas` and strategy; DaemonSet keeps DaemonSet update strategy.

Leaf targets choose one:

```yaml
# prod/kustomization.yaml
resources:
  - ../deployment
```

```yaml
# edge-node/kustomization.yaml
resources:
  - ../daemonset
```

## Whole-Template Copy Vs Selected-Field Copy

Whole-template copy is shortest and best within a compatible family:

```yaml
fieldPath: template
fieldPaths:
  - spec.template
  - spec.jobTemplate.spec.template
```

Use selected-field copy when a target kind needs to keep some pod fields in its wrapper:

```yaml
replacements:
  - source:
      kind: PodTemplate
      name: mixed-pod
      fieldPath: template.spec.containers
    targets:
      - select:
          kind: Deployment
          name: mixed
        fieldPaths:
          - spec.template.spec.containers
      - select:
          kind: Job
          name: mixed
        fieldPaths:
          - spec.template.spec.containers
```

Good selected fields include:

- `template.metadata.labels`
- `template.metadata.annotations`
- `template.spec.serviceAccountName`
- `template.spec.securityContext`
- `template.spec.containers`
- `template.spec.initContainers`
- `template.spec.volumes`
- `template.spec.nodeSelector`
- `template.spec.affinity`
- `template.spec.tolerations`
- `template.spec.topologySpreadConstraints`
- `template.spec.priorityClassName`

Container subfields can also be shared by name:

- `template.spec.containers.[name=shared].env`
- `template.spec.containers.[name=shared].envFrom`
- `template.spec.containers.[name=shared].resources`
- `template.spec.containers.[name=shared].volumeMounts`
- `template.spec.containers.[name=shared].securityContext`

Do not rely on patching fields inside a subtree after replacing that subtree. A subtree replacement can overwrite earlier patch results. Copy smaller subfields when kind-specific values must remain in the wrapper.

## Common Container Fields

When workload kinds share env, resource defaults, or mounts but not command, args, ports, or probes, define a placeholder container in the local `PodTemplate` and copy only selected fields into the real containers.

`components/shared-container/pod-template.yaml`:

```yaml
apiVersion: v1
kind: PodTemplate
metadata:
  name: shared-container-fields
  annotations:
    config.kubernetes.io/local-config: "true"
template:
  spec:
    containers:
      - name: shared
        envFrom:
          - configMapRef:
              name: app-config
          - secretRef:
              name: app-bootstrap
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            memory: 256Mi
```

`components/shared-container/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1alpha1
kind: Component
resources:
  - pod-template.yaml
configMapGenerator:
  - name: app-config
    literals:
      - LOG_LEVEL=info
secretGenerator:
  - name: app-bootstrap
    envs:
      - secrets/bootstrap.env
replacements:
  - source:
      kind: PodTemplate
      name: shared-container-fields
      fieldPath: template.spec.containers.[name=shared].envFrom
    targets:
      - select:
          kind: Deployment
          name: api
        fieldPaths:
          - spec.template.spec.containers.[name=api].envFrom
        options:
          create: true
      - select:
          kind: CronJob
          name: cleanup
        fieldPaths:
          - spec.jobTemplate.spec.template.spec.containers.[name=cleanup].envFrom
        options:
          create: true
  - source:
      kind: PodTemplate
      name: shared-container-fields
      fieldPath: template.spec.containers.[name=shared].resources
    targets:
      - select:
          kind: Deployment
          name: api
        fieldPaths:
          - spec.template.spec.containers.[name=api].resources
        options:
          create: true
      - select:
          kind: CronJob
          name: cleanup
        fieldPaths:
          - spec.jobTemplate.spec.template.spec.containers.[name=cleanup].resources
        options:
          create: true
```

The real workload YAML stays close to Kubernetes:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: api
  template:
    metadata:
      labels:
        app.kubernetes.io/name: api
    spec:
      containers:
        - name: api
          image: ghcr.io/acme/api:latest
          ports:
            - name: http
              containerPort: 8080
```

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cleanup
spec:
  schedule: "0 3 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: cleanup
              image: ghcr.io/acme/api:latest
              args: ["cleanup"]
```

This pattern defines env and resources once, while the wrapper keeps kind-specific shape, ports, command, args, and schedule.

## Overrides And Configurability

Prefer these override paths:

- Image tag or digest: use `images` in the leaf target. Keep the image name stable in the source template and wrappers so one `images` entry reaches every rendered workload.
- Environment values: use `configMapGenerator` in `base/` or a variant base, then use `behavior: merge` in the leaf target. Do not rely on a leaf generator to merge a generator that exists only inside a component.
- Secret material: use the repository's secret workflow, External Secrets, SOPS, or local-only generator inputs; do not put real secret literals in reusable examples.
- Resource requests/limits: copy shared defaults from the component only for variants that use those defaults. If one variant needs different resources, either exclude that target from the resource replacement and keep resources in its wrapper, or make a second small component for that profile.
- Volumes and volume mounts: copy common lists only when the lists are meant to be identical. Use a patch-profile component for additive per-variant mounts.

Do not assume a leaf patch can override a field that was copied by `replacements`. Depending on transformer order, the replacement may win. To override cleanly, avoid copying that specific field into the variant that needs a different value.

## Component Rules

The pod-template component may contain:

- the local `PodTemplate`
- replacements that copy pod fields into each supported variant kind
- ConfigMap and Secret generators only when leaf targets do not need generator `merge` or `replace`
- patch fallbacks for merge-style additions
- transformer configuration for CRD workload fields

The component should not contain:

- real workload wrapper resources
- environment-specific image tags, replicas, schedules, or hostnames
- every possible workload kind just because Kustomize can target it

Prefer one component per logical pod shape, such as `task-pod`, `web-pod`, or `worker-pod`.

## What Stays In Variant Wrappers

Keep kind-specific behavior in variant bases:

- Deployment: selector, replicas, rollout strategy
- StatefulSet: selector, service name, update strategy, ordinals, volume claim templates
- DaemonSet: selector and update strategy
- Job: backoff, completions, parallelism, TTL, restart policy
- CronJob: schedule, concurrency policy, suspend state, history limits, timezone, restart policy

For existing live workloads, keep selector changes especially visible. Selector label mutation often requires recreation.

## Patch Fallback

Use patches when the desired behavior is an additive merge, not a replacement. Common examples:

- add one sidecar only to Deployment
- add one volume only to StatefulSet
- add one toleration only to DaemonSet
- patch CronJob schedule in a leaf target

Patch files still need to match the real resource path:

- Job: `spec.template`
- CronJob: `spec.jobTemplate.spec.template`
- Deployment/StatefulSet/DaemonSet: `spec.template`

## Review Checklist

Before handing off a pod-template DRY design, confirm:

- the design supports the actual choice family, not every workload kind in Kubernetes
- leaf targets extend exactly one variant base when only one kind should render
- the pod-template component owns the local `PodTemplate` and replacements
- fixed component generators are used only when leaf targets do not need to merge or replace them
- local `PodTemplate` resources are annotated `config.kubernetes.io/local-config: "true"`
- local `PodTemplate` resources do not render in final output
- whole-template copy is used only inside compatible families
- selected-field copy preserves kind-specific wrapper fields
- generated ConfigMap and Secret references are updated in rendered workloads
- image transforms still reach copied containers
- rendered output is easier to review than the source abstraction
