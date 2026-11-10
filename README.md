# 🚀 Reusable GitHub Workflow: Helm Chart Deployment with ArgoCD — *Open Source*

This reusable workflow renders a [Helm](https://helm.sh) chart per environment with `helm template`, commits the rendered manifests to your **continuous deployment (CD) repo**, and uses the **ArgoCD REST API** to create, sync and validate the application.

It is the Helm counterpart of [gitopsmanager/k8s-deploy](https://github.com/gitopsmanager/k8s-deploy), which does the same with Kustomize. Both use the same environment map, CD repo layout and ArgoCD handling, so the two can deploy side by side and [k8s-restore-cluster](https://github.com/gitopsmanager/k8s-restore-cluster) restores apps from either.

🧩 **Open Source:** part of the [**GitOps Manager™**](https://gitopsmanager.io) open-source toolchain.

> ✅ **First-time deployments are supported** – ArgoCD apps are auto-created if missing.
> ⚠️ **Self-hosted runner required** – must have network access to ArgoCD.

---

## 🧭 How it works

```
client repo (chart + values)  ──helm template──▶  CD repo (plain YAML)  ──ArgoCD──▶  cluster
        inputs                                     what is deployed
```

Helm is used as a **build tool only**. Nothing is `helm install`ed, so there is no Helm release in the cluster:

| Helm feature | Replaced by |
|---|---|
| `helm rollback` | `git revert` of the rendered commit in the CD repo, or redeploy a previous `ref` + `image_tag` |
| `helm history` | `git log` of `<cluster>/<namespace>/<app>` in the CD repo, with full diffs |
| `helm diff` | The CD repo diff, plus ArgoCD's live diff |
| *(none)* | **Drift detection**: ArgoCD flags manual changes in the cluster |
| `helm test` | ArgoCD health and sync status, which the workflow waits for |

For an emergency rollback, revert the CD repo commit: it restores exactly what ran before. Redeploying an old `ref` re-renders it with today's environment map and workflow.

---

## 📁 Chart layout

`deploy_path` is the chart folder itself, a standard Helm chart:

```
charts/myapp/                 ← deploy_path
  Chart.yaml
  values.yaml                 ← chart defaults, always loaded
  values-dev.yaml             ← optional, applied when target_environment = dev
  values-prod.yaml            ← optional
  values-prod-cluster-a.yaml  ← optional, applied when prod deploys to cluster-a
  templates/
```

Values are layered in this order, later wins:

1. `values.yaml`
2. `values-<target_environment>.yaml`, if present
3. `values-<env>-<cluster>.yaml`, if present, for the cluster resolved from `env_map`
4. Image `--set` values from the workflow inputs

Write a cluster file only when one cluster genuinely differs from the rest of its environment.

---

## 🖼️ Images

The container registry differs per cluster and comes from `env_map`, so the chart cannot know it. For each image listed, the workflow sets:

```
--set-string <key>.repository=<container_registry>/<image>
--set-string <key>.tag=<image_tag>        # only when image_tag is given
```

The key defaults to `image`, which is what `helm create` and most charts use. For a chart with several images, give each image its key with `name=key`:

```yaml
image_base_name: team/myapp                        # → image.repository / image.tag
image_base_names: team/sidecar=sidecar.image       # → sidecar.image.repository / .tag
```

When `image_tag` is empty, the tag comes from the chart's values or its `appVersion`.

### Promoting to production

Keep the chart in your repo and tag images with the commit SHA. A release is then one identifier:

```
dev:   ref=abc123  image_tag=sha-abc123
qa:    ref=abc123  image_tag=sha-abc123    ← promoted
prod:  ref=abc123  image_tag=sha-abc123    ← same chart, same image
```

An image build never changes the chart. The chart changes only when the Kubernetes templates change.

---

## 📦 Third-party charts

To deploy a published chart (ingress-nginx, Redis, …) with your own values, make `deploy_path` a small **wrapper chart** that depends on it. This is standard Helm, nothing workflow-specific:

```yaml
# charts/ingress/Chart.yaml
apiVersion: v2
name: ingress
version: 0.1.0
dependencies:
  - name: ingress-nginx
    version: 4.11.3                                   # pin it
    repository: https://kubernetes.github.io/ingress-nginx
```

```yaml
# charts/ingress/values-prod.yaml — nested under the dependency's name
ingress-nginx:
  controller:
    replicaCount: 2
```

The workflow registers the dependency's repository and fetches it before rendering. `https://`, `oci://` and `file://` repositories all work. Commit a `Chart.lock` (`helm dependency update` locally) to pin the exact chart digest.

Things to know about third-party charts:

- **Usually leave the image inputs empty.** The chart already points at its publisher's registry. Set them only if you mirror the images into your own registry, and use the nested key, e.g. `bitnami/redis=redis.image`. Charts that split `registry` and `repository` (such as Bitnami) need their own values instead.
- **Public repositories only, for now.** Private chart repositories need credentials the workflow does not yet take.
- **Charts that create their own Secret.** Many charts generate a password or render a Secret from values, which the workflow rejects (see [Secrets](#-secrets)). Use the chart's `existingSecret` (or similarly named) option and let a secrets operator create that Secret.

---

## 🔐 Secrets

**Secrets are never rendered or committed by this workflow.** The rendered manifests go to the CD repo in plain text, so secret values must reach the cluster another way:

- **Your application manages its secrets separately.** Nothing secret is in the chart, and nothing needs to be done.
- **You use a secrets operator.** The chart renders the operator's resource, which holds only a reference or ciphertext, and the operator creates the real `Secret` in the cluster.

| Rendered resource | Contains | Allowed |
|---|---|---|
| `ExternalSecret` ([External Secrets Operator](https://external-secrets.io)) | A reference to the value in AWS Secrets Manager, Azure Key Vault, etc. | ✅ |
| `SecretProviderClass` ([Secrets Store CSI Driver](https://secrets-store-csi-driver.sigs.k8s.io)) | A reference | ✅ |
| `SealedSecret` ([Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets)) | `encryptedData`, ciphertext only | ✅ |
| `Secret` with no `data` / `stringData` | Nothing sensitive | ✅ |
| `Secret` with `data` or `stringData` | The secret value itself | ❌ the run fails |

The check is absolute; there is no override. If a rendered `Secret` with values is found, the run fails before anything is committed and names the Secret. To fix it:

1. Remove the value from your values files.
2. Create the secret through your operator (for example an `ExternalSecret` in the chart's templates).
3. Point the chart at the resulting Secret by name, typically with its `existingSecret` option.

Example, using External Secrets with a chart that supports `existingSecret`:

```yaml
# templates/externalsecret.yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: myapp-db
spec:
  secretStoreRef:
    kind: ClusterSecretStore
    name: aws-secrets-manager
  target:
    name: myapp-db              # the Secret the operator creates
  data:
    - secretKey: password
      remoteRef:
        key: prod/myapp/db
        property: password
```

```yaml
# values.yaml
database:
  existingSecret: myapp-db
```

---

## ⚙️ Inputs

| Input | Required | Description |
|---|---|---|
| `github_runner` | ✅ | Runner name, or a JSON array of runner labels |
| `namespace` | ✅ | Kubernetes namespace. Also passed to `helm template --namespace` |
| `target_environment` | ✅ | Environment in `env_map`. Selects `values-<env>.yaml` |
| `target_cluster` | when env has >1 cluster | Cluster in `env_map`. Selects `values-<env>-<cluster>.yaml` |
| `target_cloud` | | `aws` or `azure` when the env spans both clouds |
| `ref` | | Git ref to check out. For a chart in the repo, this is the chart version |
| `env_map` | | Environment map JSON. Falls back to the runner's `ENV_MAP` |
| `cd_repo`, `cd_repo_org` | ✅ | The CD repo |
| `github_environment` | | GitHub environment, e.g. for production approval |
| `application` | single-app | App name. Also the Helm release name |
| `deploy_path` | single-app | Chart folder (contains `Chart.yaml`). Default `Deployments` |
| `image_base_name` | | Main image, optionally `name=key` |
| `image_base_names` | | Further images, comma-separated, each optionally `name=key` |
| `application_details` | multi-app | JSON `[{name, images[], path}]`; `path` is each app's chart folder |
| `image_tag` | | Tag for every listed image. Empty → tag from the chart |
| `helm_version` | | Helm version. Default `v3.17.0` |
| `delete_first` | | Delete the ArgoCD app before redeploying |
| `delete_only` | | Delete the ArgoCD app without redeploying |
| `argocd_auth_token`, `argocd_username`, `argocd_password` | | ArgoCD credentials |
| `skip_status_check`, `insecure_argo`, `debug` | | As in k8s-deploy |

### Secrets

| Secret | Required | Description |
|---|---|---|
| `CONTINUOUS_DEPLOYMENT_GH_APP_ID` | ✅ | GitHub App used to write to the CD repo |
| `CONTINUOUS_DEPLOYMENT_GH_APP_PRIVATE_KEY` | ✅ | Its private key |
| `ARGOCD_CA_CERT`, `ARGOCD_USERNAME`, `ARGOCD_PASSWORD` | | ArgoCD connection |

### Differences from k8s-deploy

| k8s-deploy | k8s-helm-deploy |
|---|---|
| `overlay_dir` (required) | Dropped. `target_environment` is used as the overlay dir in the CD repo |
| `kustomize_version` | Replaced by `helm_version` |
| Images patched with `kustomize edit set image` | Images set with `--set-string <key>.repository/.tag` |
| Nunjucks placeholders filled from `env_map` | Per-env and per-cluster values files |

Everything else has the same meaning.

---

## 🧪 Example

```yaml
jobs:
  deploy-dev:
    uses: gitopsmanager/k8s-helm-deploy/.github/workflows/deploy.yaml@v1
    with:
      github_runner: arc-runner-set
      application: myapp
      deploy_path: charts/myapp
      image_base_name: team/myapp
      image_tag: sha-${{ github.sha }}
      namespace: myapp
      target_environment: dev
      cd_repo: continuous-deployment
      cd_repo_org: acme
    secrets: inherit
```

---

## 📂 CD repo output

```
<cluster>/<namespace>/<app>/
  create.json                   ← read by k8s-restore-cluster
  overlays/<target_environment>/
    manifests.yaml              ← helm template output
```

The ArgoCD application `<namespace>-<app>` points at `overlays/<target_environment>`. The rendered manifests are also uploaded as a run artifact.

---

## 📜 License

MIT. See [LICENSE](LICENSE) and [THIRD-PARTY-ACTIONS-AND-TOOLS.md](THIRD-PARTY-ACTIONS-AND-TOOLS.md).
