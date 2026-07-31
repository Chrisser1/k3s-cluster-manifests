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

**If the app gets a `*.hivemindcloud.dk` Ingress host**, also add it to `apps/coredns-custom/values.yaml`. Internal cluster DNS for that whole zone is overridden (to point at the ingress ClusterIP instead of the public IP, avoiding hairpin NAT) — any hostname not explicitly listed there resolves fine externally but returns nothing inside the cluster. This silently breaks anything that resolves its own hostname from inside a pod, most notably cert-manager's HTTP-01 self-check, which will hang in `Issuing certificate as Secret does not exist` indefinitely with no error until the hostname is added and CoreDNS is restarted (`kubectl rollout restart deployment -n kube-system coredns`).

### 3. Set up CI/CD (if the app has its own GitHub repo)

Add a `.github/workflows/deploy.yml` to the app repo that:
1. Builds and pushes Docker images to the internal registry (`REGISTRY_HOST:30500`)
2. Checks out this repo and bumps the image tag in `apps/myapp/values.yaml`
3. Commits and pushes

Copy `.github/workflows/deploy.yml` from the GymBros repo as a starting point.

**GitHub Actions secrets/variables to configure** in the app repo:

| Name | Type | Value | Notes |
|------|------|-------|-------|
| `REGISTRY_HOST` | Variable | `100.96.184.94` | Tailscale IP of the control plane |
| `CLUSTER_MANIFESTS_TOKEN` | Secret | Classic PAT | Needs `repo` scope on `k3s-cluster-manifests`. |

**GitHub Actions runner:**

There's no self-hosted runner managed by this repo or the cluster's NixOS config anymore — builds reach the internal registry over Tailscale instead. Configure the runner in the app repo's own CI setup.

## Family photos (Nextcloud + Immich)

Both apps read the same files on one shared RWX volume. See
[docs/family-photos.md](docs/family-photos.md) for the folder layout, how
per-family isolation is enforced, and the procedure for adding a new person.

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

The Longhorn UI's per-volume "Operation" menu (the one you get from checking a volume in the list and clicking the dropdown, or from a single volume's detail page) does **not** include an "Update Node Selector" action in this version, even though it has an entry for nearly every other volume field (replica count, data locality, anti-affinity, etc.). Node selector isn't UI-editable here — set it via `kubectl` instead:

```bash
kubectl patch volumes.longhorn.io -n longhorn-system <volume-name> \
  --type=merge -p '{"spec":{"nodeSelector":["<tag-from-step-1>"]}}'
```

`<volume-name>` is the `pvc-...` name shown as the PVC's `VOLUME` in `kubectl get pvc -n <namespace>`, or find it in the Longhorn UI's Volume list. The existing replica now violates the selector; Longhorn keeps it running, but any *new* replica must land on the tagged node.

**3. Scale replicas to 2**

In the Longhorn UI, this one *is* a real dropdown action ("Update Replicas Count"), or via `kubectl`:

```bash
kubectl patch volumes.longhorn.io -n longhorn-system <volume-name> \
  --type=merge -p '{"spec":{"numberOfReplicas":2}}'
```

The new replica's only legal home is the target node, so it syncs there. Wait until the volume's `status.robustness` is `healthy` again (check via `kubectl get volumes.longhorn.io -n longhorn-system <volume-name> -o jsonpath='{.status.robustness}'`, or the UI's volume list — it'll show `Degraded` mid-rebuild, then `Healthy`).

**4. Scale replicas back to 1 — then manually remove the stale replica**

```bash
kubectl patch volumes.longhorn.io -n longhorn-system <volume-name> \
  --type=merge -p '{"spec":{"numberOfReplicas":1}}'
```

Longhorn does *not* promptly remove the replica that violates the node selector on its own (observed >3 minutes with no change). List the replicas and delete the one **not** on the target node yourself:

```bash
kubectl get replicas.longhorn.io -n longhorn-system -l longhornvolume=<volume-name> \
  -o custom-columns=NAME:.metadata.name,NODE:.spec.nodeID,STATE:.status.currentState

kubectl delete replicas.longhorn.io -n longhorn-system <replica-name-on-old-node>
```

This is safe once step 3 confirmed `healthy` — that means the new replica is a complete, in-sync copy (verify via `kubectl get engines.longhorn.io -n longhorn-system <volume-name>-e-0 -o jsonpath='{.status.replicaModeMap}'`, both should show `RW`) before deleting the old one. In the Longhorn UI, the same removal is available from the volume's detail page → Replicas tab → the trash icon next to the specific replica.

**5. Move the pod**

Add a `nodeSelector` to the app's `values.yaml`:

```yaml
nodeSelector:
  kubernetes.io/hostname: <target-node-name>
```

Commit and push, ArgoCD syncs the deployment and the pod restarts on the target node with local data access.

**6. Verify**

Pod is Running on the target node, data is intact, and the volume shows a single `Healthy` replica on the target node.
