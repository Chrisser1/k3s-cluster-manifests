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
    repoURL: 'https://github.com/Clusterforgers/k3s-cluster-manifests.git'
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

## Storage (Longhorn)

The cluster uses [Longhorn](https://longhorn.io) for distributed block storage. It is deployed via `argocd-apps/longhorn.yaml` and is the default storage class, new PVCs automatically use it.

Key settings (in `argocd-apps/longhorn.yaml`):
- `preUpgradeChecker.jobEnabled: false`: required for ArgoCD
- `persistence.defaultClassReplicaCount: 1`: single-node cluster; bump when more nodes join
- `storageOverProvisioningPercentage: 200`: volumes are thin-provisioned; the oracle disk is smaller than the sum of PVC sizes, so watch actual usage in the Longhorn UI

Longhorn UI (no auth, don't expose):

```bash
kubectl -n longhorn-system port-forward svc/longhorn-frontend 8080:80
```

## Moving a workload to a new node

This is the workflow for migrating e.g. the Minecraft server to a dedicated node.

### Prerequisites

- New node is joined to the cluster (via the `kubernetes-agent` NixOS module)
- Longhorn is installed and the node appears in the Longhorn UI (`kubectl -n longhorn-system port-forward svc/longhorn-frontend 8080:80`)

Pod placement and data placement are separate controls: the deployment's `nodeSelector` says where the pod runs, the Longhorn volume's node selector (via node tags) says where the data lives. For local disk speed, point both at the same node. The order below matters, the volume node selector is set *before* adding the replica, so with several candidate nodes the new replica can only land on the target.

### Steps

**1. Tag the target node in Longhorn**

Longhorn UI → Node → target node → add a unique tag (e.g. `minecraft`). Only the target node gets this tag, that's what makes the placement precise when there are multiple nodes.

**2. Set the volume's node selector to the tag**

Longhorn UI → Volume → the workload's volume → Update Node Selector → the tag from step 1. The existing replica now violates the selector; Longhorn keeps it, but any *new* replica must land on the tagged node.

**3. Scale replicas to 2**

On the same volume, set replica count to 2. The new replica's only legal home is the target node, so it syncs there. Wait until both replicas show `Healthy`.

**4. Scale replicas back to 1**

Longhorn removes the replica that violates the node selector — the old one — leaving the data solely on the target node.

**5. Move the pod**

Add a `nodeSelector` to the app's `values.yaml`:

```yaml
nodeSelector:
  kubernetes.io/hostname: <target-node-name>
```

Commit and push, ArgoCD syncs the deployment and the pod restarts on the target node with local data access.

**6. Verify**

Pod is Running on the target node, data is intact, and the volume shows a single `Healthy` replica on the target node.
