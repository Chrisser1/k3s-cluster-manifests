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

The cluster uses [Longhorn](https://longhorn.io) for distributed block storage. It is deployed via `argocd-apps/longhorn.yaml` and is the default storage class, new PVCs automatically use it.

### After first install

Once Longhorn is Running, demote `local-path` so it is no longer the default:

```bash
kubectl patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
```

### Migrating an existing PVC from local-path to Longhorn

Do this once per app after Longhorn is installed (gymbros-db, minecraft, registry all started on local-path).

```bash
# 1. Pause ArgoCD auto-sync for the app
kubectl patch application <appname> -n argocd --type merge -p '{"spec":{"syncPolicy":{"automated":null}}}'

# 2. Scale down the workload
kubectl scale deployment <deployment> -n <namespace> --replicas=0

# 3. Create a new Longhorn PVC
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: <pvc-name>-longhorn
  namespace: <namespace>
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: longhorn
  resources:
    requests:
      storage: <size>
EOF

# 4. Copy data from old PVC to new
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: migrate
  namespace: <namespace>
spec:
  restartPolicy: Never
  containers:
  - name: migrate
    image: alpine
    command: ["sh", "-c", "cp -av /source/. /dest/ && echo DONE"]
    volumeMounts:
    - name: source
      mountPath: /source
    - name: dest
      mountPath: /dest
  volumes:
  - name: source
    persistentVolumeClaim:
      claimName: <pvc-name>
  - name: dest
    persistentVolumeClaim:
      claimName: <pvc-name>-longhorn
EOF

kubectl wait --for=condition=complete pod/migrate -n <namespace> --timeout=300s
kubectl delete pod migrate -n <namespace>

# 5. Delete old PVC and rename new one
kubectl delete pvc <pvc-name> -n <namespace>
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: <pvc-name>
  namespace: <namespace>
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: longhorn
  resources:
    requests:
      storage: <size>
EOF
kubectl delete pvc <pvc-name>-longhorn -n <namespace>

# 6. Scale back up and re-enable ArgoCD sync
kubectl scale deployment <deployment> -n <namespace> --replicas=1
kubectl patch application <appname> -n argocd --type merge -p '{"spec":{"syncPolicy":{"automated":{"prune":true,"selfHeal":true}}}}'
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
