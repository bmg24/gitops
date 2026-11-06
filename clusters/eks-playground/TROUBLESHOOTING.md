# Dynatrace on the EKS playground: status & troubleshooting

> **Purpose:** Self-contained context for the Dynatrace deployment on the playground cluster.
> The cluster moved from GKE to EKS on 2026-11-05 and the tenant changed to `lbn6737h`.
> The "How we got here" problems below happened on GKE with the old tenant; the token,
> image and ArgoCD lessons still apply. **Last updated:** 2026-11-05.

---

## Status: EKS cluster up, Dynatrace not yet deployed

- Cluster `brian-eks-playground` is running with ArgoCD v3.6.1 installed.
- The root app is not applied yet; once it is, ArgoCD deploys the Operator (1.11.1), the
  DynaKube and easytrade (see `bootstrap.md`).
- Last known healthy Dynatrace deployment was on GKE (2026-08-08): one DynaKube `Running`,
  ActiveGate 1/1, OneAgent on every node, all conditions `True`.

Lessons that carry over (details below):

1. **One DynaKube**, full-stack (Problem 1).
2. **Classic `dt0c01` tokens**, not platform `dt0s16` tokens (Problem 2).
3. **Public ECR images**; the sprint tenant's built-in registry hosts no agent images, and
   EEC-dependent features stay removed (Problem 3).

The GitHub Actions **SDLC events** track is independent of the cluster.

---

## EKS notes (2026-11-05)

