# gitops — environment-first structure

## Bootstrap (one-time, manual, once per environment/cluster)
```
argocd cluster add <dev-context>  --name dev
argocd cluster add <prod-context> --name prod

kubectl apply -f bootstrap/root-dev.yaml  --context <dev-context>
kubectl apply -f bootstrap/root-prod.yaml --context <prod-context>
```
Each `root-<env>` Application then manages everything under
`bootstrap/children/<env>/` for that environment/cluster only.

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
| -2 | infra-cert-manager, infra-vault, infra-kafka, infra-redis | Core infra, no interdependency between these four |
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
