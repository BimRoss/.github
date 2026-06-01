# BimRoss Release Recipe

Canonical reference for shipping any BimRoss app onto the admin Kubernetes cluster. Same recipe whether you're driving from Warp on the laptop or talking to Ross/Joanne in Slack.

If something in this doc conflicts with what you find in a repo, **this doc wins**. Open a PR here first; fix the drift in the repo second.

## The mental model

1. App repos build container images and push to Docker Hub on tag pushes.
2. App repos also open a PR on `bimross/rancher-admin` that bumps the relevant `admin/apps/<app>/*.yaml` to point at the new image tag.
3. Rancher Fleet watches `rancher-admin/master`. When that PR merges, Fleet rolls the cluster.

So the path from "code merged" to "env deployed" is **two PRs in two repos**, not one. Skipping the second step means the image is published but no env picks it up.

## The canonical workflow

All five active app repos use the same reusable workflow:

```yaml
# .github/workflows/<repo>-images.yml in each app repo
gitops-release:
  if: startsWith(github.ref, 'refs/tags/v')   # or your routing
  needs: [<build-job-name>]
  uses: BimRoss/.github/.github/workflows/gitops-release.yml@v1
  secrets: inherit
  with:
    app_slug: <repo-slug>
    version: <bare semver, no v>
    target_env: ""           # or "dev" / "prod" for the ross/joanne split
    images: |
      geeemoney/<image-one>
      geeemoney/<image-two>  # if applicable
    manifest_paths: |
      admin/apps/<app>/<file>.yaml
    commit_subject: "release(<app>): bump to <version>"
```

The reusable workflow handles: sanitize + validate version, verify the tag(s) exist on Docker Hub, patch manifests, open the rancher-admin PR, best-effort auto-merge. See `.github/workflows/gitops-release.yml` for the full implementation.

Pin via `@v1` (movable major). For a hotfix in flight, pin a SHA temporarily.

## Per-repo profiles

| Repo | Envs | Image(s) | Manifest path(s) |
|---|---|---|---|
| `claude-code-ross` | dev + prod | `geeemoney/claude-code-ross` | `admin/apps/claude-code-ross{,-prod}/deployment.yaml` |
| `claude-code-joanne` | dev + prod | `geeemoney/claude-code-joanne` | `admin/apps/claude-code-joanne{,-prod}/deployment.yaml` |
| `makeacompany-ai` | single | `geeemoney/makeacompany-ai-frontend`, `geeemoney/makeacompany-ai-backend` | `admin/apps/makeacompany-ai/{frontend,backend}.yaml` |
| `job-tracker` | single | `geeemoney/job-tracker-api`, `geeemoney/job-tracker-web` | `admin/apps/job-tracker/{api,web}.yaml` |
| `grantfoster.dev` | single | `geeemoney/grantfoster-website` | `admin/apps/grantfoster-website/deployment.yaml` |
| `dating-venue` | single | `geeemoney/dating-venue` | `admin/apps/dating-venue/deployment.yaml` |

## How to ship — dev/prod split (ross, joanne)

### Bump dev

1. Merge code PR to repo `main`.
2. Tag: `git tag -a v0.X.Y -m "<summary>" && git push origin v0.X.Y`
3. CI builds the image and opens an auto-PR on `rancher-admin` titled `release(<app>): bump dev to 0.X.Y`.
4. Merge that PR (`gh pr merge --squash --delete-branch`; if blocked by approvals, `--admin --squash`).
5. Fleet syncs within ~30s; Recreate rollout takes ~90s of Slack-quiet time. Wait it out before declaring success.

### Promote dev → prod

1. Confirm `v0.X.Y` already shipped to dev and behaves.
2. Trigger the dispatch:
   ```bash
   gh workflow run <app>-images.yml \
     --repo BimRoss/<app> \
     -f promote_to_prod=0.X.Y       # bare semver, NO v prefix
   ```
3. Merge the auto-PR on rancher-admin (same merge dance as above).
4. Fleet rolls prod (~90s outage). Confirm with `kubectl -n <app>-prod get pods`.

