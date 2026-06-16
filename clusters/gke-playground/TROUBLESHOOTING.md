# Dynatrace GKE deployment — status & troubleshooting

> **Purpose:** Self-contained context for debugging the Dynatrace deployment on the
> GKE playground. Hand this to Claude Code (or any engineer) to continue without
> prior context. **Last updated:** 2026-06-16.

---

## TL;DR — where we are now

The cluster, ArgoCD, the Dynatrace **Operator/CSI/webhook**, and the **DynaKube tokens
are all healthy.** The token problems that blocked everything are **fully resolved**
(see history below), and the operator has now created **all** workloads — ActiveGate,
OneAgent DaemonSet, extension controller (EEC), and OTel collector.

**Current (and only) blocker:** none of those pods can start because their **container
images cannot be pulled** — the operator pulls agent/AG/EEC images from the tenant's own
registry (`anr47691.sprint.dynatracelabs.com/linux/...`), and on this **sprint/internal
tenant that registry is empty** (every tag → HTTP 404 / `not found`). So the DynaKube is
in `Deploying`, all new pods are `ImagePullBackOff`, and **nothing reports in the tenant yet.**

**Fix (researched & applied to `dynakube.yaml` on 2026-06-16):** pull OneAgent /
CodeModules / ActiveGate from the **public Amazon ECR** registry instead of the empty
built-in registry, and **remove the EEC-dependent features** (because `dynatrace-eec`
is not published publicly — see "Image registry resolution" below). Remaining action:
pin the image tags and push. **This is the next action.**

The GitHub Actions **SDLC events** track is independent and not blocked by any of this.

---

## Image registry resolution (2026-06-16)

**Why the built-in registry 404s:** the operator defaults the image registry to the
`apiUrl` host (`anr47691.sprint…/linux/...`). That built-in registry is an installer
source, not a full image registry — Dynatrace explicitly recommends **against** using it
for container images (only exception: Classic Full-Stack OneAgent). On this sprint tenant
it isn't populated for `oneagent`/`activegate`/`dynatrace-eec`, so every manifest tag 404s.
"From the tenant" is therefore **not** the supported source for cloud-native full-stack.

**Supported source = public registries** (Amazon ECR `public.ecr.aws/dynatrace/...` or
Docker Hub `dynatrace/...`). Set explicit `image` fields in the DynaKube:
- `oneAgent.cloudNativeFullStack.image` = `public.ecr.aws/dynatrace/dynatrace-oneagent:<tag>`
- `oneAgent.cloudNativeFullStack.codeModulesImage` = `public.ecr.aws/dynatrace/dynatrace-codemodules:<tag>`
- `activeGate.image` = `public.ecr.aws/dynatrace/dynatrace-activegate:<tag>`

Public ECR is anonymous (no `customPullSecret`), requires the CSI driver (already on),
and uses **immutable version tags** (no `latest`). List tags with
`skopeo list-tags docker://public.ecr.aws/dynatrace/dynatrace-oneagent` or browse
<https://gallery.ecr.aws/dynatrace>.

**EEC gotcha:** `dynatrace-eec` is **not** on the public registry. So `extensions`,
`templates.extensionExecutionController`, `templates.otelCollector`, and `telemetryIngest`
(all EEC-dependent) were **removed** from the DynaKube — they can't pull EEC from a public
registry. The otel-collector crashloop was a cascade of the missing EEC. Core full-stack
(hosts, processes, K8s, services/traces) needs none of these. Re-add later only with an
EEC image source (built-in registry once populated, or a private mirror).

