# RomM

Self-hosted ROM library with in-browser play (EmulatorJS), reachable at
<https://romm.mircoporetti.me> through the existing Cloudflare tunnel.

ROMs live on the OpenMediaVault NAS and are mounted read-only over SMB. RomM's
own data (database, artwork, saves) lives on Longhorn.

## Layout

| File | What it does |
|---|---|
| `namespace.yaml` | The `romm` namespace |
| `secret.yaml` | **Template only.** Documents key names; real values created by hand (below) |
| `pv-library-smb.yaml` | Static PV for the NAS `Roms` share, read-only |
| `pvc.yaml` | The SMB claim plus five Longhorn claims |
| `mariadb.yaml` | MariaDB 11.4 StatefulSet + Service |
| `deployment.yaml` | RomM itself |
| `service.yaml` | ClusterIP on 8080 (the tunnel reaches this) |

Depends on `infrastructure/csi-driver-smb` (the SMB CSI driver) and an ingress
entry in `infrastructure/cloudflare/configmap.yaml`.

## Secrets

Not in git. **These two commands must be re-run before RomM will start on a
rebuilt cluster.** Prefix each with a space to keep it out of shell history.

```bash
 kubectl create secret generic romm-smbcreds \
  --namespace romm \
  --from-literal=username='<OMV_USER>' \
  --from-literal=password='<OMV_PASSWORD>'

 kubectl create secret generic romm-secrets \
  --namespace romm \
  --from-literal=ROMM_AUTH_SECRET_KEY="$(openssl rand -hex 32)" \
  --from-literal=DB_PASSWD='<DB_PASSWORD>' \
  --from-literal=MARIADB_ROOT_PASSWORD='<DB_ROOT_PASSWORD>'
```

Key names are load-bearing: the SMB driver requires exactly `username` and
`password`. Changing `ROMM_AUTH_SECRET_KEY` invalidates all existing sessions.

## Deploy order

```bash
helmfile -f ../../infrastructure/csi-driver-smb/helmfile.yaml apply  # once per cluster
kubectl apply -f namespace.yaml
# ...create the two secrets...
kubectl apply -f pv-library-smb.yaml -f pvc.yaml
kubectl apply -f mariadb.yaml
kubectl apply -f deployment.yaml -f service.yaml
```

## The NAS side

OMV exports a dedicated shared folder so the `romm` account can't reach the rest
of the NAS:

- Shared folder `Roms` -> relative path `PoraniPaghetti_Share/Mirco/Games/Roms/`
- SMB share name `Roms`, **read only**, Public = No
- User `romm` with read-only privileges on that folder only

Read-only is enforced in three places: the OMV privilege, the `ro` CIFS mount
option, and `readOnly` on the pod's volumeMount. RomM can scan and launch games
but never modify the library.

Platform folders inside `Roms` must be **lowercase RomM slugs**. Matching is
case-sensitive: a `GBA` folder still scans, but silently fails to match IGDB, so
you lose platform metadata and icons. Current folders:

```
3ds  gb  gba  genesis  n64  nds  nes  ngc  psp  psx  snes  switch
```

## config.yml

Lives on the `romm-config` Longhorn volume, so it must be written after the pod
is running:

```bash
kubectl -n romm exec deploy/romm -c romm -- sh -c 'cat > /romm/config/config.yml <<EOF
filesystem:
  skip_hash_calculation: true
exclude:
  roms:
    single_file:
      names: [".DS_Store"]
      extensions: ["txt", "md"]
EOF'
kubectl -n romm rollout restart deploy/romm
```

## Keeping the NAS disks asleep

The NAS spins its disks down when idle, and several settings exist to respect
that:

| Setting | Why |
|---|---|
| `ENABLE_RESCAN_ON_FILESYSTEM_CHANGE=false` | inotify can't see remote CIFS changes; its polling fallback would stat the whole tree forever and keep the disks awake |
| `ENABLE_SYNC_FOLDER_WATCHER=false` | Same reason |
| `skip_hash_calculation: true` | Hashing reads every ROM end-to-end on each scan |
| `ENABLE_SCHEDULED_RESCAN` + `0 3 * * *` | One spin-up per night instead. Drop to `0 3 * * 0` for weekly, or set `false` to only ever scan manually from the UI |

Artwork, saves, and the database are all on Longhorn, so browsing the library
costs no NAS I/O. Only launching a game reads from the NAS.

New ROMs copied to the NAS appear after the nightly scan, or immediately via the
Scan button in the UI.

## Kubernetes-specific gotchas

Things that took a fix, worth not rediscovering:

- **`enableServiceLinks: false`** is mandatory. Kubernetes injects Service
  addresses as env vars, RomM's nginx template picks them up and tries to bind
  them, and the container won't start.
- **`IPV4_ONLY=true`** — this cluster is single-stack IPv4. Without it nginx also
  tries to bind `[::]:8080` and fails.
- **`mariadb-admin ping` needs `--skip-ssl`** in the init container. MariaDB 11.4
  serves TLS with a self-signed cert and its 11.4 client verifies certs by
  default, so a plain ping fails with a TLS error even against a healthy server.
- **longhorn-sc formats XFS**, which refuses volumes under 300Mi. That's the
  floor for the small PVCs.
- **The container runs as root** by design (`/init` starts the embedded valkey and
  the nginx master). Only the nginx workers drop to uid 1000.

## Failover

RomM prefers `kreacher` (16GB, better for scans) and falls back to `dobby`;
`dumbledore` is excluded as an arm64 Pi with no Longhorn disk. Longhorn keeps
replicas on both workers, so the data is already present either way.

Both pods have 60s `not-ready`/`unreachable` tolerations instead of the 5-minute
default. Longhorn's `nodeDownPodDeletionPolicy` is set to
`delete-both-statefulset-and-deployment-pod` (in
`infrastructure/longhorn/helmfile.yaml`) — without it a StatefulSet never
recovers from a hard node failure on its own, because Kubernetes won't recreate
the ordinal while the original pod is stuck `Terminating` on an unreachable node.

## Access

Two gates: Cloudflare Access on the hostname, then RomM's own login. The tunnel
also provides the HTTPS that EmulatorJS requires for PSP titles (the `ppsspp`
core needs `SharedArrayBuffer`, which browsers only expose in a
cross-origin-isolated secure context).

Cloudflare free tier caps uploads at 100MB, so adding large ROMs through the
browser will fail — copy to the NAS instead. Note also that streaming large
volumes of ROM data through the CDN sits awkwardly with Cloudflare's terms
(section 2.8); for heavy use, a MetalLB LoadBalancer on the LAN is the
alternative.
