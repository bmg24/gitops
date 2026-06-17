# Dynatrace GKE deployment — status & troubleshooting

> **Purpose:** Self-contained context for the Dynatrace deployment on the GKE
> playground. Read this to understand the current state and the issues that were
> resolved getting there. **Last updated:** 2026-06-16.

---

## TL;DR — ✅ DEPLOYED & HEALTHY

The single `brian-obs-playground` DynaKube is **`Running`**. ActiveGate (1/1) and the
OneAgent DaemonSet (3/3, one per node) are up and **OneAgent is reporting on all three
nodes**; every DynaKube status condition is `True`. Cluster + hosts data flows to the
tenant. Three sequential problems were resolved to get here:

1. **Two DynaKubes** → consolidated into one full-stack DynaKube.
2. **`TokenError`** → the operator can't validate **platform tokens** (`dt0s16`); switched
   to **classic API tokens** (`dt0c01`).
3. **`ImagePullBackOff`** → the sprint tenant's built-in registry hosts no agent images;
   repointed OneAgent/CodeModules/ActiveGate to **public Amazon ECR**, and removed the
   EEC-dependent features (`dynatrace-eec` isn't on the public registry).

The GitHub Actions **SDLC events** track is independent and not affected by any of this.

---

## Current deployment (what's running)

| Component | State | Detail |
|---|---|---|
| `dynatrace-operator` | ✅ 1/1 | operator 1.8.1 |
| `dynatrace-webhook` | ✅ 2/2 | |
| `dynatrace-oneagent-csi-driver` | ✅ 4/4 ×3 | required for public-registry image pulls |
| `brian-obs-playground-activegate-0` | ✅ 1/1 | kubernetes-monitoring + routing + debugging |
| `brian-obs-playground-oneagent-*` | ✅ 1/1 ×3 | cloudNativeFullStack, one per node |
| DynaKube `brian-obs-playground` | ✅ `Running` | all conditions `True` |

**Images (pinned, public ECR, verified to resolve):**
- OneAgent — `public.ecr.aws/dynatrace/dynatrace-oneagent:1.339.55.20260615-110349`
- CodeModules — `public.ecr.aws/dynatrace/dynatrace-codemodules:1.339.55.20260615-110349`
- ActiveGate — `public.ecr.aws/dynatrace/dynatrace-activegate:1.339.39.20260605-153224`

> Public ECR's newest GA is `1.339.x`; the sprint tenant runs `1.341.x`. The slightly
> older public agent connects and reports normally — expected, since sprint runs newer
> internal builds than the public mirror.

---

## Environment facts

| Thing | Value |
|---|---|
| Dynatrace tenant | `anr47691` (sprint, INTERNAL) |
| Ingest/API host | `https://anr47691.sprint.dynatracelabs.com` |
| Apps/UI host | `https://anr47691.sprint.apps.dynatracelabs.com` |
| GCP project | `sales-engineering-noram` |
| GKE cluster | `easytrade-playground`, zonal `us-central1-a`, `e2-standard-4`, 3–7 nodes (autoscaling) |
| K8s namespace | `dynatrace` |
| Dynatrace Operator | `1.8.1` (GA), Helm OCI `public.ecr.aws/dynatrace`, ArgoCD sync-wave 0 |
| DynaKube CRD apiVersion | `dynatrace.com/v1beta6` |
| GitHub user/org | `bmg24` (personal) |
| Repos | `bmg24/gitops`, `bmg24/easytrade-config` (public so ArgoCD can read them) |

### DynaKube object
- **Single** `brian-obs-playground` — full-stack: `cloudNativeFullStack` OneAgent
  (`hostGroup: gke`) + one ActiveGate with `kubernetes-monitoring` + `routing` + `debugging`
  (1 replica, 10Gi), explicit public-ECR image overrides.
- **No EEC-dependent features** (`extensions`, `extensionExecutionController`, `otelCollector`,
  `telemetryIngest`, `logMonitoring`) — removed; see "Problem 3" below.
- References secret `brian-obs-playground` (`spec.tokens`).

### Tokens — IMPORTANT
- ⏳ **Platform tokens under internal review (2026-06-16).** Classic `dt0c01` tokens
  are the working path and what's deployed now. Whether/how **platform tokens** are
  meant to work for operator deployment is being reviewed with internal Dynatrace teams;
  revisit and switch if guidance changes. For now: classic tokens only.
- The operator requires **classic API tokens (`dt0c01`)**, NOT platform tokens (`dt0s16`).
  This was the central, hard-won finding — see Problem 2.
- `apiToken` (operator, template "Kubernetes: Dynatrace Operator") + a **separate**
  `dataIngestToken` (`metrics.ingest`, `logs.ingest`, `openTelemetryTrace.ingest`).
- Secret applied by hand from `dynatrace/secret.yaml` — **gitignored**, **not** managed by
  ArgoCD (kustomization references `dynakube.yaml` only). Editing the file does nothing until
  `kubectl apply` + operator restart.

---

## How we got here — problems & fixes

### Problem 1 — Two DynaKubes ✅ fixed
The connect-cluster flow generated two DynaKubes (`brian-obs-playground` AG-only +
`brian-obs-playground-agents` full-stack). The webhook warned that two DynaKubes on the
same `apiUrl` with extensions risk **double-billing**. **Fix:** consolidated into one
full-stack `brian-obs-playground`; ArgoCD (`prune: true`) deleted the `-agents` object.

### Problem 2 — `TokenError` ✅ fixed (the big one)
Layered; each fix revealed the next:
1. **Missing scopes.** First `dt0s16` token 403'd on `api-tokens:tokens:read`; probing showed
   it lacked **essentially all** operator scopes. The DynaKube only ever shows the *first*
   failure (operator validates sequentially).
2. **Platform tokens can't be validated at all.** After regenerating a `dt0s16` token *with*
   scopes, the error became `404 Token does not exist`. Proven empirically: the operator
   validates a token by POSTing it to **`/api/v2/apiTokens/lookup`**, which only knows
   **classic** tokens. A platform token *authenticates* fine (`GET /settings/schemas` → 200)
   but is **404 on lookup** → never validates. **The operator requires classic `dt0c01`.**
3. **Secret hygiene bugs found along the way:** tokens as plaintext under `data:` (must be
   base64 / `stringData:`); the same token in both keys; a `REPLACE_…` placeholder applied
   once (→ `404` / `401 invalid format`).

**Fix:** classic `dt0c01` tokens — `apiToken` from template **"Kubernetes: Dynatrace
Operator"**, separate `dataIngestToken` with ingest scopes, under `stringData:`, applied by
hand, operator restarted → `Tokens: TokenReady`, all conditions `True`, workloads created.

### Problem 3 — `ImagePullBackOff`: tenant registry empty ✅ fixed
After tokens were fixed, the operator created all workloads but every pod hit
`ImagePullBackOff` with `NotFound` (not auth). The operator derives the image registry from
the `apiUrl` host (`anr47691.sprint…/linux/...`) — correct for SaaS, but:
- That built-in registry is an **installer** source, **not** a full image registry. Dynatrace
  explicitly recommends **against** it for container images, and on this sprint tenant it
  hosts **no** `oneagent`/`activegate`/`dynatrace-eec` manifests (every tag → 404). The tenant
  *does* serve OneAgent installers via the deployment API — just not OCI images.
- **Fix:** pull from **public registries** (Amazon ECR `public.ecr.aws/dynatrace/...`), which
  Dynatrace recommends — signed, immutable, anonymous (no pull secret), CSI driver required
  (already on). Set explicit `oneAgent.cloudNativeFullStack.image` / `.codeModulesImage` /
  `activeGate.image` to verified immutable GA tags.
- **EEC gotcha:** `dynatrace-eec` is **not** on the public registry, so the EEC-dependent
  features were **removed** (extensions/EEC/otelCollector/telemetryIngest/logMonitoring). The
  earlier otel-collector crashloop was a *cascade* of the un-pullable EEC. Core full-stack
  (hosts, processes, K8s, services/traces) needs none of them. Re-add later only with an EEC
  image source (built-in registry once populated, or a private mirror).

Docs: [Use a public registry](https://docs.dynatrace.com/docs/ingest-from/setup-on-k8s/guides/container-registries/use-public-registry) ·
[Private registries](https://docs.dynatrace.com/docs/ingest-from/setup-on-k8s/guides/container-registries/use-private-registry)

### Non-bug — "Is ArgoCD reloading the secret?"
No. The Secret is gitignored and not in the kustomization, so **ArgoCD never manages or
reloads it.** Apply by hand + restart the operator to re-validate.

### Problem 4 — easytrade app: remote kustomize base fails ⏳ open

ArgoCD `easytrade` app errors on manifest generation:

```
kustomize build … failed: accumulating resources from
'https://github.com/Dynatrace/easytrade//kubernetes-manifests/release?ref=main':
URL is a git repository': evalsymlink failure on
'/tmp/kustomize-…/kubernetes-manifests/release' : … no such file or directory
```

- **Cause:** `easytrade-config/kustomization.yaml` uses a **remote git base** (`…/easytrade//kubernetes-manifests/release?ref=main`). ArgoCD's repo-server doesn't reliably resolve remote git bases in kustomize — this `evalsymlink failure … no such file or directory` is the signature — and/or the upstream subpath (`kubernetes-manifests/release`) doesn't exist on `main`.
- **Verify the real upstream path first** (run locally, where GitHub is reachable):
  ```bash
  curl -s "https://api.github.com/repos/Dynatrace/easytrade/contents/kubernetes-manifests?ref=main" | jq -r '.[].name'
  ```
- **Fix (recommended):** point the ArgoCD `easytrade` Application **directly** at the upstream repo, dropping the remote-base wrapper. In `apps/easytrade.yaml`:
  ```yaml
  source:
    repoURL: https://github.com/Dynatrace/easytrade.git
    targetRevision: main
    path: <verified path, e.g. kubernetes-manifests>   # directory or local kustomization
  ```
  If the path is plain YAML, ArgoCD deploys it as a directory; if it has a local `kustomization.yaml`, ArgoCD builds it locally (no remote base).
- **To keep patches:** instead vendor the upstream manifests into `easytrade-config/` (copy the YAML in) and kustomize over local files, or use an ArgoCD **multi-source** app (upstream repo as a `ref`, overlay as the build source). Remote git bases inside kustomize-under-ArgoCD should be avoided.
- Until fixed, easytrade won't deploy — but it does **not** block the Dynatrace stack, which is healthy and reporting.

---

## Open / optional follow-ups

- **Orphaned injection artifacts (cosmetic).** `argocd` + `default` namespaces still carry
  stale `dynakube.internal.dynatrace.com/instance: brian-goins-playground-agents` labels +
  leftover `dynatrace-bootstrapper-{certs,config}` secrets (old pre-rename instance).
- **`namespaceSelector` is empty** → OneAgent injects cluster-wide (matched `argocd`,
  `default`). Scope it to the app namespace(s) once easytrade lands.
  ```bash
  kubectl label ns argocd default dynakube.internal.dynatrace.com/instance-
  kubectl -n argocd  delete secret dynatrace-bootstrapper-certs dynatrace-bootstrapper-config
  kubectl -n default delete secret dynatrace-bootstrapper-certs dynatrace-bootstrapper-config
  ```
- **easytrade** ArgoCD app is `Unknown` (not deployed yet) — the workload to OneAgent-inject.
- Re-add EEC features if/when an EEC image source exists.

---

## Diagnostic / verify commands

```bash
# Deployment health
kubectl -n dynatrace get dynakube
kubectl -n dynatrace get pods
kubectl -n dynatrace get dynakube brian-obs-playground \
  -o jsonpath='{range .status.conditions[*]}{.type}{"="}{.status}{" "}{end}{"\n"}'

# Live image overrides
kubectl -n dynatrace get dynakube brian-obs-playground \
  -o jsonpath='OA: {.spec.oneAgent.cloudNativeFullStack.image}{"\n"}AG: {.spec.activeGate.image}{"\n"}'

# Confirm a token is CLASSIC (dt0c01) and validates via the operator's lookup path
A=$(kubectl -n dynatrace get secret brian-obs-playground -o jsonpath='{.data.apiToken}' | base64 -d)
echo "$A" | cut -c1-7            # expect dt0c01 (NOT dt0s16)
curl -s -X POST "https://anr47691.sprint.dynatracelabs.com/api/v2/apiTokens/lookup" \
  -H "Authorization: Api-Token $A" -H "Content-Type: application/json" \
  -d "{\"token\":\"$A\"}" | jq '.scopes'   # 404 = a platform token (wrong type)

# List available public-ECR tags when bumping versions
skopeo list-tags docker://public.ecr.aws/dynatrace/dynatrace-oneagent   # or browse gallery.ecr.aws/dynatrace
```

---

## File layout (gitops repo)

```
clusters/gke-playground/
├── setup.md                     # cluster + ArgoCD bootstrap recipe (+ SDLC section)
├── TROUBLESHOOTING.md           # this file
└── dynatrace/
    ├── dynakube.yaml            # ONE full-stack DynaKube, public-ECR image overrides
    ├── secret.yaml              # token Secret — GITIGNORED, applied by hand, NOT in ArgoCD
    ├── secret.example.yaml      # template (stringData; classic dt0c01 tokens)
    ├── kustomization.yaml       # references dynakube.yaml only
    └── .gitignore               # blocks secret.yaml + *.secret.yaml
apps/dynatrace-dynakube.yaml     # ArgoCD app (prune + selfHeal, ServerSideApply)
```

---

## Reference docs
- Tokens & permissions — <https://docs.dynatrace.com/docs/ingest-from/setup-on-k8s/deployment/tokens-permissions>
- Use a public registry — <https://docs.dynatrace.com/docs/ingest-from/setup-on-k8s/guides/container-registries/use-public-registry>
- DynaKube parameters (image overrides) — <https://docs.dynatrace.com/docs/ingest-from/setup-on-k8s/reference/dynakube-parameters>
- Public images gallery — <https://gallery.ecr.aws/dynatrace>
