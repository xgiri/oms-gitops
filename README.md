# gitops — environment-first structure

## Bootstrap (one-time, manual)
`dev` is Docker Desktop's own Kubernetes — the same cluster Argo CD itself
runs in — so it needs no `argocd cluster add` step, just:
```
kubectl apply -f bootstrap/root-dev.yaml
```
`prod` is a real separate cluster, registered first:
```
argocd cluster add <prod-context> --name prod
kubectl apply -f bootstrap/root-prod.yaml --context <prod-context>
```
Each `root-<env>` Application then manages everything under
`bootstrap/children/<env>/` for that environment/cluster only.

## dev == Docker Desktop, running alongside docker-compose
Postgres, Kafka, Redis, and `oms-main` all still run via docker-compose on
the same machine — dev's Kubernetes workloads reach them through Docker
Desktop's `host.docker.internal` DNS name rather than standing up a second
copy of each in-cluster. See `environments/dev/apps/shipment-service/
kustomization.yaml` for the pattern. Apply the same pattern to the other 5
services as you bring each one into dev — `DB_HOST`/`DB_PORT` per service's
own compose-mapped host port, `KAFKA_BOOTSTRAP_SERVERS` at
`host.docker.internal:29092`, and any cross-service URL (JWKS, order-
service, etc.) at `host.docker.internal:<that service's compose host port>`
instead of the in-cluster Service DNS name that `prod` uses.

KEDA itself **does** run in-cluster in dev (`bootstrap/children/dev/
infra-keda.yaml`, wave -2) — both `oms-main` and `shipment-service` ship
`ScaledObject`/`TriggerAuthentication` CRs their worker Deployments depend
on for autoscaling, so dev needs KEDA's CRDs registered regardless of
everything else running via compose. The one thing that still needs a
patch is any KEDA trigger with a *hardcoded* connection detail baked into
the CR itself rather than read from a ConfigMap/Secret — shipment-service's
Kafka trigger's `bootstrapServers` is one (see the patch in its
kustomization.yaml); check for the same pattern in each service's own
`07-scaledobject-worker.yaml` as you bring it into dev.

Any Application whose kustomization sets a Deployment's `spec.replicas`
while that same Deployment is *also* managed by an autoscaler —KEDA's
`ScaledObject` (worker Deployments) or a plain native
`HorizontalPodAutoscaler` (every service's own web-tier Deployment, e.g.
`oms-main`'s `05-hpa-web.yaml`, `shipment-service`'s `03-hpa-web.yaml`,
etc.) — needs `ignoreDifferences` on `/spec/replicas` in its bootstrap
Application for that Deployment. All 6 services already have this for both
their web and worker tiers (see any `app-*.yaml` under `bootstrap/
children/`). Without it, Argo CD's `selfHeal` fights the autoscaler,
reverting its scaling back to the git-declared value roughly every 15s (the
HPA controller's default reconcile interval) — visible as the target
Deployment flapping `Synced` → `OutOfSync` → `Synced` in `kubectl get
application <name> -n argocd -w` with nothing else actually wrong.

Each service's real `DB_USERNAME`/`DB_PASSWORD` Secret is applied
separately (see `secret.dev.example.yaml` next to shipment-service's
kustomization.yaml) — copy it, fill in the same values already in your
compose `.env`, apply the copy, never commit it (`.gitignore` already
excludes `secret.*.yaml` except the tracked `*.example.yaml` templates).

Once dev's infra tier (Vault, Kafka, Redis) actually runs in-cluster
instead of via compose, drop the `host.docker.internal` patches — at that
point dev's kustomization looks like prod's.

## Why environment-first
Component (infra vs. app) is nested *inside* environment, not the other way
around — `environments/<env>/{infra,apps}/...` — so `dev` and `prod` can
diverge in real ways (Vault HA replicas, Kafka broker count, cert-manager
replica count) without one being a thin patch on the other. See the repo
comparison this was chosen over for the trade-offs.

## Service ownership: separate repos
Each of the 6 OMS services owns its own git repo and its own `k8s/`
Kustomize base — this gitops repo never vendors their manifests. Each
environment's `environments/<env>/apps/<service>/kustomization.yaml` points
at a **remote base** (`github.com/xgiri/<service>//k8s?ref=<ref>`) and layers
env-specific patches on top (replica count, `SPRING_PROFILES_ACTIVE`,
`DB_HOST`, image tag). Service teams change their own Deployment/ConfigMap/
etc. entirely within their own repo; this repo only decides *which ref* each
environment tracks and *what overrides* apply.

| Environment | ref each service repo is pinned to |
|---|---|
| dev  | `main` — always latest |
| prod | a pinned release tag (e.g. `v1.0.0`) — bumped deliberately, never tracks a branch |

> `test` and `uat` are removed for now — re-add them the same way `dev`/`prod`
> are structured (a `root-<env>.yaml`, a `bootstrap/children/<env>/` folder,
> and an `environments/<env>/` tree) whenever they're actually needed.

## Sync wave plan (same shape in every environment)
Derived directly from the `depends_on` graph in the docker-compose you
shared — infra first, then `oms-main` (everything else needs its JWKS
endpoint), then the services that depend on it, then the gateway that needs
all of them healthy.

| Wave | Component | Why |
|---|---|---|
| -3 | infra-namespaces | Everything else needs its namespace to exist |
| -2 | infra-cert-manager, infra-vault, infra-kafka, infra-redis, infra-keda | Core infra, no interdependency between these five |
| -1 | infra-ingress-nginx, infra-observability, infra-loki | Needs cert-manager's CRDs/webhook up first; installs the Prometheus Operator CRDs every service's PodMonitor + Grafana dashboard ConfigMap depends on |
| 10 | app-oms-main | Every other service verifies JWTs against its `/.well-known/jwks.json` |
| 11 | app-product-service, app-customer-service, app-shipment-service, app-oms-bff | Each needs `oms-main` healthy (JWKS); independent of each other |
| 12 | app-oms-gateway | Routes to all 5 other services — needs every one of them healthy first |

## What this repo does NOT deploy
- **Postgres** — all 4 databases (main, product, customer, shipment) are
  external managed instances (RDS/Cloud SQL). `DB_HOST` per environment is
  patched into each service's ConfigMap (see
  `environments/<env>/apps/<service>/kustomization.yaml`) but provisioning
  those instances is a separate Terraform/IaC concern, not this repo.
