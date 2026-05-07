# PLM Cloud Kubernetes Deployment

This directory contains a single-namespace Kubernetes deployment for the current PLM Cloud stack:

- PostgreSQL
- Redis
- plm-infrastructure migration job
- plm-auth-service
- plm-attribute-service
- plm-gateway
- plm-cloud-frontend
- optional Ingress

## Structure

- `base/`: reusable base manifests shared by all environments
- `apps/local/`: local overlay, including namespace, ConfigMap, SealedSecret, and ingress
- `apps/demo/`: domain overlay for `demo.altair288.eu.org`, including ConfigMap, SealedSecret, and ingress
- `kustomization.yaml`: convenience entrypoint that points to `apps/local`

For Argo CD, the recommended path is:

- `SYSTEM-DEPLOY/Kubernetes/apps/demo`

That keeps the environment-specific overlay as the sync target and leaves `base/` reusable for future environments such as `apps/dev` or `apps/prod`.

## Sealed Secrets

Plaintext application secrets should not be committed in this repository.
This repo now expects each overlay to provide a `sealed-secret.yaml` with the fixed Secret name `plm-cloud-secret`.

Before your first sync, generate the real encrypted file on the target cluster or on a machine that has:

- `kubectl` access to the target cluster
- `kubeseal` installed
- the Sealed Secrets controller already installed in that cluster

Recommended overlays:

- local: `apps/local/sealed-secret.yaml`
- demo: `apps/demo/sealed-secret.yaml`

### Step By Step: Generate A Sealed Secret

1. Find the controller namespace and name on the real server:

```bash
kubectl get pods -A | grep sealed-secrets
```

Typical results are one of these:

- namespace `sealed-secrets`, controller name `sealed-secrets`
- namespace `kube-system`, controller name `sealed-secrets`

1. Fetch the cluster public certificate from the real server:

```bash
kubeseal \
   --controller-name sealed-secrets \
   --controller-namespace sealed-secrets \
   --fetch-cert > sealed-secrets-cert.pem
```

If your controller runs in another namespace, change `--controller-namespace` accordingly.

1. Generate the local overlay sealed secret directly, without creating a plaintext file in the repo:

```bash
kubectl create secret generic plm-cloud-secret \
   --namespace plm-cloud \
   --from-literal=POSTGRES_PASSWORD='REPLACE_ME' \
   --from-literal=REDIS_PASSWORD='REPLACE_ME' \
   --from-literal=PLM_AUTH_PLATFORM_ADMIN_PASSWORD='REPLACE_ME' \
   --from-literal=PLM_AUTH_RESEND_API_KEY='REPLACE_ME' \
   --dry-run=client -o yaml \
| kubeseal \
   --format yaml \
   --cert sealed-secrets-cert.pem \
> apps/local/sealed-secret.yaml
```

1. Generate the demo overlay sealed secret in the same way:

```bash
kubectl create secret generic plm-cloud-secret \
   --namespace plm-cloud \
   --from-literal=POSTGRES_PASSWORD='REPLACE_ME' \
   --from-literal=REDIS_PASSWORD='REPLACE_ME' \
   --from-literal=PLM_AUTH_PLATFORM_ADMIN_PASSWORD='REPLACE_ME' \
   --from-literal=PLM_AUTH_RESEND_API_KEY='REPLACE_ME' \
   --dry-run=client -o yaml \
| kubeseal \
   --format yaml \
   --cert sealed-secrets-cert.pem \
> apps/demo/sealed-secret.yaml
```

1. Review the generated file. It should contain:

- `apiVersion: bitnami.com/v1alpha1`
- `kind: SealedSecret`
- `metadata.name: plm-cloud-secret`
- encrypted `spec.encryptedData`

1. Commit the generated `sealed-secret.yaml` to git.

1. Sync Argo CD or apply the overlay normally.

### Important Notes

- Do not generate SealedSecrets on a random local machine that does not target the real cluster.
- SealedSecrets are encrypted against the target cluster controller key pair, so generation must match the cluster that will decrypt them.
- The placeholder `sealed-secret.yaml` files committed in this repo must be replaced with real encrypted output before deployment.
- Because these plaintext values were already committed before, rotate them on the real environment even if they were only test credentials.

## Before Apply

1. Replace the non-secret values in `apps/local/config.yaml` or `apps/demo/config.yaml` as needed:
    - `PLM_PUBLIC_BASE_URL`
    - invitation URL templates