Docs: [Use a public registry](https://docs.dynatrace.com/docs/ingest-from/setup-on-k8s/guides/container-registries/use-public-registry) ·
[Store images in private registries](https://docs.dynatrace.com/docs/ingest-from/setup-on-k8s/guides/container-registries/prepare-private-registry) ·
[Use a private registry](https://docs.dynatrace.com/docs/ingest-from/setup-on-k8s/guides/container-registries/use-private-registry)

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
| Dynatrace Operator | `1.8.1` (GA), Helm OCI `public.ecr.aws/dynatrace`, installed by ArgoCD (sync-wave 0) |
| DynaKube CRD apiVersion | `dynatrace.com/v1beta6` |
| GitHub user/org | `bmg24` (personal) |
| Repos | `bmg24/gitops`, `bmg24/easytrade-config` (made **public** so ArgoCD can read them) |

### DynaKube object (current)

- **Single** `brian-obs-playground` DynaKube — full-stack all-in-one:
  - `oneAgent.cloudNativeFullStack` (OneAgent DaemonSet), `hostGroup: gke`
  - one `activeGate` with **all** capabilities: `kubernetes-monitoring`, `routing`, `debugging` (1 replica, 10Gi)
  - `logMonitoring`, `telemetryIngest` (otlp/jaeger/statsd/zipkin), `extensions.prometheus`, EEC + OTel templates
- References secret `brian-obs-playground` (`spec.tokens: brian-obs-playground`).
- **History:** originally TWO DynaKubes (`brian-obs-playground` AG-only + `brian-obs-playground-agents`
  full-stack), themselves renamed from `brian-goins-playground*`. Consolidated to one on 2026-06-16
  (see Problem 1). ArgoCD pruned the `-agents` object on sync.

### Tokens — IMPORTANT

- The operator requires **classic API tokens (`dt0c01`)**, NOT platform tokens (`dt0s16`).
  This was the central, hard-won finding — see Problem 2.
- `apiToken` (operator) + `dataIngestToken` live in secret `brian-obs-playground`.
- Secret is applied by hand from `dynatrace/secret.yaml` (gitignored) — **not** in Git,
  and **not** managed by ArgoCD (kustomization references `dynakube.yaml` only).

---

## What's working ✅

- GKE cluster up; `kubectl` context = `easytrade-playground`.
- ArgoCD reads `bmg24/gitops` + `bmg24/easytrade-config` (fixed by making repos public).
- ArgoCD app-of-apps syncing in waves: operator (0) → dynakube (5) → easytrade (10).
  `dynatrace-dynakube` app = `automated {prune, selfHeal}`, `ServerSideApply`.
- `dynatrace` namespace control-plane pods healthy:
  - `dynatrace-operator` (1/1), `dynatrace-webhook` (2/2), `dynatrace-oneagent-csi-driver` (4/4 ×3).
- **DynaKube tokens fully valid** — every status condition is `True` (`Tokens: TokenReady`,
  settings read/write, ActiveGate connection/version/auth-token, OneAgent connection,
  code-modules, extensions, etc.).
- Operator **created all workloads**: ActiveGate StatefulSet, OneAgent DaemonSet (3),
  extension-controller StatefulSet, OTel-collector StatefulSet, plus all supporting
  secrets/services/configmaps.

## What's NOT working ❌ (current blocker)

`kubectl -n dynatrace get dynakube` → `STATUS=Deploying`; pods can't pull images:

| Pod | State | Image (auto-derived from apiUrl host) |
|---|---|---|
| `…-activegate-0` | `Init:ImagePullBackOff` | `anr47691.sprint…/linux/activegate:1.341.11-raw` → **404 not found** |
| `…-extension-controller-0` (EEC) | `ImagePullBackOff` | `anr47691.sprint…/linux/dynatrace-eec:latest` → **404 not found** |
| `…-oneagent-*` (×3) | `ImagePullBackOff` | `anr47691.sprint…/linux/oneagent:1.341.20-raw` → **404 not found** |
| `…-otel-collector-0` | `CrashLoopBackOff` | pulls fine (`public.ecr.aws/dynatrace/dynatrace-otel-collector`) — **cascade**: can't reach EEC at `:14599/otcconfig/...` because EEC isn't running |

Because no agent/AG pod runs, **nothing reports in the tenant** (no hosts, no K8s data, no traces).

### Why the images 404 (verified)
- Pull error is `code = NotFound … not found`, **not** an auth/401 — the pull secret
  (`brian-obs-playground-pull-secret`) authenticates fine.
- Registry root `GET /v2/` → **200**, but `GET /v2/linux/{oneagent,activegate,dynatrace-eec}/manifests/<any tag>` → **404** for every tag tried (`-raw`, full version, `latest`).
- The tenant **does** serve OneAgent *installers* via `/api/v1/deployment/installer/agent/versions/...`
  (latest `1.341.20.20260616-144004`) — so it's a *container-image registry* gap, not a tenant outage.
- Conclusion: the operator defaults the image registry to the `apiUrl` host (correct for SaaS),
  but this **sprint/internal tenant doesn't host container images there**.

---

## History — problems hit and how they were fixed

### Problem 1 — Two DynaKubes (cleanup) ✅ fixed
- The connect-cluster flow generated two DynaKubes (`brian-obs-playground` AG-only +
  `brian-obs-playground-agents` full-stack). The webhook also warned that two DynaKubes
  on the same `apiUrl` with extensions risk **double-billing**.
- **Fix:** consolidated into a **single** full-stack `brian-obs-playground` DynaKube
  (one ActiveGate carrying `kubernetes-monitoring` + `routing` + `debugging`, plus
  cloudNativeFullStack OneAgent and all extras). Committed to `dynakube.yaml`; ArgoCD
  (`prune: true`) deleted the `-agents` object on sync.

### Problem 2 — DynaKube stuck in `Error` / `TokenError` ✅ fixed (the big one)
This went through several layers; each fix revealed the next:

1. **Missing scopes (platform token).** Initial `dt0s16` token 403'd on `api-tokens:tokens:read`.
   Probing showed it was missing **essentially all** operator scopes (settings, metrics,
   activegate-tokens, installer). The DynaKube only ever showed the *first* failure because
   the operator validates scopes sequentially.
2. **Platform tokens don't work with the operator at all.** After regenerating a `dt0s16`
   token *with* scopes, the error changed to `404: Token does not exist`. Root cause proven
   empirically: the operator validates a token by POSTing it to
   **`/api/v2/apiTokens/lookup`**, which only knows **classic** tokens. A platform token
   *authenticates* fine (`GET /settings/schemas` → 200) but is **404 on lookup** → the
   operator can never validate it. **The operator requires classic `dt0c01` tokens.**
3. **Secret hygiene bugs found along the way:**
   - tokens pasted as **plaintext under `data:`** (must be base64, or use `stringData:`) —
     would corrupt the token on apply;
   - the **same token in both** `apiToken` and `dataIngestToken` (they need different scopes);
   - a **placeholder** (`dt0s16.REPLACE_WITH_…`) got applied once → produced
     `404 Token does not exist` / `401 invalid format`.
- **Fix:** switched both tokens to **classic `dt0c01`** —
  `apiToken` from template **"Kubernetes: Dynatrace Operator"** (all operator scopes incl.
  Read API tokens / DataExport / settings r-w / activeGateTokenManagement.create / installer
  download), and a **separate** `dataIngestToken` (`metrics.ingest`, `logs.ingest`,
  `openTelemetryTrace.ingest`). Put under `stringData:` in `secret.yaml`, applied by hand,
  operator restarted. Result: `Tokens: TokenReady`, all conditions `True`, workloads created.

### Problem 3 — "Is ArgoCD reloading the secret?" ✅ clarified (not a bug)
- The Secret is **gitignored and not in the kustomization**, so **ArgoCD never manages or
  reloads it**. It must be applied by hand (`kubectl apply -f secret.yaml`) and the operator
  restarted to re-validate. Confirmed the live secret matched the file (apply had taken),
  so the failures were token *content*, not staleness.

### Problem 4 — Orphaned injection artifacts ⚠️ open (cosmetic)
- `argocd` and `default` namespaces still carry stale labels
  `dynakube.internal.dynatrace.com/instance: brian-goins-playground-agents` (the original
  pre-rename name) plus leftover `dynatrace-bootstrapper-{certs,config}` secrets — never
  cleaned up because the old DynaKubes were deleted while in `Error`.
- Also: `namespaceSelector` is **empty**, so OneAgent is set to inject **cluster-wide**
  (matched `argocd`, `default`; `easytrade` not deployed yet). Consider scoping it.
- Cleanup (safe — operator recreates only where it injects):
  ```bash
  kubectl label ns argocd default dynakube.internal.dynatrace.com/instance-
  kubectl -n argocd  delete secret dynatrace-bootstrapper-certs dynatrace-bootstrapper-config
  kubectl -n default delete secret dynatrace-bootstrapper-certs dynatrace-bootstrapper-config
  ```

### Problem 5 — ImagePullBackOff: tenant registry empty 🔴 OPEN (current blocker)
- See "What's NOT working" above. **Next action:** override image repositories in the
  DynaKube to a registry that hosts the sprint images.

---

## The fix for the current blocker (image pull)

Override the auto-derived image registry in `dynakube.yaml` to a registry that actually
hosts the images for this sprint tenant (internal Dynatrace registry or a public mirror —
confirm the correct one for sprint/internal tenants):

```yaml
spec:
  oneAgent:
    cloudNativeFullStack:
      image: <registry>/oneagent:<tag>
  activeGate:
    image: <registry>/activegate:<tag>
  templates:
    extensionExecutionController:
      imageRef:
        repository: <registry>/dynatrace-eec        # currently anr47691…/linux/dynatrace-eec → 404
        tag: <tag>
    otelCollector:
      imageRef:
        repository: public.ecr.aws/dynatrace/dynatrace-otel-collector   # already works, keep
        tag: latest
```

Then commit + push (ArgoCD syncs the DynaKube). The OTel collector will recover on its own
once the EEC is reachable. Watch:
```bash
kubectl -n dynatrace get dynakube,pods -w
```

---

## Diagnostic commands

```bash
# DynaKube status + all conditions
kubectl -n dynatrace get dynakube
kubectl -n dynatrace get dynakube brian-obs-playground \
  -o jsonpath='{range .status.conditions[*]}{.type}{": "}{.status}{"/"}{.reason}{" — "}{.message}{"\n"}{end}'

# Pods / workloads
kubectl -n dynatrace get pods -o wide
kubectl -n dynatrace get statefulset,daemonset,deploy

# Image pull errors
kubectl -n dynatrace describe pod <pod> | grep -iE "Failed|Back-off|not found"

# Confirm a token is CLASSIC (dt0c01) and validates via the operator's lookup path
A=$(kubectl -n dynatrace get secret brian-obs-playground -o jsonpath='{.data.apiToken}' | base64 -d)
echo "$A" | cut -c1-7            # expect dt0c01 (NOT dt0s16)
curl -s -X POST "https://anr47691.sprint.dynatracelabs.com/api/v2/apiTokens/lookup" \
  -H "Authorization: Api-Token $A" -H "Content-Type: application/json" \
  -d "{\"token\":\"$A\"}" | jq '.scopes'   # 404 "Token does not exist" = it's a platform token (wrong type)

# Tenant registry sanity (auth works but images absent on sprint)
AUTH=$(kubectl -n dynatrace get secret brian-obs-playground-pull-secret \
  -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d \
  | python3 -c "import sys,json;print(json.load(sys.stdin)['auths']['anr47691.sprint.dynatracelabs.com']['auth'])")
curl -s -o /dev/null -w "v2 root: %{http_code}\n" -H "Authorization: Basic $AUTH" \
  "https://anr47691.sprint.dynatracelabs.com/v2/"
curl -s -o /dev/null -w "oneagent manifest: %{http_code}\n" -H "Authorization: Basic $AUTH" \
  "https://anr47691.sprint.dynatracelabs.com/v2/linux/oneagent/manifests/latest"
```

---

## File layout (gitops repo)

```
clusters/gke-playground/
├── setup.md                     # full cluster + ArgoCD bootstrap recipe (+ SDLC section)
├── TROUBLESHOOTING.md           # this file
└── dynatrace/
    ├── dynakube.yaml            # ONE full-stack DynaKube (NO secret) — committed, ArgoCD-synced
    ├── secret.yaml              # the token Secret — GITIGNORED, applied by hand, NOT in ArgoCD
    ├── secret.example.yaml      # template (stringData; name brian-obs-playground)
    ├── kustomization.yaml       # references dynakube.yaml only
    └── .gitignore               # blocks secret.yaml + *.secret.yaml
apps/
├── dynatrace-operator.yaml      # Helm chart, targetRevision 1.8.1, csidriver + installCRD
├── dynatrace-dynakube.yaml      # points at clusters/gke-playground/dynatrace; prune+selfHeal
└── easytrade.yaml               # points at bmg24/easytrade-config (status Unknown — not deployed)
bootstrap/root-app.yaml          # app-of-apps
projects/playground.yaml         # ArgoCD AppProject
```

---

## SDLC / pipeline-observability events (separate track — not blocked by the above)

- Reusable action: `gitops/.github/actions/dt-sdlc-event` → POSTs normalized
  `SDLC_EVENT`s to `https://anr47691.sprint.dynatracelabs.com/platform/ingest/v1/events.sdlc`.
- Workflow: `easytrade-config/.github/workflows/sdlc.yml` — full lifecycle.
- **Needs repo secrets on `bmg24/easytrade-config`:**
  - `DT_URL` = `https://anr47691.sprint.dynatracelabs.com`
  - `DT_SDLC_TOKEN` = access token, scope **`openpipeline.events_sdlc`** (different from operator tokens)
- Verify in tenant (Notebooks → DQL):
  ```dql
  fetch events, from: now()-1h
  | filter event.kind == "SDLC_EVENT" and event.provider == "github.com"
  | sort timestamp desc
  | fields timestamp, event.type, event.status, cicd.pipeline.run.id, artifact.version
  ```
- Design detail: `gitops/docs/sdlc-events.md`. Note `dynatrace-oss/dynatrace-github-action`
  is intentionally NOT used (it only writes classic Events v2, not `SDLC_EVENT`).

---

## Next steps (checklist)

- [ ] **Override image repositories** in `dynakube.yaml` (oneAgent / activeGate / EEC) to a
      registry that hosts the sprint images; keep OTel on `public.ecr.aws`. Commit + push;
      let ArgoCD sync; confirm pods leave `ImagePullBackOff`.
- [ ] Confirm DynaKube leaves `Deploying` → `Running`; ActiveGate + OneAgent + EEC + OTel all `Ready`.
- [ ] Confirm cluster `easytrade-playground` + hosts report in the tenant.
- [ ] Clean up orphaned `brian-goins-playground-agents` namespace labels + bootstrapper secrets
      in `argocd`/`default`; add a `namespaceSelector` to scope OneAgent injection.
- [ ] Deploy easytrade (ArgoCD app currently `Unknown`); confirm OneAgent injection; grab
      `frontendreverseproxy` EXTERNAL-IP (logins `demouser/demopass`).
- [ ] Add `DT_URL` + `DT_SDLC_TOKEN` to `easytrade-config`; trigger SDLC workflow; verify via DQL.
- [ ] Optional: scale-to-zero / nightly teardown for cost.

---

## Reference docs

- Operator token scopes & permissions — <https://docs.dynatrace.com/docs/ingest-from/setup-on-k8s/deployment/tokens-permissions>
- Platform tokens (IAM) — <https://docs.dynatrace.com/docs/manage/identity-access-management/access-tokens-and-oauth-clients/platform-tokens>
- DynaKube parameters reference (image overrides) — <https://docs.dynatrace.com/docs/ingest-from/setup-on-k8s/reference/dynakube-parameters>
- Full-stack observability setup — <https://docs.dynatrace.com/docs/ingest-from/setup-on-k8s/deployment/full-stack-observability>
- Operator releases — <https://github.com/Dynatrace/dynatrace-operator/releases>
- Ingest SDLC events — <https://docs.dynatrace.com/docs/deliver/pipeline-observability-sdlc-events/sdlc-events>