- **Secrets** — `DB_PASSWORD`, `REDIS_PASSWORD`, `JWT_PRIVATE_KEY`, etc. are
  never committed here. Every service already reads them from Vault directly
  via Spring Cloud Vault (see each service's `application-<profile>.properties`
  and the `docker-compose.yml` `vault-init` step this mirrors) — the only
  thing this repo sets is `VAULT_ADDR`, pointed at the in-cluster Vault
  release for that environment.

## Adding a 7th service
1. New service gets its own repo with a `k8s/` Kustomize base (same shape as
   the 6 existing ones — `kustomization.yaml` + numbered manifests).
2. Add `environments/<env>/apps/<new-service>/kustomization.yaml` (copy an
   existing one, point `resources:` at the new repo).
3. Add `bootstrap/children/<env>/app-<new-service>.yaml` — wave 11 if it
   only depends on `oms-main`, otherwise pick the wave after whatever it
   actually depends on.
4. Repeat for both environments (`dev`, `prod`).
5. Add the new repo URL to `projects/apps-project.yaml`'s `sourceRepos`.

## Layout
```
bootstrap/
  root-{dev,prod}.yaml   <- 2 manual applies, one per environment
  children/{dev,prod}/    <- per-env Application objects, sync-wave annotated
environments/
  {dev,prod}/
    infra/
      namespaces/                  <- plain kustomize
      cert-manager/values.yaml     <- helm values, chart pulled from jetstack
      vault/values.yaml            <- helm values, chart pulled from hashicorp
      kafka/values.yaml            <- helm values, chart pulled from bitnami
      redis/values.yaml            <- helm values, chart pulled from bitnami
      ingress-nginx/values.yaml
      observability/values.yaml    <- kube-prometheus-stack (Prometheus + Grafana)
      loki/values.yaml             <- loki-stack (Loki + Promtail)
    apps/
      {oms-main,oms-gateway,oms-bff,product-service,customer-service,shipment-service}/
        kustomization.yaml         <- remote base (service's own repo) + env patches
projects/
  platform-project.yaml            <- scope for the 2 root apps
  infra-project.yaml               <- scope for infra tier
  apps-project.yaml                <- scope for app tier (namespaced only)
```

## Before this is real
- Replace every `your-registry.example.com/*`, `github.com/xgiri/*`, and
  `*.oms-rds.example.internal` placeholder with your actual values.
- `grafana.adminPassword: CHANGE_ME` in `infra/observability/values.yaml`
  needs to come from Vault/ExternalSecret, not a committed value.
- Vault's Kubernetes auth method + policies/roles for each service's
  ServiceAccount aren't Helm values — that's a one-time `vault` CLI step (or
  a Job) run after the Vault chart installs.
