# k3s-cluster-manifests

Helm charts and ArgoCD application manifests for the home k3s cluster.

## Structure

```
apps/           # One Helm chart per application
argocd-apps/    # ArgoCD Application manifests (auto-discovered via app-of-apps)
```

`argocd-apps/apps.yaml` is the root app-of-apps, any new file added to `argocd-apps/` is automatically picked up by ArgoCD.

## Adding a new application

### 1. Create the Helm chart

```
apps/myapp/
  Chart.yaml
  values.yaml
  templates/
    ...
```

### 2. Add an ArgoCD Application manifest

Create `argocd-apps/myapp.yaml` (copy from an existing one, e.g. `gymbros.yaml`):

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
spec:
  project: default
  source:
    repoURL: 'https://github.com/Chrisser1/k3s-cluster-manifests.git'
    targetRevision: HEAD
    path: apps/myapp
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: myapp
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

Commit and push, ArgoCD will deploy it automatically.

### 3. Set up CI/CD (if the app has its own GitHub repo)

Add a `.github/workflows/deploy.yml` to the app repo that:
1. Builds and pushes Docker images to the internal registry (`REGISTRY_HOST:30500`)
2. Checks out this repo and bumps the image tag in `apps/myapp/values.yaml`
3. Commits and pushes

Copy `.github/workflows/deploy.yml` from the GymBros repo as a starting point.

**GitHub Actions secrets/variables to configure** in the app repo:

| Name | Type | Value | Notes |
|------|------|-------|-------|
| `REGISTRY_HOST` | Variable | `100.81.239.16` | Tailscale IP of the control plane |
| `CLUSTER_MANIFESTS_TOKEN` | Secret | Classic PAT | Needs `repo` scope on `k3s-cluster-manifests`. |

**GitHub Actions runner:**

The self-hosted runner on the oracle node handles all builds. It is registered via `services.github-runners.gymbros` in the NixOS config using a **classic PAT** with `repo` scope stored in `secrets.nix` as `githubRunnerToken`. If adding a new application with a repo, then remember to edit the token to have access to that repository.

> Fine-grained PATs cannot register GitHub Actions runners regardless of permissions, use a classic PAT.
