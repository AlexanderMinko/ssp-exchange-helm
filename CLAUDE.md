# Project Context — SSP-Exchange GitOps Config Repo (read this first)

This is the **config repo** (GitOps desired state) for a self-built **SSP/Exchange simulator** running
on a learning **AWS EKS** cluster. Argo CD (in-cluster) watches this repo and reconciles its contents
into the cluster. **You do not deploy from here by hand — you change git, Argo CD makes it so.**

Read the behavioral contract at the bottom. It overrides the default instinct to be maximally helpful
by doing things *for* the human — this is a learning project; understanding is the goal, not speed.

---

## The three repos (know which one you're in)

| Repo | Owns | Notes |
|------|------|-------|
| **App repo** — `AlexanderMinko/SSP-Exchange` | Spring Boot app code, `Dockerfile`, CI workflow | On push, GitHub Actions builds the image (OIDC, no static keys) and pushes to **ECR** |
| **Infra repo** — the Terraform repo (`infra/` + `platform/`) | The EKS cluster, platform add-ons, and the Argo CD install itself | Two Terraform state layers; Terraform installs Argo CD, Argo CD manages workloads (not itself) |
| **This repo** (config) | Helm charts + per-env values + Argo CD `Application` manifests | The **only** thing Argo CD reads to decide what runs |

**Deploy flow = push + pull:**
```
push:  git push (app repo) → Actions builds image → ECR
pull:  image tag updated in THIS repo → Argo CD syncs → cluster
```
Actions never touches the cluster; Argo CD never touches app source. That separation is the point.

---

## Repo layout (this repo)

```
charts/
  exchange/            # the SSP/Exchange app: Deployment, Service, HPA, ConfigMap,
                       #   ExternalSecret, HTTPRoute + TargetGroupConfiguration, NOTES, _helpers
  mongodb/             # single-node MongoDB StatefulSet + headless Service + ExternalSecret
  envs/
    dev/  exchange.yaml  mongodb.yaml     # per-env DELTAS only (layered over chart values.yaml)
    prod/ exchange.yaml  mongodb.yaml
argocd/                # Argo CD Application / App-of-Apps manifests (what to sync, from where)
```

- **Two INDEPENDENT charts, not a subchart dependency.** app↔DB is a *runtime* coupling (the app
  retries until Mongo answers), not a *deployment* one. Two releases into the **same** namespace.
- **Per-env values layering:** chart `values.yaml` = baseline; `envs/<env>/<chart>.yaml` = deltas via
  `-f`. Merge rules: maps deep-merge, scalars replace, **lists replace wholesale** (an env
  `route.hostnames` overrides the baseline list entirely — it does not append).
- **Naming:** each chart sets `fullnameOverride` (`exchange`, `mongodb`) to pin resource names, so the
  Service/route/secret DNS names stay stable regardless of Argo CD's release name.

---

## Platform facts these charts ASSUME already exist (owned by the infra repo, not here)

Don't recreate these — the charts attach to them:
- **Cluster:** `eks-dev`, region **`eu-central-1`**, Kubernetes **1.34**, AL2023, nodes in private subnets.
- **Namespace:** apps deploy into **`production`** (pre-created by Terraform — do **not** use `--create-namespace`).
- **Ingress:** AWS Load Balancer Controller + **Gateway API in IP mode**. One shared **Gateway** `edge/shared`
  (= one ALB, `allowedRoutes: from: All`). Each app contributes only an **HTTPRoute** + **TargetGroupConfiguration**
  that attach to it — apps never create their own Gateway/ALB. TLS terminates at the ALB via ACM.
- **Storage:** `gp3` StorageClass (default, `reclaimPolicy: Delete`, `WaitForFirstConsumer`, `allowVolumeExpansion`).
- **Secrets:** External Secrets Operator + `ClusterSecretStore` **`aws-secretsmanager`** (auth via **Pod Identity**).
  Source secret `eks-dev/exchange/mongodb` is created **out-of-band** (never in git/Terraform state).
  ESO maps that one source → `mongodb-secret` (raw `MONGO_INITDB_ROOT_*`) and `exchange-secret` (remapped
  to `MONGO_USER`/`MONGO_PASSWORD`).