| Issue | What happened | Fix / rule |
|---|---|---|
| `eksctl create cluster` denied | An AWS Organizations SCP explicitly denied `eks:CreateCluster` for our role via eksctl/CloudFormation | Cluster created in the console instead. Don't try to route around the SCP. |
| VPC limit | The shared account is at its VPC quota, so a new VPC failed | Reuse the public subnets of `brian-test-vpc` (shared with other people's clusters: never modify or delete it) |
| PVCs stuck `Pending` | SCP statement `DenyCreationOfGP2Volumes` blocks gp2 volumes; EKS set no default StorageClass | Apply `storageclass-gp3.yaml` (default gp3). Never name `gp2` in a chart. |
| EKS Auto Mode | Console "Quick configuration" enables Auto Mode | Always use custom configuration with Auto Mode **off**; OneAgent needs managed EC2 nodes |
| Kubernetes 1.37 | Console picked 1.37 | Supported by Dynatrace (Operator 1.6.x min, 1.9.0+ recommended; repo pins 1.11.1) and ArgoCD 3.6 |

---

## Current configuration (in Git)

| Component | Detail |
|---|---|
| `dynatrace-operator` | Helm chart **1.11.1** from `public.ecr.aws/dynatrace`, CSI driver enabled, ArgoCD wave 0 |
| DynaKube `brian-obs-playground` | `dynatrace.com/v1beta6`, wave 5 |
| OneAgent | `public.ecr.aws/dynatrace/dynatrace-oneagent:1.339.55.20260615-110349` |
| CodeModules | `public.ecr.aws/dynatrace/dynatrace-codemodules:1.339.55.20260615-110349` |
| ActiveGate | `public.ecr.aws/dynatrace/dynatrace-activegate:1.339.39.20260605-153224` |

> Public ECR's newest GA was `1.339.x` when pinned; sprint tenants run newer internal builds.
> The slightly older public agent connects and reports normally.

---

## Environment facts

| Thing | Value |
|---|---|
| Dynatrace tenant | `lbn6737h` (sprint, INTERNAL) |
| Ingest/API host | `https://lbn6737h.sprint.dynatracelabs.com` |
| Apps/UI host | `https://lbn6737h.sprint.apps.dynatracelabs.com` |
| AWS account / region | `446130280781` / `us-east-1` |
| EKS cluster | `brian-eks-playground`, K8s 1.37, managed node group `ng-main` (2 × m6i.xlarge, 0 to 3) |
| K8s namespace | `dynatrace` |
| DynaKube CRD apiVersion | `dynatrace.com/v1beta6` |
| GitHub user/org | `bmg24` (personal) |
| Repos | `bmg24/gitops`, `bmg24/easytrade-1` (public so ArgoCD can read them) |

### DynaKube object
- **Single** `brian-obs-playground`: `cloudNativeFullStack` OneAgent
  (`hostGroup: eks-playground`, `networkZone: eks-playground`) + one ActiveGate with
  `kubernetes-monitoring` + `routing` + `debugging` (1 replica, 10Gi), explicit public-ECR
  image overrides.
- **No EEC-dependent features** (`extensions`, `extensionExecutionController`, `otelCollector`,
  `telemetryIngest`, `logMonitoring`): removed; see "Problem 3" below.
- References secret `brian-obs-playground` (`spec.tokens`).

### Tokens — IMPORTANT
- The operator requires **classic API tokens (`dt0c01`)**, NOT platform tokens (`dt0s16`).
  This was the central, hard-won finding: see Problem 2. (Platform-token support was under
  internal review as of 2026-06-16; revisit if guidance changes.)
- `apiToken` (operator, template "Kubernetes: Dynatrace Operator") + a **separate**
  `dataIngestToken` (`metrics.ingest`, `logs.ingest`, `openTelemetryTrace.ingest`).
- Secret created by hand with `kubectl` (see `bootstrap.md`), **not** in Git and **not**
  managed by ArgoCD (kustomization references `dynakube.yaml` only). Changing it requires
  recreating the Secret and restarting the operator.

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

### Problem 5 — ArgoCD `applicationset-controller` restart loop (`BackOff`) ✅ fixed

Dynatrace problems **P-26080324** and **P-26080850** flagged the ArgoCD
`argocd-applicationset-controller` deployment as unhealthy with repeated `BackOff`
events.

- **Symptom:** pod showed very high historical restart count and repeated backoff.
- **Root cause:** the `ApplicationSet` CRD was missing:
  `applicationsets.argoproj.io` did not exist, and controller logs showed:
  `failed to get restmapping: no matches for kind "ApplicationSet" in version "argoproj.io/v1alpha1"`.
- **Fix applied (2026-08-08):** restored only the missing CRD at the currently-running
  ArgoCD version:
  ```bash
  kubectl apply --server-side -f \
    https://raw.githubusercontent.com/argoproj/argo-cd/v3.4.3/manifests/crds/applicationset-crd.yaml
  ```
  (`--server-side` avoids the CRD annotation size limit from client-side apply.)
- **Validation:** `argocd-applicationset-controller` reached `1/1 Running`; restart
  count stayed unchanged over 5+ minutes; logs progressed from restmapping errors to
  `Starting Controller` / `Starting workers`.
- **Classification:** config/drift issue (missing CRD), **not** node capacity pressure.

### Problem 6 — AppProject not GitOps-managed → new namespaces rejected ✅ fixed (2026-08-08)

Deploying the astroshop OTel path, the `otel-collector` ArgoCD app was rejected with:
```
application destination server 'https://kubernetes.default.svc' and namespace 'otel-collector'
do not match any of the allowed destinations in project 'playground'
```
- **Root cause:** the app-of-apps root (`bootstrap/root-app.yaml`) syncs **only** `path: apps`, so
  `projects/` was never reconciled by ArgoCD. The live `playground` AppProject was frozen at
  whatever was hand-applied at bootstrap; adding `otel-collector` (and `astroshop`) to
  `projects/playground.yaml` in git had no effect on the cluster. The error surfaces as an
  Application **condition** (`InvalidSpecError`), not a stdout log — see it via
  `kubectl -n argocd get app otel-collector -o jsonpath='{.status.conditions}'` or the UI tile.
  The controller's reconciliation trail *is* in Dynatrace logs: `fetch logs | filter
  k8s.namespace.name == "argocd" | filter contains(content, "otel-collector")`.
- **Immediate unblock:** `kubectl apply -f projects/playground.yaml` (idempotent).
- **Durable fix (2026-08-08):** `apps/projects.yaml` (new) — an Application under the built-in
  `default` project, sync-wave **-5**, that GitOps-manages `projects/`. AppProject edits now
  auto-sync. `prune: false` on it intentionally (pruning a project would orphan all its apps).
  Bootstrap still seeds `projects/playground.yaml` once by hand; see `bootstrap.md` step 5.
- **Classification:** GitOps coverage gap (project outside the managed path), not a runtime fault.

---

## Open / optional follow-ups

- The GKE-era follow-ups (orphaned injection labels, empty `namespaceSelector`, easytrade
  `Unknown`) don't apply to the fresh EKS cluster: injection is now scoped by the
  `dynatrace-injection` namespace label, and easytrade deploys from its Helm chart.
- Problem 4 (remote kustomize base) is historical: easytrade now deploys from the
  `bmg24/easytrade-1` Helm chart.
- Lower the ActiveGate memory (10Gi) once real usage is known; nodes have ~14.4 GiB allocatable.
- Verify the DynaKube setting that makes the cluster show as `eks-playground` in Dynatrace
  (decision D4) against Operator 1.11 docs before relying on it.
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
curl -s -X POST "https://lbn6737h.sprint.dynatracelabs.com/api/v2/apiTokens/lookup" \
  -H "Authorization: Api-Token $A" -H "Content-Type: application/json" \
  -d "{\"token\":\"$A\"}" | jq '.scopes'   # 404 = a platform token (wrong type)

# List available public-ECR tags when bumping versions
skopeo list-tags docker://public.ecr.aws/dynatrace/dynatrace-oneagent   # or browse gallery.ecr.aws/dynatrace
```

---

## File layout (gitops repo)

```
clusters/eks-playground/
├── cluster-contract.md          # what the cluster must provide
├── bootstrap.md                 # connect, token Secret, ArgoCD, root app (+ SDLC section)
├── BUILD_LOG.md                 # how brian-eks-playground was built + teardown
├── AUDIT.md                     # audit of pre-existing AWS resources (2026-11-05)
├── storageclass-gp3.yaml        # default StorageClass (gp2 blocked by org policy)
├── TROUBLESHOOTING.md           # this file
└── dynatrace/
    ├── dynakube.yaml            # ONE full-stack DynaKube, public-ECR image overrides
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
