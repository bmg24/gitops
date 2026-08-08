# astroshop — pure-OTel via Bindplane (no OneAgent)

Deploys [Astroshop](https://github.com/open-telemetry/opentelemetry-demo) (the OpenTelemetry
Demo) to the GKE playground, observed **100% through OpenTelemetry + Bindplane** with **zero
OneAgent injection** — the vendor-neutral counterpart to easytrade's OneAgent full-stack story.

## How it's wired

| Piece | Where | Notes |
|---|---|---|
| ArgoCD Application | [`apps/astroshop.yaml`](../../../apps/astroshop.yaml) | Multi-source: community chart + this repo's values file (`$values` ref). Sync-wave **20** (after Bindplane collectors at 15). |
| Values overrides | [`values.yaml`](./values.yaml) | Disables bundled collector + demo backends; repoints all OTLP to the Bindplane gateway. |
| Chart | `opentelemetry-demo` from `https://open-telemetry.github.io/opentelemetry-helm-charts` | Pin a version (`REPLACE_CHART_VERSION`). The community chart lives in `opentelemetry-helm-charts`, **not** in the app fork. |
| App fork | `bmg24/astroshop` (fork of `open-telemetry/opentelemetry-demo`) | Only needed to build **custom images** for DevCycle (Step 4); referenced via `components.<svc>.imageOverride`. Not the chart source. |

## Deviations from the stock demo

- **Bundled `opentelemetry-collector` disabled** — Bindplane-managed collectors are the sole egress to Dynatrace.
- **Jaeger / Prometheus / Grafana / OpenSearch disabled** — local backends that only fed the bundled collector; dropping them frees ~2.4 GB. The frontend-proxy's `/jaeger` `/grafana` `/prometheus` links will 502 (harmless).
- **`OTEL_COLLECTOR_NAME` overridden** to the Bindplane gateway Service DNS — one global env flips every service's OTLP target.
- **flagd kept** — it's the OpenFeature seam; DevCycle (an OpenFeature provider) plugs in here in Step 4.

## Manual steps (Brian)

1. **Fork** `open-telemetry/opentelemetry-demo` → `bmg24/astroshop` (upstream remote for syncing). Needed for Step 4 image builds; not required just to deploy.
2. **Pin the chart version** — replace `REPLACE_CHART_VERSION` in `apps/astroshop.yaml`.
3. **Collector DNS is already wired** — `values.yaml` points `OTEL_COLLECTOR_NAME` at the raw
   Dynatrace OTel Gateway: `otel-collector.otel-collector.svc.cluster.local`
   (see [`../otel-collector/`](../otel-collector/README.md)). No placeholder left to fill.
4. Commit + push; ArgoCD syncs (after the `otel-collector` app at sync-wave 15).

> **Decision (2026-08-08):** starting collector-first with a raw, Bindplane-unmanaged Gateway
> collector (`../otel-collector/`) instead of the full Bindplane Node+Gateway topology. Same
> telemetry, lowest friction; promote to Bindplane management later. The native Kubernetes app is
> already covered by the existing DynaKube `kubernetes-monitoring` ActiveGate, so the collector
> ships app telemetry only.

## Order of operations

The `otel-collector` app (sync-wave 15) must land before this app for telemetry to flow. Astroshop
will start and run without it, but services will fail to export until the gateway Service exists.
