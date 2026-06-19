# platform-apps

Business application registry and Helm values for the [platform-engineering-lab](https://github.com/platform-engineer-lab) hub-and-spoke Argo CD setup.

Modelled after [marqeta/argo-cd-app-registry](https://github.com/marqeta/argo-cd-app-registry) with the following intentional differences:

- **Single git-file generator** instead of SCM Provider + region matrix — lab has two named spokes (`dev`, `prod`), no regions, no org of app repos to scan.
- **Destination by `name`** (`dev` / `prod`) instead of computed cluster names — stable across k3d restarts.
- **AppProject + ApplicationSet created imperatively by bootstrap** (consistent with the existing lab convention) rather than synced from a control repo.

## How it works

```
platform-control-plane/scripts/bootstrap.sh
  ├── creates AppProject "business-apps"  (scoped: dev + prod destinations)
  └── creates ApplicationSet "cd-apps"   (reads registry/*.yaml × envs)
            ↓  one Application per (service × env)
        podinfo-dev   → destination.name: dev   namespace: podinfo
        podinfo-prod  → destination.name: prod  namespace: podinfo
```

The `cd-apps` ApplicationSet uses a **matrix of two generators**:

1. **Git file generator** — reads every `registry/*.yaml`; each file yields top-level params (`name`, `chart`, `chartRepoURL`, `chartVersion`, `namespace`, `environments`).
2. **List generator** — expands `.environments` into one element per env, yielding `env` + `valueFiles`.

The matrix cross-product → one Argo CD Application per (service × env). Multi-source Helm: upstream chart source + `ref: values` source pointing at this repo so `valueFiles` paths resolve with `$values/`.

## Repository layout

```
registry/
  <service>.yaml          one file per app — consumed by the cd-apps ApplicationSet

apps/
  <service>/
    default-values.yaml   base Helm values shared across all envs
    dev-values.yaml       dev-specific overrides
    prod-values.yaml      prod-specific overrides
```

## Registration file schema

```yaml
name: <service>                          # used in Application name (<name>-<env>)
chartRepoURL: https://...                # Helm chart repository URL
chart: <chart-name>
chartVersion: <semver>                   # pin explicitly
namespace: <target-namespace>
environments:
  - env: dev
    valueFiles:
      - apps/<service>/default-values.yaml
      - apps/<service>/dev-values.yaml
  - env: prod
    valueFiles:
      - apps/<service>/default-values.yaml
      - apps/<service>/prod-values.yaml
```

## Adding a new application

1. Create `registry/<service>.yaml` with the schema above.
2. Create `apps/<service>/default-values.yaml`, `dev-values.yaml`, and `prod-values.yaml`.
3. Push — the `cd-apps` ApplicationSet discovers the new registration on next sync and generates `<service>-dev` and `<service>-prod` Applications automatically.
4. Verify: `kubectl --context k3d-management get applications -n argocd`.

## Env → cluster mapping

| `env` value in registry | Argo CD `destination.name` | k3d cluster |
|---|---|---|
| `dev`  | `dev`  | `k3d-dev`  |
| `prod` | `prod` | `k3d-prod` |
