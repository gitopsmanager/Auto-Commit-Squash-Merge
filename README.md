# Auto-Commit-Squash-Merge — *Open Source* & Stable `@v2`

**Automates GitHub commits, pull requests, squash merges, and cleanup.**  
Commits and pushes changes to a temporary branch, opens a pull request, retries squash-merge with fixed and random backoff, then deletes the branch.  
Generic version usable on any repository.

**`v2` takes every `v1` call unchanged** — same inputs, same defaults — and adds optional inputs for committing from any checkout, to any branch, with your own message, and for committing nothing when nothing changed. Moving from `@v1` to `@v2` needs no other change.


--- 

## 🧩 Overview

This composite action streamlines GitOps-style commits and merges for continuous deployment pipelines.  
It is used to programmatically commit and squash-merge updates in repositories — no manual review required.  
The action is **idempotent** and retries merges intelligently to handle transient GitHub API race conditions.

It is used by both **K8s-deploy** and **K8s-restore-cluster** workflows to automatically commit and merge  
ArgoCD or configuration repository changes as part of the cluster deployment and restoration lifecycle.

---

## 🌐 GitOps Manager™ Enterprise Platform

[**GitOps Manager™ Enterprise**](https://gitopsmanager.io) is the full platform that powers this open-source workflow.  
It’s a **turnkey GitOps automation platform** for AWS and Azure — combining open-source GitHub Actions, Kubernetes infrastructure automation, and global-scale CI/CD.

**Highlights:**
- Secure, opinionated **multi-cloud GitOps automation** for Kubernetes workloads.  
- Deep integration with **ArgoCD**, **Argo Workflows**, **Traefik**, **ECK**, and **Kubernetes Dashboard**.  
- Built for **high availability**, **autoscaling**, and **managed upgrades**.  
- Supports **Workload Identity**, **Pod Identity**, and **private, network-isolated clusters**.  
- Enables **global deployments**, **secret management**, and **production-grade infrastructure** with **zero vendor lock-in**.

🔗 Learn more: [https://gitopsmanager.io](https://gitopsmanager.io)

---

## 🚀 Inputs

| Name | Description | Required | Default |
|------|--------------|-----------|----------|
| `token` | GitHub App or PAT token used for commits and PRs | ✅ Yes | — |
| `repo_owner` | Owner or organization of the target repository | ✅ Yes | — |
| `repo_name` | Name of the target repository | ✅ Yes | — |
| `namespace` | Contextual label (e.g., environment or namespace), used in the branch name and default messages | ❌ No (v1: Yes) | `""` |
| `cluster` | Contextual label (e.g., cluster or system name), as above | ❌ No (v1: Yes) | `""` |
| `app_name` | Application or logical name, as above | ❌ No (v1: Yes) | `""` |
| `repo_path_rel` | Relative path to the files being committed | ✅ Yes | — |
| `delete_only` | If true, the action does nothing (kept for v1 callers) | ❌ No | `false` |
| `path` | **v2.** The checkout to commit from, relative to the workspace | ❌ No | `continuous-deployment` |
| `extra_paths` | **v2.** More paths to commit with `repo_path_rel`, relative to `path`, space-separated | ❌ No | `""` |
| `base_branch` | **v2.** The branch to merge into | ❌ No | the default branch |
| `skip_if_unchanged` | **v2.** `true`: nothing changed → no commit, no PR, `changed=false` | ❌ No | `false` (an empty commit, as v1) |
| `commit_message` | **v2.** The commit message; its first line is the squash commit's title | ❌ No | `Deploy to <cluster>/<namespace> for <app_name>` |
| `pr_title` | **v2.** The PR's title | ❌ No | the message's first line, else v1's title |
| `push_auth` | **v2.** `remote`: the checkout's credentials. `token`: the `token` input, for a checkout made with `persist-credentials: false` | ❌ No | `remote` |

---

## 📤 Outputs

| Name | Description |
|------|--------------|
| `branch` | Temporary branch created by this action |
| `changed` | **v2.** `true` when something was committed |
| `commit_sha` | **v2.** The squash commit on the base branch (the checkout's HEAD when nothing was committed) |
| `previous_sha` | **v2.** The commit the change was made on, when `repo_path_rel` already existed there |
| `pr_url` | **v2.** The pull request, merged or not |

---

## 🧠 Usage Example

```yaml
jobs:
  auto-commit:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v7

      - name: Auto Commit and Squash Merge
        uses: gitopsmanager/Auto-Commit-Squash-Merge@v2
        with:
          token: ${{ secrets.GH_APP_TOKEN }}
          repo_owner: affinity7software
          repo_name: k8s-config
          namespace: prod
          cluster: aks-prod-weu
          app_name: frontend
          repo_path_rel: clusters/prod/frontend
```

From any checkout, with your own message, committing nothing when nothing changed (v2):

```yaml
      - name: Checkout
        uses: actions/checkout@v7
        with:
          path: config
          persist-credentials: false

      - name: Commit the values file
        id: commit
        uses: gitopsmanager/Auto-Commit-Squash-Merge@v2
        with:
          token: ${{ steps.app-token.outputs.token }}
          repo_owner: client-org
          repo_name: config
          path: config
          repo_path_rel: components/nodepools
          base_branch: main
          skip_if_unchanged: "true"
          push_auth: token
          commit_message: "Node pool versions as the cloud has them"

      - run: echo "merged ${{ steps.commit.outputs.commit_sha }}"
        if: steps.commit.outputs.changed == 'true'
```

---

## ⚙️ Behavior Summary

| Phase | Description |
|--------|--------------|
| **Commit & Push** | Creates a new temporary branch and commits relevant changes |
| **Pull Request** | Opens a pull request targeting `base_branch` (default: the repository’s default branch) |
| **Retry Merge** | Attempts squash-merge with fixed (2s) and random (1–7.5s) backoff |
| **Conflict** | Fails at once when the merge error names a conflict, or when GitHub reports the PR `dirty` (conflicted) on two checks in a row. Every other state — `unknown`, `blocked`, `behind`, `unstable`, or one GitHub adds later — is retried |
| **Cleanup** | Deletes temporary branch (skips protected branches like `main`/`master`). On a failure that also closes the PR; its diff stays on it, and the branch can be restored from the PR page |

---

## 🧱 Use Cases

This action is used to support automated configuration or application updates in CI/CD pipelines, such as:
- Automatically pushing generated Kubernetes manifests or overlays.
- Triggering downstream ArgoCD or GitOps reconciliations after deployments.

---

## 📦 Internal Logic Summary

- Performs empty commits when no file changes are detected (ensures workflow continuity), unless `skip_if_unchanged` is `true`.  
- Two runs editing different lines of the same file both merge; editing the same lines, the second fails rather than overwriting the first.  
- The temporary branch's name carries the run id, so two runs can never pick the same one.  
- Uses the GitHub REST API for pull request creation and squash-merge attempts.  
- Handles merge conflicts gracefully and retries transient errors automatically.  
- Supports both GitHub App tokens and standard PAT tokens.

---

## 🔢 Versioning Policy — Official Release

All versions follow the same **stable tagging model** used across GitOps Manager™ Actions. `v1` is frozen at `v1.0.1`; new releases are `v2.x.x`.

| Tag | Moves When | Purpose |
|------|-------------|----------|
| **`v2`** | Any new release in the `v2.x.x` series | Always points to the latest stable release (no breaking changes). |
| **`v2.3`** | New patch within that feature line (e.g. `v2.3.5 → v2.3.6`) | Tracks bug fixes and improvements only — no new required inputs. |
| **`v2.3.7`** | Never | Fully immutable, reproducible snapshot. |

All tags will now **increment forward permanently** — no re-use or re-tagging of old versions. 

---

## 📄 License

MIT License © GitOps Manager, S.L.
