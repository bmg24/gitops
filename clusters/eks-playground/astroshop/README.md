# astroshop — pure-OTel via Bindplane (no OneAgent)

Deploys [Astroshop](https://github.com/open-telemetry/opentelemetry-demo) (the OpenTelemetry
Demo) to the EKS playground, observed **100% through OpenTelemetry + Bindplane** with **zero
OneAgent injection** — the vendor-neutral counterpart to easytrade's OneAgent full-stack story.

## How it's wired

| Piece | Where | Notes |
|---|---|---|
| ArgoCD Application | [`apps/astroshop.yaml`](../../../apps/astroshop.yaml) | Multi-source: community chart + this repo's values file (`$values` ref). Sync-wave **20**. |
| Values overrides | [`values.yaml`](./values.yaml) | Disables bundled collector + demo backends; repoints all OTLP to the Bindplane gateway. |
| Chart | `opentelemetry-demo` from `https://open-telemetry.github.io/opentelemetry-helm-charts` | Pinned to **0.41.0** in `apps/astroshop.yaml`. The community chart lives in `opentelemetry-helm-charts`, **not** in the app fork. |
| OTLP gateway | Bindplane-managed Gateway in `bindplane-agent` (see [`../bindplane/`](../bindplane/README.md)) | Installed out of band from Bindplane Cloud (UI → kubectl apply), **not** via ArgoCD. astroshop exports here. |
| App fork | `bmg24/astroshop` (fork of `open-telemetry/opentelemetry-demo`) | Only needed to build **custom images** for DevCycle (Step 4); referenced via `components.<svc>.imageOverride`. Not the chart source. |

## Deviations from the stock demo

- **Bundled `opentelemetry-collector` disabled** — the Bindplane-managed Gateway is the sole egress to Dynatrace.
- **Jaeger / Prometheus / Grafana / OpenSearch disabled** — local backends that only fed the bundled collector; dropping them frees ~2.4 GB. The frontend-proxy's `/jaeger` `/grafana` `/prometheus` links will 502 (harmless).
- **`OTEL_COLLECTOR_NAME` overridden** to the Bindplane gateway Service DNS — one global env flips every service's OTLP target.
- **flagd kept** — it's the OpenFeature seam; DevCycle (an OpenFeature provider) plugs in here in Step 4.

## Manual steps (Brian)

1. **Stand up the Bindplane Gateway** — Bindplane Cloud → build the Dynatrace destination + a
   Gateway configuration, then Install Collectors → `kubectl apply` into `bindplane-agent`.
   Full steps in [`../bindplane/README.md`](../bindplane/README.md).
2. **Confirm the gateway Service DNS** — `kubectl get svc -n bindplane-agent`; ensure it matches
   `OTEL_COLLECTOR_NAME` in `values.yaml` (default `bindplane-gateway-agent.bindplane-agent.svc.cluster.local`).
3. **Fork** `open-telemetry/opentelemetry-demo` → `bmg24/astroshop` — only needed for Step 4
   (DevCycle custom images); not required just to deploy.
4. Commit + push; ArgoCD syncs astroshop (sync-wave 20). Chart is already pinned to 0.41.0.

> **Decision (2026-08-08):** the raw, Bindplane-unmanaged collector (the earlier collector-first
> step) was **dropped** in favor of a Bindplane-managed Gateway from the start. Same telemetry,
> but managed by Bindplane Cloud over OpAMP. The native Kubernetes app is already covered by the
> existing DynaKube `kubernetes-monitoring` ActiveGate, so the gateway ships app telemetry only.

## Order of operations

The Bindplane Gateway must be receiving before astroshop for telemetry to flow. astroshop will
start and run without it, but services retry their OTLP export until the gateway Service exists.
The gateway is hand-applied (not ArgoCD), so there's no cross-app sync-wave dependency now.
