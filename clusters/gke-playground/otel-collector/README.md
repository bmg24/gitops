# otel-collector — raw Dynatrace OTel Gateway (collector-first)

A single, **Bindplane-unmanaged** Dynatrace OTel Collector in **Gateway** mode. It receives
astroshop's OTLP (traces, metrics, logs) and exports straight to the Dynatrace tenant. This is
the low-friction first step of the astroshop pure-OTel pipeline — prove telemetry lands, then
adopt Bindplane later without changing what the collector emits.

## Why this before Bindplane

"Bindplane vs. raw collector" is only a **management** choice — the telemetry is identical either
way. Starting raw gets astroshop's app observability into Dynatrace with one `kubectl apply` and
zero external control-plane dependency. To promote later, point this collector's OpAMP endpoint at
Bindplane Cloud; the pipeline config gets rebuilt in Bindplane's UI, but the data path is unchanged.

## What it does / doesn't do

- **Does:** OTLP receive (gRPC 4317 / HTTP 4318) → `k8sattributes` enrich → `batch` → `otlphttp`
  export to `https://anr47691.sprint.dynatracelabs.com/api/v2/otlp`.
- **Doesn't:** collect node/cluster infra metrics or k8s events. The native **Kubernetes app**
  (namespace/workload/pod topology, node utilization) is already populated cluster-wide by the
  existing DynaKube `kubernetes-monitoring` ActiveGate — no OTel receivers needed for that.
  Add `k8s_cluster` / `kubeletstats` / `k8s_events` receivers later only if you want the separate
  pure-OTel "OpenTelemetry K8s" ready-made dashboards.

## Topology

```
astroshop services ──OTLP (4317/4318)──▶ otel-collector (Gateway, Deployment) ──otlphttp──▶ Dynatrace /api/v2/otlp
```

Gateway Service DNS (what astroshop points at): `otel-collector.otel-collector.svc.cluster.local`

## Manual steps (Brian)

1. **Create the data-ingest token** in the tenant → Access Tokens. Scopes:
   `openTelemetryTrace.ingest`, `metrics.ingest`, `logs.ingest` (classic `dt0c01`).
2. **Create the credential Secret** (gitignored — never committed). See `secret.example.yaml`:
   ```bash
   kubectl create namespace otel-collector   # or let ArgoCD create it first
   kubectl -n otel-collector create secret generic dynatrace-otelcol-credentials \
     --from-literal=DT_ENDPOINT='https://anr47691.sprint.dynatracelabs.com/api/v2/otlp' \
     --from-literal=DT_API_TOKEN='dt0c01.<DATA_INGEST_TOKEN>'
   ```
3. **Push** the gitops changes → ArgoCD syncs `apps/otel-collector.yaml` (sync-wave 15), which
   applies `collector.yaml` into the `otel-collector` namespace, before astroshop (wave 20).

## Verify

```bash
kubectl -n otel-collector get pods                 # collector Running
kubectl -n otel-collector logs deploy/otel-collector | grep -i export   # no auth/export errors
```

Then in Dynatrace, once astroshop is sending: astroshop spans/metrics/logs arrive via OTLP
(source = collector, not OneAgent). Filter by `k8s.namespace.name == "astroshop"`.

## GitOps note

`apps/otel-collector.yaml` (sync-wave 15) directory-syncs the manifests in this folder, excluding
docs. The credential Secret is applied by hand (gitignored), the same pattern as the DynaKube token.
