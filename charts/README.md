# Helm charts

- **`exchange/`** — the SSP/Exchange app + edge (`Gateway`/`HTTPRoute`) + `exchange-secret`.
- **`mongodb/`** — single-node MongoDB + `mongodb-secret`.

**Two independent releases** (not a subchart). They're deployed separately because the app handles
its DB dependency at runtime (retries until Mongo answers) — there's no *deployment* coupling, only
a runtime one. Install both **into the same namespace** so the app resolves the `mongodb` Service.
Order doesn't matter.

## Environments (values layering)

Chart baselines (`<chart>/values.yaml`) hold environment-agnostic defaults. Per-env **deltas** live
under `envs/<env>/<chart>.yaml` and are layered on top with `-f` (later `-f` wins; maps deep-merge,
scalars replace, **lists replace wholesale**). Each chart is its own release, so each needs its own
env file — hence the chart×env matrix:

```
charts/
  exchange/values.yaml      mongodb/values.yaml     # baselines (in-chart defaults)
  envs/
    dev/  exchange.yaml  mongodb.yaml               # dev deltas only
    prod/ exchange.yaml  mongodb.yaml               # prod deltas only
```

"Everything about dev" is one folder; `diff charts/envs/dev charts/envs/prod` shows env drift.
This maps 1:1 onto ArgoCD later — one `Application` per env pointing its `valueFiles` at `envs/<env>/`.

## Deploy
```sh
ENV=dev   # or prod

# render-check first
helm template mongodb  charts/mongodb  -n production -f charts/envs/$ENV/mongodb.yaml  | less
helm template exchange charts/exchange -n production -f charts/envs/$ENV/exchange.yaml | less

# install (two releases, same namespace)
# the `production` namespace is pre-created by platform/ (kubernetes_namespace_v1) — no --create-namespace
helm upgrade --install mongodb  charts/mongodb  -n production -f charts/envs/$ENV/mongodb.yaml
helm upgrade --install exchange charts/exchange -n production -f charts/envs/$ENV/exchange.yaml
```
Each chart keeps its own `externalSecrets` values (both point at the same `ClusterSecretStore` +
source secret `eks-dev/exchange/mongodb`) — a small, explicit duplication that's the cost of decoupling.

## Prereqs (NOT in these charts — shared cluster infra, managed in `platform/`)
ESO operator + `ClusterSecretStore aws-secretsmanager` · AWS secret `eks-dev/exchange/mongodb` · `gp3` StorageClass · LB Controller + `alb` GatewayClass.
