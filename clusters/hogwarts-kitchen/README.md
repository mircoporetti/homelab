# hogwarts-kitchen (Flux root)

Flux syncs this folder. Each app folder registers the app's own repo
(GitRepository), applies its `k8s/app` (Kustomization) and bumps its image
when CI pushes a new `<timestamp>-<sha>` tag (Image automation).

## Adding an app

Copy `gazzetta/`, rename, list it in `kustomization.yaml`, and add the
hostname to `infrastructure/cloudflare/configmap.yaml`.

## One-time move from the hangman repo

Flux was bootstrapped from `mircoporetti/hangman` (`k8s/flux`). To move it here:

```bash
export GITHUB_TOKEN=<token with repo admin on homelab>
flux bootstrap github --owner=mircoporetti --repository=homelab \
  --branch=main --path=clusters/hogwarts-kitchen --personal
```

The hangman objects keep the same names, so nothing is recreated. Once
`flux get kustomizations` is all green, delete `k8s/flux` from the hangman repo.