2. Replace the placeholder `sealed-secret.yaml` in the target overlay with real kubeseal output.
3. If your GHCR images are private, create an image pull secret in the `plm-cloud` namespace and attach it to the Deployments.
4. If your cluster has a default StorageClass, the PVCs will bind automatically. If not, add `storageClassName` to the PVC definitions.
5. Change the host in the target overlay ingress file if needed.

## Local Testing Base URL

For local Kubernetes testing, keep:

- `PLM_PUBLIC_BASE_URL=http://localhost:3000`
- `PLM_AUTH_INVITATION_EMAIL_ACCEPT_URL_TEMPLATE=http://localhost:3000/invite/email?token=%s`
- `PLM_AUTH_INVITATION_LINK_ACCEPT_URL_TEMPLATE=http://localhost:3000/invite/link?token=%s`

This repository's default local path is to expose the frontend with `kubectl port-forward svc/frontend 3000:3000 -n plm-cloud`.
In that mode, the browser-visible origin really is `http://localhost:3000`, so the port must stay in the URL.

Only remove `:3000` when you switch to a real Ingress or LoadBalancer that exposes the app on standard `80` or `443`.

## Local Startup

1. Make sure your local cluster is running.
2. Review `apps/local/config.yaml` and confirm local values are correct.
3. Apply the stack:

```bash
kubectl apply -k .
```

1. Wait for the data layer:

```bash
kubectl rollout status statefulset/postgres -n plm-cloud
kubectl rollout status statefulset/redis -n plm-cloud
```

1. Wait for the migration job:

```bash
kubectl wait --for=condition=complete job/plm-infrastructure-migrate -n plm-cloud --timeout=300s
```

1. Wait for the application workloads:

```bash
kubectl rollout status deployment/auth-service -n plm-cloud
kubectl rollout status deployment/attribute-service -n plm-cloud
kubectl rollout status deployment/gateway -n plm-cloud
kubectl rollout status deployment/frontend -n plm-cloud
```

1. Expose the frontend locally:

```bash
kubectl port-forward svc/frontend 3000:3000 -n plm-cloud
```

1. Open:

```text
http://localhost:3000
```

1. Optional checks:

```bash
kubectl get pods -n plm-cloud
kubectl logs job/plm-infrastructure-migrate -n plm-cloud
kubectl logs deployment/auth-service -n plm-cloud
kubectl logs deployment/gateway -n plm-cloud
```

## Apply Order

The manifests can be rendered together with kustomize, but the operational order should still be:

1. `kubectl apply -f apps/local/namespace.yaml`
2. `kubectl apply -f apps/local/config.yaml -n plm-cloud`
3. `kubectl apply -f base/data.yaml -n plm-cloud`
4. `kubectl rollout status statefulset/postgres -n plm-cloud`
5. `kubectl rollout status statefulset/redis -n plm-cloud`
6. `kubectl apply -f base/migration-job.yaml -n plm-cloud`
7. `kubectl wait --for=condition=complete job/plm-infrastructure-migrate -n plm-cloud --timeout=300s`
8. `kubectl apply -f base/apps.yaml -n plm-cloud`
9. `kubectl rollout status deployment/auth-service -n plm-cloud`
10. `kubectl rollout status deployment/attribute-service -n plm-cloud`
11. `kubectl rollout status deployment/gateway -n plm-cloud`
12. `kubectl rollout status deployment/frontend -n plm-cloud`
13. `kubectl apply -f apps/local/ingress.yaml -n plm-cloud`

If you prefer a single render step:

```bash
kubectl apply -k .
```

Then explicitly wait for the migration job and the app rollouts.

## Notes

- Frontend proxies `/auth/*` and `/api/*` internally to `http://gateway:8080`, matching the standalone image behavior validated in Docker.
- Gateway routes are fully declared through environment variables so Spring Cloud Gateway does not partially rebuild route list items.
- Auth and attribute wait for PostgreSQL, and auth also waits for Redis.
- Auth and attribute both wait until the Flyway history table exists before starting, which prevents booting against an unmigrated database.
- Redis now uses a StatefulSet plus `volumeClaimTemplates`, which is more consistent with persistent data workloads and avoids binding a single hand-managed PVC to a Deployment.
- These manifests were rendered locally with `kubectl kustomize` on the current workstation. They have not been applied to a live cluster from this session.
