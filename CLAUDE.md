# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Delivery pipeline

```mermaid
flowchart LR
    subgraph github["GitHub"]
        SRC["sample-service\nsource + Dockerfile + CI"]
        CFG["sample-service-config\ndry Helm chart"]
        ADD["platform-addons\nroles/<role>/"]
        APP["platform-apps\nregistry/*.yaml"]
    end

    GHCR[("GHCR\nghcr.io/…/sample-service:<sha>")]

    subgraph mgmt["management cluster"]
        AC(["Argo CD"])
        HY["Source\nHydrator"]
        GP["gitops-\npromoter"]
    end

    DEV(["dev spoke"])
    PROD(["prod spoke"])

    SRC -->|"CI: build + push :sha"| GHCR
    SRC -->|"CI: PR bump image.tag"| CFG
    ADD -->|"App-of-Apps"| AC
    APP -->|"cd-apps ApplicationSet"| AC
    CFG -->|"dry source HEAD"| HY
    HY -->|"push env/dev-next\nenv/prod-next"| CFG
    CFG -->|"env/dev · env/prod"| AC
    GP -->|"merge env/*-next → env/*"| CFG
    AC -->|"sync"| DEV
    AC -->|"sync"| PROD
    GHCR -.->|"pull"| DEV
    GHCR -.->|"pull"| PROD
    DEV -->|"argocd-health ✓\nunlocks prod"| GP
```

## Architecture

`platform-apps` is the central registry for all business applications. It contains two discovery directories read by ApplicationSets in `platform-control-plane/scripts/bootstrap.sh`:

### `registry/` — standard Helm apps (`cd-apps` ApplicationSet)

Each `registry/<service>.yaml` declares a Helm app: chart source, version, namespace, `valuesRepoURL`, and environments. The `cd-apps` ApplicationSet uses a matrix generator (git-file over `registry/*.yaml` × list expanding `environments`) to generate one Argo CD Application per (service × env). Multi-source Helm: upstream chart + `ref: values` source pointing at `valuesRepoURL` so `$values/` paths resolve into the app's own config repo.

Helm values live in dedicated per-app config repos (e.g. `podinfo-config`) — **not** in `apps/` inside this repo.

### `promoter/` — gitops-promoter apps (`cd-promoter-config` ApplicationSet)

Each `promoter/<service>.yaml` declares a promoter-enabled app with `configRepoURL` and `configPath`. The `cd-promoter-config` ApplicationSet generates one Argo CD Application per entry, pointing at `<configRepoURL>/<configPath>/` on the management cluster. That Application syncs all app-level Argo CD Applications and gitops-promoter CRs from the app's own config repo.

## Registry file schemas

**`registry/<service>.yaml`** (standard Helm via `cd-apps`):
```yaml
name: <service>
chartRepoURL: https://...
chart: <chart-name>
chartVersion: <semver>
namespace: <target-namespace>
valuesRepoURL: https://github.com/platform-engineer-lab/<service>-config
environments:
  - env: dev
    defaultValuesFile: $values/values/default-values.yaml
    envValuesFile: $values/values/dev-values.yaml
  - env: prod
    defaultValuesFile: $values/values/default-values.yaml
    envValuesFile: $values/values/prod-values.yaml
```

**`promoter/<service>.yaml`** (gitops-promoter pipeline via `cd-promoter-config`):
```yaml
name: <service>
configRepoURL: https://github.com/platform-engineer-lab/<service>-config
configPath: config
```

## Key conventions

- **Never use `destination.server`** — always `destination.name` (`dev` or `prod`).
- **Helm values live in per-app config repos** — `apps/` no longer exists in this repo; `$values/` resolves to `valuesRepoURL`.
- **Adding a standard Helm app**: create `registry/<svc>.yaml` with `valuesRepoURL` pointing at a dedicated `<svc>-config` repo that holds the values files.
- **Adding a promoter-enabled app**: create `promoter/<svc>.yaml` pointing at the app's config repo; ensure `config/apps/` and `config/promoter/` exist there.
