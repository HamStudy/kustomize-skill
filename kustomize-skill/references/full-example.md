# Full Example

## Contents

- Goal
- File tree
- Base files
- Components
- Dev target
- Prod target
- Render checks

## Goal

Create a Kustomize layout where a tiny target kustomization renders a complete application containing:

- Deployment
- Service
- ServiceAccount
- ConfigMap
- Secret
- CronJob
- cert-manager Certificate
- Ingress
- NetworkPolicy

This example favors readability over every possible production knob.

This example keeps the optional maintenance CronJob as a small component because it is a different task shape from the API Deployment. When the same pod shape can render as either Job/CronJob or Deployment/StatefulSet/DaemonSet, use the variant-base and local `PodTemplate` patterns in `pod-template-dry-patterns.md`.

## File Tree

```text
payments/
  base/
    kustomization.yaml
    deployment.yaml
    service.yaml
    serviceaccount.yaml
    networkpolicy.yaml
    config/app.env
    secrets/bootstrap.env
  components/
    ingress-cert/
      kustomization.yaml
      ingress.yaml
      certificate.yaml
    maintenance-cron/
      kustomization.yaml
      cronjob.yaml
  dev/
    kustomization.yaml
    config.env
    ingress-cert.patch.yaml
  prod/
    kustomization.yaml
    config.env
    ingress-cert.patch.yaml
    deployment-resources.patch.yaml
    cron-schedule.patch.yaml
```

## Base Files

`base/kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
  - serviceaccount.yaml
  - networkpolicy.yaml
configMapGenerator:
  - name: app-config
    envs:
      - config/app.env
secretGenerator:
  - name: app-bootstrap
    envs:
      - secrets/bootstrap.env
generatorOptions:
  labels:
    app.kubernetes.io/managed-by: kustomize
```

`base/config/app.env`

```dotenv
LOG_LEVEL=info
FEATURE_PAYMENTS=true
```

`base/secrets/bootstrap.env`

```dotenv
BOOTSTRAP_TOKEN=replace-in-secret-workflow
```

`base/serviceaccount.yaml`

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: api
```

`base/deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  labels:
    app.kubernetes.io/name: api
spec:
  replicas: 1
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
          image: ghcr.io/acme/payments-api:latest
          ports:
            - name: http
              containerPort: 8080
          envFrom:
            - configMapRef:
                name: app-config
            - secretRef:
                name: app-bootstrap
          readinessProbe:
            httpGet:
              path: /ready
              port: http
          livenessProbe:
            httpGet:
              path: /live
              port: http
```

`base/service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api
spec:
  selector:
    app.kubernetes.io/name: api
  ports:
    - name: http
      port: 80
      targetPort: http
```

`base/networkpolicy.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-default
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: api
  policyTypes: ["Ingress"]
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-nginx
      ports:
        - port: http
          protocol: TCP
```

## Components

`components/ingress-cert/kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1alpha1
kind: Component
resources:
  - ingress.yaml
  - certificate.yaml
```

`components/ingress-cert/certificate.yaml`

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

`components/ingress-cert/ingress.yaml`

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

`components/maintenance-cron/kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1alpha1
kind: Component
resources:
  - cronjob.yaml
```

`components/maintenance-cron/cronjob.yaml`

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
              image: ghcr.io/acme/payments-api:latest
              args: ["cleanup"]
              envFrom:
                - configMapRef:
                    name: app-config
```

## Dev Target

`dev/kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../base
components:
  - ../components/ingress-cert
namespace: payments-dev
labels:
  - pairs:
      app.kubernetes.io/part-of: payments
      environment: dev
    includeTemplates: true
images:
  - name: ghcr.io/acme/payments-api
    newTag: dev
configMapGenerator:
  - name: app-config
    behavior: merge
    envs:
      - config.env
patches:
  - path: ingress-cert.patch.yaml
```

`dev/config.env`

```dotenv
LOG_LEVEL=debug
PAYMENTS_API_MODE=dev
```

`dev/ingress-cert.patch.yaml`

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: api-cert
spec:
  issuerRef:
    name: letsencrypt-staging
    kind: ClusterIssuer
  dnsNames:
    - api.dev.example.com
---
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

## Prod Target

`prod/kustomization.yaml`

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
  - path: ingress-cert.patch.yaml
  - path: deployment-resources.patch.yaml
  - path: cron-schedule.patch.yaml
    target:
      group: batch
      version: v1
      kind: CronJob
      name: cleanup
```

`prod/config.env`

```dotenv
LOG_LEVEL=warn
PAYMENTS_API_MODE=prod
```

`prod/ingress-cert.patch.yaml`

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
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
    - hosts:
        - api.example.com
      secretName: api-tls
  rules:
    - host: api.example.com
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

`prod/deployment-resources.patch.yaml`

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
            limits:
              cpu: "1"
              memory: 512Mi
```

`prod/cron-schedule.patch.yaml`

```yaml
- op: replace
  path: /spec/schedule
  value: "15 */6 * * *"
```

The target kustomization points at this JSON6902 patch explicitly because JSON6902 patch files need a target selector.

## Render Checks

Run:

```bash
kustomize build dev
kustomize build prod
```

Check that:

- `Deployment/api` references hashed `app-config` and `app-bootstrap` names.
- `CronJob/cleanup` uses the same image tag as the Deployment.
- `Certificate/api-cert` keeps `spec.secretName: api-tls`.
- `Ingress/api` and `Certificate/api-cert` agree on hostname and TLS secret.
- Dev does not render the maintenance CronJob; prod does.
- No real secret values are committed in `base/secrets/bootstrap.env`.
