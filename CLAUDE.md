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

`platform-apps` is the business application registry. The `cd-apps` ApplicationSet (created by `platform-control-plane/scripts/bootstrap.sh`) reads `registry/*.yaml` and generates one Argo CD Application per (service × env).

- `registry/<service>.yaml` — declares the app: chart source, version, namespace, and environments array.
- `apps/<service>/` — Helm values files referenced by the ApplicationSet (`default-values.yaml`, `dev-values.yaml`, `prod-values.yaml`).

The ApplicationSet uses a matrix generator: git-file generator over `registry/*.yaml` × list generator expanding each file's `environments` array. Multi-source Helm: upstream chart + `ref: values` source pointing at this repo so value file paths resolve with `$values/`.

## Key conventions

- **Never use `destination.server`** — always `destination.name` (`dev` or `prod`).
- **Adding a new app**: push `registry/<svc>.yaml` + `apps/<svc>/` values — the ApplicationSet discovers it on next sync, no root manifests to update.
- **`$values` prefix** in `valueFiles` paths refers to the `ref: values` source; paths are relative to the repo root.
