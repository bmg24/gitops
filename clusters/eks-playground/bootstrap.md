# EKS playground: one-time bootstrap

Run top to bottom on a cluster that already meets [`cluster-contract.md`](cluster-contract.md).
How the current cluster was actually built (and how to tear it down) is in
[`BUILD_LOG.md`](BUILD_LOG.md). Keep this file updated if you change a step: it's the
rebuild recipe.

Cluster: `brian-eks-playground` · Region: `us-east-1` · Dynatrace tenant: `lbn6737h` (sprint)

## 1. Connect

Requires the AWS CLI signed in to the account that owns the cluster.

```bash
aws eks update-kubeconfig --name brian-eks-playground --region us-east-1
kubectl config current-context        # must end in cluster/brian-eks-playground
kubectl get nodes                     # expect 2 Ready nodes (scales 0 to 3)
```

## 2. Default StorageClass (gp3)

The AWS org policy denies creating **gp2** volumes, and EKS marks no StorageClass as
default on this cluster. Without this step every PVC stays `Pending`.

```bash
kubectl apply -f clusters/eks-playground/storageclass-gp3.yaml
kubectl get storageclass              # expect: gp3 (default)
```

## 3. Dynatrace token Secret (NOT in Git)

Classic **`dt0c01`** tokens from tenant `lbn6737h`: an operator token (template
"Kubernetes: Dynatrace Operator") and a data-ingest token (`metrics.ingest`,
`logs.ingest`, `openTelemetryTrace.ingest`). Run this in your own terminal, not in a
chat or anything that records output:

```bash
kubectl create namespace dynatrace --dry-run=client -o yaml | kubectl apply -f -
kubectl -n dynatrace create secret generic brian-obs-playground \
  --from-literal=apiToken='<CLASSIC_OPERATOR_TOKEN>' \
  --from-literal=dataIngestToken='<DATA_INGEST_TOKEN>'
```

The DynaKube (`spec.tokens: brian-obs-playground`) reads this Secret by name; it never
lives in the repo.

## 4. Install ArgoCD (pinned)

```bash
kubectl create namespace argocd --dry-run=client -o yaml | kubectl apply -f -
kubectl apply -n argocd --server-side --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/v3.6.1/manifests/install.yaml
kubectl -n argocd rollout status deploy/argocd-server
kubectl get crd applicationsets.argoproj.io
```

- `--server-side` is required: the ApplicationSet CRD exceeds the client-side apply
  annotation limit.
- v3.6.x is the only ArgoCD line tested on Kubernetes 1.37. Avoid 3.4.x below 3.4.10
  (October 2026 critical advisories). ArgoCD 3.5+ bundles Helm 4.

UI access (run in your own terminal so the password stays private):

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d; echo
kubectl -n argocd port-forward svc/argocd-server 8080:443
# browse https://localhost:8080 (user: admin), change the password, then:
kubectl -n argocd delete secret argocd-initial-admin-secret
```

## 5. Bootstrap the app-of-apps

ArgoCD reads `main` of `https://github.com/bmg24/gitops.git`, so push your changes
first. Then, from the repo root:

```bash
kubectl apply -f projects/playground.yaml    # one-time seed; thereafter managed by apps/projects.yaml
kubectl apply -f bootstrap/root-app.yaml
```

The first `projects/playground.yaml` apply is a one-time seed so the `root` and `projects` apps
have a project to belong to. **After bootstrap, the AppProject is GitOps-managed** by
`apps/projects.yaml` (sync-wave -5): edit `projects/playground.yaml`, commit, push, and ArgoCD
reconciles it. Do **not** hand-apply it again.

Sync waves: projects (-5) → operator (0) → DynaKube (5) → easytrade (10).

Astroshop, Bindplane and the retired raw OTel collector are parked in `apps-disabled/`
(not synced). To enable one, `git mv apps-disabled/<app>.yaml apps/`, commit and push.
Astroshop needs the Bindplane gateway first; it's installed by hand from Bindplane Cloud
(see `bindplane/README.md`).

## 6. Verify

```bash
kubectl -n argocd get applications            # all Synced / Healthy
kubectl -n dynatrace get pods                 # operator, webhook, CSI driver, ActiveGate, OneAgent
kubectl -n dynatrace get dynakube
kubectl -n easytrade get pods
kubectl -n easytrade get svc frontendreverseproxy   # EXTERNAL-IP = the app URL (creates a billable AWS load balancer)
```

In the tenant (`lbn6737h.sprint.apps.dynatracelabs.com`):
- Kubernetes app → the cluster is reporting
- Hosts → host group `eks-playground`; network zone `eks-playground`
- Services → easytrade services with OneAgent traces and business events

## 7. SDLC / pipeline observability (GitHub Actions)

This sends **normalized `SDLC_EVENT`s** (pipeline run, build, deployment, change)
to Dynatrace every time `easytrade-config` changes. It's independent of the cluster,
so you can wire it up before or after the cluster is live.

1. **Create the SDLC ingest token** in the tenant
   (Access Tokens → Generate). Scope: **`openpipeline.events_sdlc`**.
   This is a *different* token from the operator/data-ingest Secret above.

2. **Add repo (or org) secrets** on the `easytrade-config` repo
   (Settings → Secrets and variables → Actions):

   | Secret | Value |
   |---|---|
   | `DT_URL` | `https://lbn6737h.sprint.dynatracelabs.com`  (no `/api`, no trailing slash) |
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
field mapping, and how to extend it to new repos.

## Troubleshooting

- **dynakube app fails on first sync** ("no matches for kind DynaKube" / webhook
  not ready): the CRD/webhook needed a moment. Re-sync the `dynatrace-dynakube`
  app (retry is configured, so it usually self-heals).
- **ActiveGate pod Pending**: no node has room for its memory request (10Gi in
  `dynatrace/dynakube.yaml`; nodes have ~14.4 GiB allocatable). Lower the AG memory
  or scale the node group up.
- **PVC stuck Pending**: check `kubectl get storageclass` shows `gp3 (default)` and that
  the chart doesn't name `gp2` (blocked by org policy). After scaling nodes to 0 and
  back, a pod whose volume is in the other AZ waits for a node in that AZ
  ("volume node affinity conflict"): scale back to 2 nodes.
- **easytrade pods CrashLoop on first boot**: DB init ordering. Give it a minute
  or restart the failing deployment.

Teardown steps are in [`BUILD_LOG.md`](BUILD_LOG.md#teardown-when-done).
