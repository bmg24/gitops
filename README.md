# gitops

ArgoCD control repo (app-of-apps) for the observability playground.
This repo is the **source of truth** for what runs on the GKE cluster.

## Layout

```
gitops/
├── bootstrap/
│   └── root-app.yaml          # app-of-apps: syncs everything in apps/
├── projects/
│   └── playground.yaml        # ArgoCD AppProject (scopes repos + destinations)
├── apps/                      # one ArgoCD Application per workload
│   ├── dynatrace-operator.yaml  # sync-wave 0  — Helm chart from public.ecr.aws (v1.8.1)
│   ├── dynatrace-dynakube.yaml  # sync-wave 5  — DynaKube CRs (this repo)
│   └── easytrade.yaml           # sync-wave 10 — points at easytrade-config repo
├── docs/
│   └── sdlc-events.md         # SDLC / pipeline-observability design + DQL
├── .github/actions/
│   └── dt-sdlc-event/         # reusable composite action: POST SDLC_EVENT to Dynatrace
└── clusters/
    └── gke-playground/
        ├── setup.md           # one-time manual cluster + ArgoCD bootstrap + SDLC setup
        └── dynatrace/
            ├── kustomization.yaml
            ├── dynakube.yaml         # DynaKube CRs (NO secret)
            ├── secret.example.yaml   # template — create the real one by hand
            └── .gitignore            # blocks the real secret from being committed
```

## Before you push

1. Set your GitHub username/org everywhere `bmg24` appears — run the helper
   from the `repos/` folder: `./set-github-org.sh <your-github-username>`
   (covers `bootstrap/`, `projects/`, `apps/`, and the `easytrade-config` workflow).
2. `apps/dynatrace-operator.yaml` `targetRevision` is pinned to the latest GA
   chart (**1.8.1**); bump deliberately from
   https://github.com/Dynatrace/dynatrace-operator/releases
3. **Never commit the Dynatrace token Secret.** It is created manually in
   `clusters/gke-playground/setup.md`. The `.gitignore` blocks `secret.yaml`.
4. **SDLC events:** add `DT_URL` + `DT_SDLC_TOKEN` secrets to the `easytrade-config`
   repo. See [`docs/sdlc-events.md`](docs/sdlc-events.md).

## Bootstrap

See [`clusters/gke-playground/setup.md`](clusters/gke-playground/setup.md).

## Sync ordering

ArgoCD sync-waves guarantee order on first apply:

| Wave | App | Why |
|---|---|---|
| 0 | dynatrace-operator | Installs CRDs (incl. DynaKube) + admission webhook |
| 5 | dynatrace-dynakube | Needs the DynaKube CRD + webhook to exist first |
| 10 | easytrade | App workload; OneAgent injects once DynaKube is ready |

If the dynakube app errors on the very first sync ("no matches for kind DynaKube"
or webhook not ready), just re-sync it — the CRD/webhook needed a moment to come up.
