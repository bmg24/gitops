# GKE playground — one-time bootstrap

Manual provisioning (no Terraform). Run top to bottom. Keep this file updated if
you change flags — it's your rebuild recipe.

Project: `sales-engineering-noram` · Account: `brian.goins@dynatrace.com`

## 0. Prereqs (local / Cloud Shell)

```bash
gcloud components install gke-gcloud-auth-plugin   # needed for kubectl auth
gcloud config set project sales-engineering-noram
```

## 1. Create the GKE cluster

Sized for the DynaKube in this repo: the `kubernetes-monitoring` ActiveGate
requests **10Gi** memory and there are **3× routing** ActiveGates at 4Gi each,
plus the OneAgent DaemonSet, CSI driver, and easytrade. A 3–4 node
`e2-standard-4` pool (16Gi/node) handles it; autoscaling gives headroom.

```bash
gcloud container clusters create easytrade-playground \
  --zone=us-central1-a \
  --release-channel=regular \
  --num-nodes=4 \
  --machine-type=e2-standard-4 \
  --enable-autoscaling --min-nodes=3 --max-nodes=7 \
  --workload-pool=sales-engineering-noram.svc.id.goog
```

Notes:
- Zonal (single `--zone`) keeps cost down; `--region` would triple node count.
- `--workload-pool` enables Workload Identity now so it's ready later (it does
  NOT require `setIamPolicy` — that's only needed when you *bind* a GCP SA).
- Today's images all come from **public** registries, so no Artifact Registry
  IAM is needed. Your `roles/editor` + `container.admin` are sufficient.

Get credentials:

```bash
gcloud container clusters get-credentials easytrade-playground --zone=us-central1-a
kubectl get nodes
```

## 2. Create the Dynatrace token Secret (NOT in Git)

⚠️ Rotate both tokens in the tenant first — the originals were shared in
plaintext. Then create the Secret directly in the cluster:

```bash
kubectl create namespace dynatrace
kubectl -n dynatrace create secret generic brian-goins-playground \
  --from-literal=apiToken='dt0s16.<ROTATED_API_TOKEN>' \
  --from-literal=dataIngestToken='dt0s16.<ROTATED_DATA_INGEST_TOKEN>'
```

The operator (DynaKube `spec.tokens: brian-goins-playground`) reads this Secret.
It is referenced by name only and never lives in the repo.

## 3. Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl -n argocd rollout status deploy/argocd-server
```

Get the initial admin password and open the UI:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo
kubectl -n argocd port-forward svc/argocd-server 8080:443
# browse https://localhost:8080  (user: admin)
```

## 4. Bootstrap the app-of-apps

Make sure you've pushed this repo to GitHub and replaced `bmg24`
everywhere (see repo README). Then:

```bash
kubectl apply -f projects/playground.yaml
kubectl apply -f bootstrap/root-app.yaml
```

ArgoCD now syncs in waves: operator (0) → DynaKube (5) → easytrade (10).

## 5. Verify

```bash
kubectl -n dynatrace get pods                 # operator, CSI, ActiveGates, OneAgent
kubectl -n dynatrace get dynakube
kubectl -n easytrade get pods
kubectl -n easytrade get svc frontendreverseproxy   # EXTERNAL-IP = the app URL
```

In the tenant (`anr47691.sprint.dynatracelabs.com`):
- Kubernetes app → cluster `easytrade-playground` reporting
- Hosts/Services → easytrade services with OneAgent traces + business events

## 6. SDLC / pipeline observability (GitHub Actions)

This sends **normalized `SDLC_EVENT`s** (pipeline run, build, deployment, change)
to Dynatrace every time `easytrade-config` changes — independent of the cluster,
so you can wire it up before or after the cluster is live.

1. **Create the SDLC ingest token** in the tenant
   (Access Tokens → Generate). Scope: **`openpipeline.events_sdlc`**.
   This is a *different* token from the OneAgent api/dataIngest Secret above.

2. **Add repo (or org) secrets** on the `easytrade-config` repo
   (Settings → Secrets and variables → Actions):

   | Secret | Value |
   |---|---|
   | `DT_URL` | `https://anr47691.sprint.dynatracelabs.com`  (no `/api`, no trailing slash) |
   | `DT_SDLC_TOKEN` | the `openpipeline.events_sdlc` token from step 1 |

3. The reusable sender action lives in **this** (`gitops`) repo at
   `.github/actions/dt-sdlc-event`. The `easytrade-config` workflow
   (`.github/workflows/sdlc.yml`) references it as
   `<your-org>/gitops/.github/actions/dt-sdlc-event@main`, so `gitops` must be
   pushed and `set-github-org.sh` must have run.

4. **Trigger it**: merge any change to `easytrade-config` `main`
   (or run the workflow manually via *Actions → Run workflow*). It emits, in order:
   pipeline `run` started → `build` started/finished → `deployment`
   started/finished → pipeline `run` finished. Pull requests emit `change` events.

5. **Verify in the tenant** (Notebooks → DQL):

   ```dql
   fetch events, from: now()-1h
   | filter event.kind == "SDLC_EVENT"
   | filter event.provider == "github.com"
   | sort timestamp desc
   | fields timestamp, event.type, event.status, cicd.pipeline.run.id, artifact.version
   ```

See [`../../docs/sdlc-events.md`](../../docs/sdlc-events.md) for the full design,
field mapping, and how to extend it to new repos (e.g. the future `ai-app`).

## Troubleshooting

- **dynakube app fails on first sync** ("no matches for kind DynaKube" / webhook
  not ready): the CRD/webhook needed a moment. Re-sync the `dynatrace-dynakube`
  app (retry is configured, so it usually self-heals).
- **ActiveGate pod Pending**: not enough memory on any node for the 10Gi request.
  Bump `--machine-type` (e.g. `e2-standard-8`) or lower the AG memory in dynakube.yaml.
- **easytrade pods CrashLoop on first boot**: DB init ordering — give it a minute
  or restart the failing deployment.

## Teardown

```bash
gcloud container clusters delete easytrade-playground --zone=us-central1-a
```