**Critical:** `promote_to_prod` is **bare semver**. The reusable workflow strips a leading `v` defensively, but the convention is "no v in the input." Typing `v0.X.Y` historically produced `ImagePullBackOff` (see rancher-admin#367 / claude-code-ross#209).

## How to ship — single-env (makeacompany-ai, job-tracker, grantfoster.dev)

1. Merge code PR to repo main.
2. Tag: `git tag -a v0.X.Y -m "..." && git push origin v0.X.Y`
3. CI builds image(s) and opens auto-PR on rancher-admin.
4. Merge it. Fleet rolls prod.

There is no separate "promote to prod" step — the tag is the release.

## Verification

After Fleet rolls:

```bash
# Did the deployment image actually update?
KUBECONFIG=~/.kube/config/admin.yaml \
  kubectl -n <app>[-prod] get deploy <app> \
  -o jsonpath='{.spec.template.spec.containers[*].image}'

# Are pods Running, not ImagePullBackOff or CrashLoopBackOff?
KUBECONFIG=~/.kube/config/admin.yaml kubectl -n <app>[-prod] get pods
```

If you see `ImagePullBackOff`: the deployed image tag and Docker Hub disagree. Most common cause is the `v`-vs-no-`v` mismatch above; check the tag string on the deployment vs. the Docker Hub tags list (`curl https://hub.docker.com/v2/repositories/<image>/tags/<version>`).

If you see `CrashLoopBackOff`: most often an env-contract change — code expects a variable the manifest/Secret doesn't supply. See `MEMORY.md` entries on the ross env contract for the pattern.

## Rollback

Rolling back is "open a PR on rancher-admin that reverts the image tag." Same recipe as forward.

```bash
cd ~/dev/rancher-admin
git checkout -b fix/<app>-rollback-<version>
# Hand-edit the relevant deployment.yaml, change the tag back
git add admin/apps/<app>/... && git commit -m "fix(<app>): roll back to <last-good>"
git push -u origin fix/<app>-rollback-<version>
gh pr create --title "..." --body "..."
gh pr merge <pr> --admin --squash --delete-branch  # if you need it merged now
```

Never `kubectl edit` the cluster directly — Fleet reconciles from `rancher-admin/master` and will revert your edit within ~30s.

## Auto-merge — flaky, treat as best-effort

Per learned lessons (`feedback_gitops_release_workflow`), `gh pr merge --auto --squash` arms the PR but doesn't always stick — race conditions with required status checks, self-approval blocks, and inconsistent branch-protection rules all play in. The reusable workflow arms it and exits 0 regardless.

If the PR is still open after the workflow finishes:

```bash
gh pr merge <pr> --squash --delete-branch         # try first
gh pr merge <pr> --admin --squash --delete-branch # if approval rule blocks
```

Don't re-arm `--auto` from your own shell — same self-approval block, silently no-ops.

## Don't do these things

- **Don't `git push` directly to `rancher-admin/master`.** Branch protection rejects it. Always PR.
- **Don't `kubectl edit` deployments in admin clusters.** Fleet reverts within ~30s.
- **Don't `--no-verify` past failing hooks.** Investigate the failure.
- **Don't type `v0.X.Y` into the `promote_to_prod` workflow_dispatch input.** Bare semver.
- **Don't copy the reusable workflow into a new repo.** `uses:` it. Drift is the whole problem we're solving.
- **Don't infer dev/prod split for new repos.** Check the workflow file. job-tracker, makeacompany-ai, and grantfoster.dev are single-env.

## Migration status

This recipe assumes all five active repos have migrated to the reusable workflow. If a repo's workflow file still has an inline `gitops-release` job, it's pre-migration. Until it migrates:

- The recipe above still describes what *will* happen, because the inline jobs were hardened (BimRoss/claude-code-ross#209, BimRoss/claude-code-joanne#92, BimRoss/makeacompany-ai#118, BimRoss/job-tracker#7, BimRoss/grantfoster.dev#9) to behave identically to the reusable workflow.
- But future fixes to the reusable workflow will not reach unmigrated repos. Migrate opportunistically on their next routine release.

Order of migration (lowest blast radius first):
1. grantfoster.dev (canary)
2. job-tracker
3. makeacompany-ai
4. claude-code-joanne
5. claude-code-ross
