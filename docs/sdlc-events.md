# SDLC / pipeline-observability events

How this playground ships **normalized SDLC events** to Dynatrace from GitHub
Actions, so the tenant gets pipeline-observability data (DORA-style delivery
metrics, workflow timings, deployment tracking).

## Why not the `dynatrace-github-action`?

The popular [`dynatrace-oss/dynatrace-github-action`](https://github.com/dynatrace-oss/dynatrace-github-action)
posts to the **classic Events v2 API** (`CUSTOM_DEPLOYMENT`, `CUSTOM_INFO`, …)
attached to monitored entities. Those are *not* the normalized `SDLC_EVENT`
kind used by Pipeline Observability, Dashboards, and the DORA tooling.

For real SDLC events we POST directly to the **built-in SDLC ingest endpoint**:

```
POST https://<env>/platform/ingest/v1/events.sdlc
Authorization: Api-Token <token with scope openpipeline.events_sdlc>
Content-Type: application/json
```

The endpoint auto-enriches each event: it sets `event.kind = "SDLC_EVENT"`,
derives `event.category` (`pipeline` if `cicd.pipeline.*` / `pipeline.*` fields
are present, `task` if `task.*` fields are present), and fills `start_time` /
`end_time` with "now" when omitted. A `202` response means accepted.

Docs:
- Ingest guide — https://docs.dynatrace.com/docs/deliver/pipeline-observability-sdlc-events/sdlc-events
- Field schema — https://docs.dynatrace.com/docs/semantic-dictionary/model/sdlc-events

## Components

| File | Role |
|---|---|
| `gitops/.github/actions/dt-sdlc-event/action.yml` | Reusable composite action. Takes `event-type`, `event-status`, and a `properties` JSON blob; merges and POSTs to the endpoint; fails the job on non-202. |
| `easytrade-config/.github/workflows/sdlc.yml` | Full-lifecycle workflow that calls the action across the delivery flow. |

The action is referenced cross-repo as:

```yaml
uses: <your-org>/gitops/.github/actions/dt-sdlc-event@main
```

so `gitops` must be pushed to GitHub and `set-github-org.sh` must have run.

## Lifecycle emitted by `easytrade-config`

On **merge to `main`** (or manual run):

1. `run` / `started` — pipeline run begins
2. `build` / `started`
3. *(real work: render the kustomization with `kustomize build`)*
4. `build` / `finished` — `task.outcome` = Success/Failure from step 3
5. `deployment` / `started`
6. `deployment` / `finished` — `cicd.deployment.status` = succeeded/failed
7. `run` / `finished` — `cicd.pipeline.run.outcome` = success/failure

On **pull requests**: `change` / `started` (opened/reopened) and
`change` / `finished` (closed → `vcs.change.state` = merged or closed).

> The actual rollout to GKE is done by **ArgoCD** watching this repo (GitOps).
> The `deployment` events above represent that rollout keyed to the merged
> commit. For an event emitted by the deployer itself, add **ArgoCD
> notifications** posting to the same endpoint — a good Phase-2 enhancement.

## Field mapping (GitHub → SDLC semantic dictionary)

| SDLC field | Source |
|---|---|
| `event.provider` | `github.com` |
| `cicd.pipeline.id` / `.name` | workflow name |
| `cicd.pipeline.run.id` | `github.run_id` |
| `cicd.pipeline.run.url.full` | `…/actions/runs/<run_id>` |
| `cicd.pipeline.url.full` | `…/actions/workflows/sdlc.yml` |
| `cicd.pipeline.run.outcome` | success / failure |
| `artifact.id` / `.name` / `.version` | `easytrade` / `EasyTrade` / short SHA |
| `cicd.deployment.namespace` | `easytrade` |
| `cicd.deployment.release_stage` | `production` |
| `cicd.deployment.server.url.full` | `https://kubernetes.default.svc` |
| `vcs.repository.name` / `.url.full` | repo name / URL |
| `vcs.ref.head.name` / `.head.revision` | branch / SHA |
| `vcs.change.id` / `.title` / `.url.full` / `.state` | PR number / title / URL / state |

## Verify in Dynatrace

```dql
fetch events, from: now()-1h
| filter event.kind == "SDLC_EVENT" and event.provider == "github.com"
| sort timestamp desc
| fields timestamp, event.type, event.status, cicd.pipeline.run.id, artifact.version, cicd.deployment.release_stage
```

DORA-flavored example — deployment frequency per day:

```dql
fetch events, from: now()-30d
| filter event.kind == "SDLC_EVENT" and event.type == "deployment" and event.status == "finished"
| makeTimeseries deployments = count(), interval: 1d
```

## Reusing for another repo (e.g. `ai-app`)

1. Add the same `DT_URL` + `DT_SDLC_TOKEN` secrets to that repo.
2. Copy `easytrade-config/.github/workflows/sdlc.yml`, adjust `ARTIFACT_ID`,
   `ARTIFACT_NAME`, `DEPLOY_NS`, and add a real `build`/`test` step.
3. Keep the `uses: <org>/gitops/.github/actions/dt-sdlc-event@main` reference.
