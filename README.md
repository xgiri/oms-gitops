# gitops repo — app-of-apps layout

## Bootstrap (one-time, manual)
```
kubectl apply -f bootstrap/root-app.yaml
```
Everything else is then managed by Argo CD automatically.

## Sync wave plan
| Wave | Component                          | Why                                   |
|------|-------------------------------------|----------------------------------------|
| -2   | infra-namespaces                    | namespaces must exist before anything else |
| -1   | infra-cert-manager                  | CRDs + controller other infra may depend on |
|  0   | infra-ingress-nginx, infra-external-secrets | rest of platform infra, no interdependency |
| 10   | app-payments, app-notifications     | app tier — starts only once all infra waves are Healthy |
| 11   | app-checkout                        | depends on payments being up first    |

Argo CD processes waves in ascending order and will not start wave N+1
until every Application/resource in wave N reports `Healthy`. Add or
remove `argocd.argoproj.io/sync-wave` annotations in
`bootstrap/children/*.yaml` to change this ordering.

## Adding a new infra component
1. Add `infra/base/<name>/` + `infra/overlays/prod/<name>/`.
2. Add `bootstrap/children/infra-<name>.yaml`, project: `infra`, pick a wave <= 0.

## Adding a new app
1. Add `apps/base/<name>/` + `apps/overlays/prod/<name>/`.
2. Add `bootstrap/children/app-<name>.yaml`, project: `apps`, wave >= 10.
3. Add the namespace to `infra/base/namespaces/namespaces.yaml`.
4. Add the destination namespace to `projects/apps-project.yaml`.

## Layout
```
bootstrap/
  root-app.yaml        <- apply manually, the only manual step
  children/             <- one Application per infra component / app
infra/
  base/<component>/     <- kustomize base (upstream manifests, pinned versions)
  overlays/prod/<component>/
apps/
  base/<service>/        <- Deployment + Service
  overlays/prod/<service>/  <- replicas, image tag per env
projects/
  platform-project.yaml  <- scope for root app (Application objects only)
  infra-project.yaml     <- scope for infra tier (cluster-wide access)
  apps-project.yaml      <- scope for app tier (namespaced only, no cluster resources)
```
