# pipeline-gitops

The deployment source of truth for the
[end-to-end CI/CD + GitOps platform](../infra/README.md).

**Whatever is committed here is what runs in the cluster.** ArgoCD reconciles
continuously, so a manual `kubectl edit` against a managed resource is reverted
within seconds. To change what is deployed, change this repository.

```
charts/sample-app/      one Helm chart, three values files
  values.yaml           shared defaults
  values-dev.yaml       written by Argo Image Updater
  values-staging.yaml   written by humans
  values-prod.yaml      written by humans
apps/root/              the three ArgoCD Applications
```

## How a change reaches production

```
ECR push  ->  values-dev.yaml     (Image Updater commits)    -> dev auto-syncs
              values-staging.yaml (promotion PR, merged)     -> staging auto-syncs
              values-prod.yaml    (promotion PR, merged)     -> prod waits for a manual sync
```

| Environment | Advances when | Sync policy |
|-------------|---------------|-------------|
| **dev** | a new image lands in ECR | automated, prune + selfHeal |
| **staging** | a promotion PR is merged | automated, prune + selfHeal |
| **prod** | a promotion PR is merged **and** someone syncs | manual |

### Why dev is different

Only `apps/root/dev-app.yaml` carries `argocd-image-updater.argoproj.io/*`
annotations. Staging and prod have none, so no automation can move them —
their tags change only through a reviewed pull request.

Image Updater is configured with `write-back-method: git`, so it commits the
new tag to `values-dev.yaml` rather than patching the live Deployment. Patching
the cluster directly would create drift that ArgoCD's `selfHeal` would revert.
Committing keeps git authoritative and leaves every deploy in `git log`.

> `image.tag` in `values-dev.yaml` is machine-written. Editing it by hand works
> until the next build overwrites it.

## Promoting a build

```bash
# 1. Take the tag dev is currently running and verified good
grep -A1 '^image:' charts/sample-app/values-dev.yaml

# 2. Put it in the next environment up, on a branch
git switch -c promote/staging-<tag>
#    edit charts/sample-app/values-staging.yaml -> image.tag
git commit -am "promote <tag> to staging" && git push -u origin HEAD

# 3. Open a PR. Merging it deploys — that review is the gate.
```

Production follows the same pattern against `values-prod.yaml`, then needs an
explicit release:

```bash
argocd app sync sample-app-prod
```

## Rolling back

Reverting the commit is the durable fix:

```bash
git revert <the-bad-commit> && git push
```

`argocd app rollback` is faster and better for a live demo, but it only changes
cluster state — git still holds the bad tag, and for dev the next Image Updater
poll can quietly reapply it. Use it to stop the bleeding, then revert in git.

## Secrets

None are stored here. `templates/externalsecret.yaml` declares *where* a value
lives (`pipeline-app/<env>` in AWS Secrets Manager); External Secrets Operator
fetches it using an IRSA role and materialises a Kubernetes Secret. This
repository only ever contains the reference.

## First-time setup

Replace the placeholders before the first sync:

| Placeholder | Where | Value |
|-------------|-------|-------|
| `<ACCOUNT_ID>` | `values.yaml`, `apps/root/dev-app.yaml` | your AWS account's ECR registry host |
| `<GITHUB_USER>` | `apps/root/*.yaml` | the account owning this repo |
| `REPLACE_ME` | each `values-*.yaml` | any tag already in ECR; dev self-corrects |

```bash
terraform -chdir=../infra/envs/cluster output -raw ecr_repository_url
kubectl apply -f apps/root/
```
