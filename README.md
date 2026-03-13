# Argo CD Progressive Sync POC

This repository is a small proof of concept for Argo CD `RollingSync` progressive syncs on Argo CD `3.3.3`.

It demonstrates three rollout stages:

- gcp first
- infrastructure second
- backends third
- frontends last

The generated Applications do not move to the next stage until the current stage reports `Healthy`.

This version of the POC uses a mixed deployment model:

- infrastructure Applications are plain manifests in this repo
- backend and frontend Applications are Helm charts plus values files

## What This Shows

- stage-based ordering across many Applications instead of one large sync wave inside a single app
- health-gated progression so frontends do not start until gcp, infrastructure, and backends are healthy
- limited concurrency for noisy tiers that may contain many apps

The example uses:

- 1 gcp placeholder Application that simulates cloud resource provisioning with a 10-second startup delay
- 1 infrastructure Application that deploys PostgreSQL, MongoDB, Envoy, seed jobs, and shared credentials
- 4 backend placeholder Applications synced in batches of 2
- 3 frontend placeholder Applications synced one at a time

Charts used:

- `bitnami/nginx` for backends and frontends

## Repository Layout

```text
bootstrap/
  root-app.yaml
  core/
    project.yaml
    progressive-sync-appset.yaml
gcp/
  resources.yaml
apps/
  infrastructure/
    infrastructure.yaml
    envoy.yaml
    seed-jobs.yaml
    shared-secrets.yaml
services/
  backend/
    orders-api/
      values.yaml
    users-api/
      values.yaml
    payments-api/
      values.yaml
    catalog-api/
      values.yaml
  frontend/
    web-ui/
      values.yaml
    admin-ui/
      values.yaml
    support-ui/
      values.yaml
```

## Prerequisites

- Argo CD `3.3.3`
- the progressive sync feature enabled on the ApplicationSet controller
- a cluster where Argo CD can create resources in the `progressive-sync-poc` namespace

Reference documentation:

- https://argo-cd.readthedocs.io/en/release-3.3/operator-manual/applicationset/Progressive-Syncs/

The easiest feature flag for Argo CD `3.3.3` is:

```yaml
data:
  applicationsetcontroller.enable.progressive.syncs: "true"
```

in `argocd-cmd-params-cm`.

## Install

1. Replace `https://github.com/trojocerqlar/argocd-test` in:
   - `bootstrap/root-app.yaml`
   - `bootstrap/core/progressive-sync-appset.yaml`
2. Apply the root Application:

```bash
kubectl apply -n argocd -f bootstrap/root-app.yaml
```

3. In Argo CD, sync `progressive-sync-poc-bootstrap`.

## Expected Rollout Behavior

1. The `gcp` Application syncs first and stays pending for about 10 seconds.
2. When the `gcp` Application is `Healthy`, the `infrastructure` Application starts.
3. When the infrastructure Application is `Healthy`, the backend stage starts.
4. Backends roll out in batches of 2.
5. When all backend Applications are `Healthy`, frontends start.
6. Frontends roll out one at a time.

If one Application stays unhealthy, the rollout pauses on that stage.

Generated Applications intentionally do not use `spec.syncPolicy.automated`, because `RollingSync` disables autosync for managed Applications.

Backend and frontend Applications use Argo CD multiple sources:

- source 1: an external Helm chart
- source 2: this Git repository as a values-only source via `$services/...`

Infrastructure Applications use a normal Git directory source so each app can contain:

- a shared Secret
- infrastructure Services and Deployments
- an Envoy gateway configuration
- separate seed Jobs

Backend values files inject credentials from the `backend-service-credentials` Secret.

## Why This POC Is Intentionally Simple

- infrastructure is plain manifests so the runtime, secret, and seeding steps stay obvious
- backends and frontends reuse one generic chart to keep the example easy to scale out
- only the services that already follow your Helm deployment model use values files

To scale the demo up, add another values file and one more element to the ApplicationSet list generator.
