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

## Storage (Longhorn)

The cluster uses [Longhorn](https://longhorn.io) for distributed block storage. It is deployed via `argocd-apps/longhorn.yaml` and is the default storage class — new PVCs automatically use it. All apps were migrated off `local-path` in July 2026; `local-storage` is disabled in the k3s config.

Key settings (in `argocd-apps/longhorn.yaml`):
- `preUpgradeChecker.jobEnabled: false` — required for ArgoCD (the hook needs a service account that doesn't exist until the chart installs; chicken-and-egg)
- `persistence.defaultClassReplicaCount: 1` — single-node cluster; bump when more nodes join
- `storageOverProvisioningPercentage: 200` — volumes are thin-provisioned; the oracle disk is smaller than the sum of PVC sizes, so watch actual usage in the Longhorn UI

The NixOS side needs `services.openiscsi` enabled and a `/usr/local/bin → /run/current-system/sw/bin` symlink on every node (Longhorn nsenters into the host and expects `iscsiadm` on an FHS path) — see `modules/features/kubernetes/` in the nixos repo.

Longhorn UI (no auth, don't expose):

```bash
kubectl -n longhorn-system port-forward svc/longhorn-frontend 8080:80
```

### Backup / restore a PVC via the PC

Used for the local-path → Longhorn migration; works for any PVC move or disaster recovery.

```bash
# 1. Pause ArgoCD auto-sync — ROOT APP FIRST, otherwise the app-of-apps
#    reverts the child patches and selfHeal scales pods right back up
kubectl patch application apps -n argocd --type merge -p '{"spec":{"syncPolicy":{"automated":null}}}'
kubectl patch application <appname> -n argocd --type merge -p '{"spec":{"syncPolicy":{"automated":null}}}'

# 2. Scale down so the data is quiescent (critical for postgres)
kubectl scale deployment <deployment> -n <namespace> --replicas=0

# 3. Pull the data to the PC
#    - local-path PVs: rsync the dir shown by
#      kubectl get pv -o custom-columns=NAME:.metadata.name,PATH:.spec.local.path,CLAIM:.spec.claimRef.name
#    - Longhorn PVs: mount the PVC in a temp pod (see step 5) and
#      kubectl exec ... tar czf - -C /data . > backup.tar.gz
rsync -avz oracle-server:<pv-path>/ ~/cluster-backup/<app>/

# 4. Delete the old PVC and create the replacement (same name, same size,
#    storageClassName: longhorn) while sync is still paused

# 5. Restore: run a temp pod mounting the new PVC...
#    (image: alpine, command: ["sleep","infinity"], volumeMounts /data)
#    ...then stream the data in:
tar czf - -C ~/cluster-backup/<app> . | kubectl exec -i -n <namespace> restore -- tar xzf - -C /data

# 5b. rsync does NOT preserve ownership when run as a normal user — fix uids
#     for apps that don't run as uid 1000. postgres:16-alpine is uid 70:
kubectl exec -n <namespace> restore -- chown -R 70:70 /data

# 6. Delete the temp pod, scale up, VERIFY the app works
kubectl scale deployment <deployment> -n <namespace> --replicas=1

# 7. Re-enable sync — children first, root app last
kubectl patch application <appname> -n argocd --type merge -p '{"spec":{"syncPolicy":{"automated":{"prune":true,"selfHeal":true}}}}'
kubectl patch application apps -n argocd --type merge -p '{"spec":{"syncPolicy":{"automated":{"prune":true,"selfHeal":true}}}}'
```

## Moving a workload to a new node

This is the workflow for migrating e.g. the Minecraft server to a dedicated node.

### Prerequisites

- New node is joined to the cluster (via the `kubernetes-agent` NixOS module)
- Longhorn is installed and the node appears in the Longhorn UI (`kubectl -n longhorn-system port-forward svc/longhorn-frontend 8080:80`)

### Steps

**1. Tag nodes in Longhorn**

In the Longhorn UI → Node, add a tag to the new node (e.g. `minecraft`) and a tag to oracle (e.g. `oracle`).

**2. Scale replicas to 2 on the volume**

In the Longhorn UI → Volume, find the volume for the workload and set replica count to 2. Longhorn will start syncing to the new node. Wait until both replicas show `Healthy`.

**3. Pin the volume to the new node**

Once sync is complete, update the volume's node selector to only the new node's tag, then scale replicas back to 1. Longhorn removes the oracle replica automatically.

**4. Move the pod**

Add a `nodeSelector` to the app's `values.yaml`:

```yaml
nodeSelector:
  kubernetes.io/hostname: <new-node-name>
```

Commit and push — ArgoCD syncs the deployment and the pod restarts on the new node with local data access.

**5. Verify**

Confirm the pod is Running on the new node and data is intact, then the oracle replica is already gone (removed in step 3).
