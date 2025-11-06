# Auto-Commit-Squash-Merge — *Open Source* & Stable `@v1`

**Automates GitHub commits, pull requests, squash merges, and cleanup.**  
Commits and pushes changes to a temporary branch, opens a pull request, retries squash-merge with fixed and random backoff, then deletes the branch.  
Generic version usable on any repository.


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
| `namespace` | Optional contextual label (e.g., environment or namespace) | ✅ Yes | — |
| `cluster` | Optional contextual label (e.g., cluster or system name) | ✅ Yes | — |
| `app_name` | Application or logical name for commit message | ✅ Yes | — |
| `repo_path_rel` | Relative path to the files being committed | ✅ Yes | — |
| `delete_only` | If true, performs delete-only commit and merge path | ❌ No | `false` |

---

## 📤 Outputs

| Name | Description |
|------|--------------|
| `branch` | Temporary branch created by this action |

---

## 🧠 Usage Example

```yaml
jobs:
  auto-commit:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Auto Commit and Squash Merge
        uses: ./.github/actions/auto-commit-squash-merge
        with:
          token: ${{ secrets.GH_APP_TOKEN }}
          repo_owner: affinity7software
          repo_name: k8s-config
          namespace: prod
          cluster: aks-prod-weu
          app_name: frontend
          repo_path_rel: clusters/prod/frontend
```

---

## ⚙️ Behavior Summary

| Phase | Description |
|--------|--------------|
| **Commit & Push** | Creates a new temporary branch and commits relevant changes |
| **Pull Request** | Opens a pull request targeting the repository’s default branch |
| **Retry Merge** | Attempts squash-merge with fixed (2s) and random (1–7.5s) backoff |
| **Cleanup** | Deletes temporary branch (skips protected branches like `main`/`master`) |

---

## 🧱 Use Cases

This action is used to support automated configuration or application updates in CI/CD pipelines, such as:
- Automatically pushing generated Kubernetes manifests or overlays.
- Triggering downstream ArgoCD or GitOps reconciliations after deployments.

---

## 📦 Internal Logic Summary

- Performs empty commits when no file changes are detected (ensures workflow continuity).  
- Uses the GitHub REST API for pull request creation and squash-merge attempts.  
- Handles merge conflicts gracefully and retries transient errors automatically.  
- Supports both GitHub App tokens and standard PAT tokens.

---

## 🔢 Versioning Policy — Official Release

Starting with this release, all `v1` versions follow the same **stable tagging model** used across GitOps Manager™ Actions.

| Tag | Moves When | Purpose |
|------|-------------|----------|
| **`v1`** | Any new release in the `v1.x.x` series | Always points to the latest stable release (no breaking changes). |
| **`v1.3`** | New patch within that feature line (e.g. `v1.3.5 → v1.3.6`) | Tracks bug fixes and improvements only — no new required inputs. |
| **`v1.3.7`** | Never | Fully immutable, reproducible snapshot. |

All tags will now **increment forward permanently** — no re-use or re-tagging of old versions. 

---

## 📄 License

MIT License © Affinity7 Consulting Ltd
