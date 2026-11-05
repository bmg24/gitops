# Bindplane Gateway — the OTLP pipeline for astroshop

A single **Bindplane-managed Gateway** collector in the GKE cluster routes **all astroshop
telemetry (traces, metrics, logs) → Bindplane-managed collector → Dynatrace**, with **no
OneAgent**. Control plane is **Bindplane Cloud** (`app.bindplane.com`); the collector dials out
over OpAMP. This replaces the earlier raw `otel-collector` (dropped 2026-08-08).

> **Scope:** Gateway only. We are **not** deploying Node/Cluster collectors — the native
> Kubernetes app (namespace/workload/pod topology) is already covered cluster-wide by the
> existing DynaKube `kubernetes-monitoring` ActiveGate. The gateway ships astroshop **app**
> telemetry only. Add a Node collector later only if you want the separate pure-OTel k8s
> dashboards.

> **How the install works (read first):** Bindplane collectors are **not** installed with Helm
> or ArgoCD. You build a *configuration* in Bindplane Cloud, then the **Install Collectors** page
> generates a k8s manifest (OpAMP endpoint + registration secret baked in) that you `kubectl
> apply`. Bindplane then pushes the pipeline config to the running collector over OpAMP. So the
> real pipeline lives in Bindplane Cloud, not in this repo — this folder holds the applied
> manifest for the record (secret gitignored). **It is intentionally not managed by ArgoCD.**

## Topology

```
astroshop services ──OTLP (4317/4318)──▶ Bindplane Gateway (Deployment) ──OTLP──▶ Dynatrace /api/v2/otlp
                                          (bindplane-agent ns, OpAMP → Bindplane Cloud)
```

astroshop points at the gateway Service DNS via `OTEL_COLLECTOR_NAME` in
[`../astroshop/values.yaml`](../astroshop/values.yaml).

## Step-by-step (Brian, in Bindplane Cloud + kubectl)

### 1. Dynatrace destination (Bindplane Cloud → Destinations)
- Type: **Dynatrace** (OTLP).
- OTLP endpoint: `https://anr47691.sprint.dynatracelabs.com/api/v2/otlp`
- API token: **reuse the data-ingest token already created** for the raw collector
  (`dt0c01`, scopes `openTelemetryTrace.ingest`, `metrics.ingest`, `logs.ingest`). Bindplane
  applies the `Authorization: Api-Token <token>` header for you.

### 2. Gateway configuration (Bindplane Cloud → Configurations)
- Platform: **Kubernetes Gateway**.
- **Source:** OpenTelemetry (OTLP) — listens on 4317/4318.
- **Processors** (to match what the raw collector did):
  - **Kubernetes Attributes** — enrich by pod IP; extract namespace, pod name/uid, deployment,
    replicaset, statefulset, daemonset, node, container.
  - **Batch** — (optional tuning; Bindplane batches by default).
  - `memory_limiter` + `health_check` are added by the Bindplane agent automatically — don't add them.
- **Destination:** the Dynatrace destination from Step 1.

### 3. Install the Gateway (Bindplane Cloud → Collectors → Install Collectors)
- Choose **Kubernetes** + the Gateway config → copy the generated manifest.
- Apply it: `kubectl apply -n bindplane-agent -f <generated>.yaml`
  (creates the namespace if the manifest includes it; otherwise `kubectl create ns bindplane-agent` first).
- Save the manifest here as `gateway.yaml` for the record — the OpAMP **registration secret stays
  out of git** (see `.gitignore`; recreate via kubectl on a fresh cluster, like the DynaKube token).

### 4. Wire astroshop to the Gateway
- Find the gateway OTLP Service: `kubectl get svc -n bindplane-agent` (ports 4317/4318).
- Confirm its cluster DNS matches `OTEL_COLLECTOR_NAME` in `../astroshop/values.yaml`
  (default `bindplane-gateway-agent.bindplane-agent.svc.cluster.local`). Update the value if you
  named the collector/config differently, then commit + push (ArgoCD deploys astroshop, wave 20).

### 5. Verify
- Bindplane Cloud shows the Gateway collector **connected/healthy** with throughput.
- Pods: `kubectl -n bindplane-agent get pods` (gateway Running).
- Dynatrace: astroshop spans/metrics/logs arriving via OTLP (source = collector, not OneAgent):
  `fetch spans, from:now()-30m | filter k8s.namespace.name == "astroshop" | summarize count()` → > 0.

## Why not ArgoCD for the gateway?

The Bindplane-generated manifest is just a thin OpAMP agent with a baked-in registration secret;
the actual pipeline config is pushed from Bindplane Cloud, not stored in git. GitOps-managing it
would mean committing a secret-bearing generated blob for little benefit. Hand-apply is Bindplane's
supported path and keeps the secret out of the repo — same pattern as the DynaKube token.
