# ARK: Survival Evolved server

One password-gated Extinction server, pinned to `queen`, deployed from
`apps/ark/` via the [SickHub `ark-cluster`](https://github.com/SickHub/ark-server-charts)
chart. The image (`drpsychick/arkserver`) runs
[`arkmanager`](https://github.com/arkmanager/ark-server-tools), which installs
and updates the game from Steam itself — the game files are not baked into the
image.

The chart is built for a *cluster* of maps sharing one set of game files. We run
a single map today; adding another is a new entry under `servers:` in
`values.yaml` (see [Adding a second map](#adding-a-second-map)).

## Connecting

Players connect to **`ark.hivemindcloud.dk`**, an A record pointing at the same
public IP as the rest of the zone (`82.211.219.198`). The most reliable method is
the in-game console (`Tab`), which does resolve hostnames:

```
open ark.hivemindcloud.dk:7777
```

The server also advertises itself in ARK's unofficial server browser as
**"Hivemind Extinction"**; either way players are prompted for the join password.

Two caveats worth knowing:

- **Do not add `ark.hivemindcloud.dk` to `apps/coredns-custom/values.yaml`.** The
  README's rule about that file applies to *Ingress* hosts. ARK isn't behind the
  ingress, and the CoreDNS override resolves every listed name to the ingress
  ClusterIP — which would point ARK players inside the cluster at the wrong
  service entirely. ARK never resolves its own hostname, so there's nothing to
  fix by listing it.
- **Players on the home LAN** resolve the public IP and depend on the router
  doing hairpin NAT. If it doesn't, they connect to `192.168.50.187:7777`
  directly.

## Networking

ARK tells the client which port to connect on, so the port players use must be
**identical** to the container port. There is no NodePort or LoadBalancer
remapping possible — the chart uses `hostPort`, binding the pod's ports directly
onto `queen`.

| Port | Proto | Purpose | Exposure |
|------|-------|---------|----------|
| 7777 | UDP | Game | Public |
| 7778 | UDP | Game (engine uses `game+1`) | Public |
| 27015 | UDP | Steam query / server browser | Public |
| 32330 | TCP | RCON | **Tailnet only** |

The NixOS side lives in `servers/modules/queen/configuration.nix`
(`networking.firewall.allowedUDPPorts`). RCON is deliberately *not* there: its
password is also ARK's in-game admin password, so anyone with it can cheat.
`tailscale0` is a trusted interface, so RCON stays reachable over the tailnet
without being exposed to the internet.

Router port-forwarding of UDP 7777, 7778 and 27015 to `192.168.50.187` is a
manual step — nothing in this repo can do it.

## Passwords

**This repo is public on GitHub.** A password in `values.yaml` would be readable
by anyone, which would make the join gate pointless. Both passwords therefore
live in a hand-created Secret that is never committed — the same pattern
`gymbros-secrets` / `nextcloud-credentials` / `collabora-secrets` use — and are
injected via `extraEnvVars` + `secretKeyRef`:

| Secret key | ARK env var | What it does |
|------------|-------------|--------------|
| `server-password` | `am_ark_ServerPassword` | Required to join. Share this one. |
| `admin-password` | `am_ark_ServerAdminPassword` | RCON + in-game admin (cheats). **Never share.** |

Create or rotate them with:

```bash
kubectl -n ark create secret generic ark-secrets \
  --from-literal=server-password='...' \
  --from-literal=admin-password='...' \
  --dry-run=client -o yaml | kubectl apply -f -

kubectl -n ark rollout restart deploy/ark-extinction
```

The chart only emits its own `am_ark_ServerPassword` / `am_ark_ServerAdminPassword`
when `servers.<name>.password` / `rcon.password` are set in values. Both are
deliberately left unset — setting either would produce a duplicate env key
alongside the `secretKeyRef` versions.

## Storage

ARK lives on `/dev/sdb`, a **dedicated 366 GB disk** on queen mounted at
`/mnt/games` and registered with Longhorn under the disk tag `games`. This keeps
game data off both the 3.6 TB `/mnt/data` array and the OS disk. (The disk
previously held an old AMP game-panel install; it was reformatted 2026-07-31.)

`apps/ark/templates/storageclass.yaml` defines `longhorn-queen-games`, which
pins replicas to that disk (`nodeSelector: queen`, `diskSelector: games`,
1 replica, `Retain`).

| PVC | Size | Contents |
|-----|------|----------|
| `ark-game` | 60Gi | Steam server files + installed mods. Re-downloadable. |
| `ark-extinction` | 50Gi | **The save game and configs.** This is the one to back up. |
| `ark-cluster` | 1Gi | Cross-map transfer data (uploaded characters/dinos). |

`storageClass` must stay set on all three. The chart renders
`storageClassName: ""` when it's unset, which disables dynamic provisioning
outright and leaves the PVCs `Pending` with no obvious error.

### Registering the disk with Longhorn

Longhorn does not pick up new mounts on its own — the node CR has to be patched
(this is how `/mnt/data` was added too). One-time, already done:

```bash
kubectl -n longhorn-system patch nodes.longhorn.io queen --type=merge -p '{
  "spec": {"disks": {"mnt-games": {
    "path": "/mnt/games/", "allowScheduling": true, "diskType": "filesystem",
    "storageReserved": 10737418240, "tags": ["games"]
  }}}}'
```

If the disk ever shows `Unschedulable` in the Longhorn UI, the usual cause is
the mount not being present when the Longhorn manager pod started — restart
`longhorn-manager` on queen after confirming `/mnt/games` is mounted.

## First boot takes a long time

On the very first start the pod downloads ~25 GB of server files from Steam
before ARK ever starts listening. The chart's default startup budget is 12
minutes, which expires mid-download and puts the pod into a restart loop, so
`startupProbe` is widened to ~3 hours in `values.yaml`.

Watch progress with:

```bash
kubectl -n ark logs -f deploy/ark-extinction
```

The server is joinable once the startup probe passes (`kubectl -n ark get pod`
shows `1/1 Running`). Until then the pod sits at `0/1 Running`, which is
expected, not a failure.

## Operating

```bash
# status
kubectl -n ark get pod,pvc

# stop the server (the chart's own default is replicas: 0)
kubectl -n ark scale deploy/ark-extinction --replicas=0

# start it again
kubectl -n ark scale deploy/ark-extinction --replicas=1

# RCON / arkmanager from inside the pod
kubectl -n ark exec -it deploy/ark-extinction -- arkmanager rconcmd "ListPlayers"
kubectl -n ark exec -it deploy/ark-extinction -- arkmanager status
kubectl -n ark exec -it deploy/ark-extinction -- arkmanager saveworld
```

ArgoCD has `selfHeal: true`, so a manual `scale` is reverted on the next sync.
To stop the server durably, set `replicaCount: 0` in `values.yaml` and push.

`updateOnStart: true` means the server checks Steam for a game update on every
restart, so restarting after an ARK patch is enough to update it.

## Game settings

`Game.ini` and `GameUserSettings.ini` are rendered from
`servers.extinction.customConfigMap` in `values.yaml` (rates, multipliers,
difficulty). Changing them changes the ConfigMap checksum, which restarts the
server automatically. Note that the chart's `xpMultiplier` shortcut only applies
if you *don't* supply your own `GameUserSettingsIni` — once you do, it has to be
set inside that block.

## Mods

Add Steam Workshop IDs as strings under `mods:` in `values.yaml`. They're
downloaded by whichever server has `updateOnStart: true` (ours), into the shared
`ark-game` volume. Large modpacks may need `persistence.game.size` raised.

## Adding a second map

```yaml
servers:
  extinction:
    updateOnStart: true   # keep this on exactly one server
    # ...
  ragnarok:
    map: Ragnarok
    sessionName: "Hivemind Ragnarok"
    ports:                # must not collide with Extinction's
      gameudp: 7779
      queryudp: 27017
      rcon: 32331
```

Three things change when a second map exists:

1. The new ports need opening in `queen/configuration.nix` and forwarding on the
   router.
2. `ark-game` and `ark-cluster` are mounted by both pods, so their `accessModes`
   must become `ReadWriteMany` — which pulls in Longhorn's share-manager.
   They're `ReadWriteOnce` today precisely because a single server doesn't need it.
3. A second DNS record (or a shared one plus different ports) for players.

Both maps share `clusterName: hivemind`, which is what lets players transfer
characters and dinos between them. Changing that value orphans anything already
uploaded.