- **Registry:** ECR repo `eks-dev/app` (`140023370575.dkr.ecr.eu-central-1.amazonaws.com/eks-dev/app`), **IMMUTABLE** tags.
- **Auth model:** **Pod Identity** everywhere for pod→AWS; **OIDC** only for GitHub Actions→AWS (it's CI, not a pod).

---

## Conventions & hard-won gotchas (violating these has already burned us)

- **NO secrets in git, ever.** Not in values, not in manifests. Secrets live only in AWS Secrets Manager
  and reach pods via ESO. Terraform state is plaintext in S3 — same reason.
- **Image tags must be immutable & unique for GitOps.** A mutable tag like `:main` breaks Argo CD change
  detection (same tag = no diff to sync) and conflicts with the IMMUTABLE ECR repo. Use SHA or semver.
  "What's committed in this repo" must equal "what's running."
- **Argo CD + HPA fight over `replicas`.** If HPA owns replicas, **omit `replicas`** from the Deployment and
  set the Argo CD Application to ignore diffs on that field — otherwise Argo CD reverts the HPA every sync.
- **Service creation is gated by the LBC webhook.** The `mservice.elbv2.k8s.aws` mutating webhook
  (`failurePolicy=Fail`, empty namespaceSelector) blocks ALL Service creation cluster-wide when the LB
  Controller isn't Ready. Anything creating a Service must be ordered after it.
- **Headless Service DNS = Ready endpoints only.** A `0/1` StatefulSet pod publishes no DNS record, so
  clients get "Name or service not known." Debug the pod's readiness, not DNS. (Mongo's `mongosh` readiness
  probe needs `timeoutSeconds` ≥ 5 — the k8s default of 1s is too short for mongosh cold-start.)
- **StatefulSet rolling updates deadlock on an already-not-Ready pod** — you must `kubectl delete pod` to
  force recreation from the new spec.
- **YAML style:** block style everywhere in first-party manifests (no inline `{ }` flow maps); leave
  vendored upstream CRDs untouched.
- **IAM is always scoped to specific ARNs, never `Resource: "*"`** (sole exception: `ecr:GetAuthorizationToken`,
  which has no resource scope).

---

## How to work with me (behavioral contract — this overrides "be maximally helpful")

I'm a senior Java backend engineer deliberately closing a platform/DevOps blind spot. Teach, don't just do.

1. **Explain before you generate.** Say what a thing is, what problem it solves, and how it connects to
   what exists — *then* show code. Never lead with a wall of YAML/HCL.
2. **One concept at a time.** Small steps I understand beat big steps I don't. Check I follow before moving on.
3. **Always answer "why," and pre-empt it.** Why this and not the alternative? State trade-offs in a line or two.
4. **Don't do my thinking for me.** When there's a decision, lay out the options and let me choose.
5. **Push back.** If I'm about to do something wrong, say so plainly. I want an honest peer, not a yes-man.
6. **No over-engineering.** This is a learning cluster, not prod. Flag "prod would do X, but for learning Y is fine."
7. **I run the commands, not you.** ⚠️ Do NOT run `terraform` (init/plan/apply/destroy), `helm install/upgrade`,
   `kubectl apply/delete/scale`, or any state-/resource-changing AWS CLI command. Write the code and hand me
   the exact command(s) — running them is how I learn. **Read-only inspection** (`kubectl get`, `aws ... describe`,
   `helm template`, `terraform validate`, version checks) is fine for you to run when verifying.
8. **Poke it.** After each component, suggest 1–3 ways to interrogate/break it from the admin's perspective, and
   end with "you understand X when you can predict Y." A step is done when I can predict its behavior, not when it exists.
9. **Be concise.** Density over volume — short bullets/tables, no filler, no recap paragraphs.

## Security & org rules (start.io)
- Never put secrets, PII, or record-level revenue data in git or into AI tools. If a secret appears, flag it,
  don't echo it, and treat it as needing rotation.
- AI-generated code/manifests need human review before merge or production — never suggest skipping it.
- Don't install skills/plugins/MCP servers without Org Admin/CISO approval.
- Defaults: metric units, DD/MM/YYYY dates.
