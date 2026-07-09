# Resource Recipes

## Contents

- Deployments and Services
- CronJobs
- ConfigMaps and Secrets
- cert-manager Certificates
- Ingress and Gateway resources
- RBAC and ServiceAccounts
- NetworkPolicy
- External secret resources
- Multi-resource components

## Deployments And Services

Put the full Deployment and Service in the base. Keep selectors stable and intentional.

Base Deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  labels:
    app.kubernetes.io/name: api
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: api
  template:
    metadata:
      labels:
        app.kubernetes.io/name: api
    spec:
      serviceAccountName: api
      containers:
        - name: api
          image: ghcr.io/acme/api:latest
          ports:
            - name: http
              containerPort: 8080
          envFrom:
            - configMapRef:
                name: app-config
            - secretRef:
                name: app-bootstrap
```

Target choices:

```yaml
images:
  - name: ghcr.io/acme/api
    newTag: "2026-07-09.1"
replicas:
  - name: api
    count: 4
patches:
  - path: deployment-resources.patch.yaml
```

Use `images` and `replicas` before patches. Patch resource requests, node selectors, affinity, tolerations, topology spread, and probes only when they are environment-specific.

## CronJobs

Put a common CronJob in the base if every target has the job:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cleanup
spec:
  schedule: "0 3 * * *"
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 1
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: cleanup
              image: ghcr.io/acme/api:latest
              args: ["cleanup"]
              envFrom:
                - configMapRef:
                    name: app-config
```

Patch schedule, suspend, or resources in target kustomizations:

```yaml
- op: replace
  path: /spec/schedule
  value: "15 */6 * * *"
- op: replace
  path: /spec/suspend
  value: false
```

If only some environments run the job, move the CronJob into a component such as `components/maintenance-cron/`.

## ConfigMaps And Secrets

Use generators in the base and generator merge in target kustomizations:

```yaml
# base/kustomization.yaml
configMapGenerator:
  - name: app-config
    envs:
      - config/app.env
secretGenerator:
  - name: app-bootstrap
    envs:
      - secrets/bootstrap.env
```

```yaml
# prod/kustomization.yaml
resources:
  - ../base
configMapGenerator:
  - name: app-config
    behavior: merge
    envs:
      - config.env
```

Do not commit real `secrets/bootstrap.env` unless it is encrypted by the repository's secret workflow.

## Cert-Manager Certificates

Use cert-manager `Certificate` resources when cert-manager owns TLS material. The generated TLS Secret should normally have a stable name because it is controlled by cert-manager, not Kustomize.

Base or component Certificate:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: api-cert
spec:
  secretName: api-tls
  issuerRef:
    name: letsencrypt-staging
    kind: ClusterIssuer
  dnsNames:
    - api.dev.example.com
```

Target patch:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: api-cert
spec:
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
    - api.example.com
```

Guidance:

- Put the Certificate in a component if only public-facing targets need TLS.
- Put it in the base if every deployment target needs a certificate.
- Avoid generating the same TLS Secret with `secretGenerator` when cert-manager also owns it.
- If `namePrefix` or `nameSuffix` should transform `spec.secretName`, verify output. Certificate is a CRD, so Kustomize may need transformer configuration or a replacement.

## Ingress And Gateway Resources

Put ingress exposure in a component unless every target has the same exposure model.

Component Ingress:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-staging
spec:
  tls:
    - hosts:
        - api.dev.example.com
      secretName: api-tls
  rules:
    - host: api.dev.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api
                port:
                  name: http
```

Patch hostnames and issuer annotations in target kustomizations. For Gateway API CRDs, verify whether name references and hostnames need replacements or custom transformer config.

## RBAC And ServiceAccounts

Put ServiceAccount and core Role/RoleBinding in the base when all targets use them.

Patch cloud workload identity annotations in cluster targets:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: api
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/prod-api
```

If identity differs by cluster, keep that patch in `clusters/<cluster>/<app>/`, not in the generic `prod/` target.

## NetworkPolicy

Use a baseline NetworkPolicy in the base when the app always has the same traffic shape. Put environment-specific egress, namespace selectors, or monitoring ingress in components.

Component example:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-prometheus
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: api
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: monitoring
      ports:
        - port: http
          protocol: TCP
```

## External Secret Resources

For External Secrets Operator or similar controllers, Kustomize should manage the CRD instance and let the controller create the Kubernetes Secret.

Component example:

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: app-bootstrap
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: app-secrets
    kind: ClusterSecretStore
  target:
    name: app-bootstrap
  dataFrom:
    - extract:
        key: prod/payments/api
```

Patch `secretStoreRef`, remote key, or target name per environment. Verify CRD field transformations explicitly.

## Multi-Resource Components

A good component bundles everything needed for one optional capability:

```text
components/ingress-cert/
  kustomization.yaml
  ingress.yaml
  certificate.yaml
  certificate-host.patch.yaml
  kustomizeconfig/name-reference.yaml
```

Component `kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1alpha1
kind: Component
resources:
  - ingress.yaml
  - certificate.yaml
configurations:
  - kustomizeconfig/name-reference.yaml
```

Targets select it:

```yaml
components:
  - ../components/ingress-cert
patches:
  - path: ingress-host.patch.yaml
```

If every target always selects a component, promote it into the base.
