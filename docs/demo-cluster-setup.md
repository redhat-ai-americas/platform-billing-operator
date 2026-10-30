# Demo Cluster Setup Guide

Stand up a cluster from scratch for the platform-billing-operator and run the demos. Two paths: local kind or OpenShift.

> **Status: first draft, assembled from the demo scripts, manifests and recorded deployments. Nobody has followed it end to end on a fresh cluster yet.** Steps marked *unverified* were not taken from a tested source. If you are the first to walk it, fix what you find in place and remove this note when it holds up.

## Prerequisites

**Common**:
- kubectl 1.28+ or oc 4.14+
- Podman (Makefile default) or Docker
- Go 1.22+ (operator, billctl)
- Rust 1.75+ with `wasm32-wasip1` (capture filter): `rustup target add wasm32-wasip1`
- jq (demo scripts)
- Helm 3.x (kind only)

**OpenShift**:
- 4.14+ cluster, cluster-admin access
- 16+ vCPU, 32 GB RAM
- x86_64 architecture (model serving requirement)

**Cluster context**: Set these at the start to match your environment:
```bash
export CTX="kind-platform-billing-dev"  # or your OpenShift context
export NS="billing-demo"
export OPERATOR_NS="platform-billing-operator-system"
export POSTGRES_NS="billing-system"
```

---

## Path A: Local Development (kind)

### 1. Create cluster and dependencies

```bash
make dev-cluster
```

Installs: kind cluster `platform-billing-dev`, cert-manager v1.16.1, Gateway API CRDs v1.4.1, Istio 1.29.1, Kuadrant v1.5.2, demo namespace `billing-demo` with Gateway `billing-demo-gateway`. Wait ~5 min for all pods ready.

Verify:
```bash
kubectl --context=$CTX get pods -n kuadrant-system
kubectl --context=$CTX get gateway -n $NS
```

### 2. Deploy Postgres

```bash
./hack/setup-postgres.sh $CTX
```

Deploys Postgres 16 to `billing-system`. Migrations applied by ledger-svc on startup.

### 3. Build and deploy operator

The Makefile uses `CONTAINER_TOOL=podman` by default. **Mac users**: kind loads the correct arch automatically; no `--platform` flag needed for kind. For OpenShift deployment from a Mac, see Path B.

```bash
make docker-build IMG=controller:latest
make deploy IMG=controller:latest
```

Verify:
```bash
kubectl --context=$CTX get pods -n $OPERATOR_NS
```

### 4. Run demos

```bash
make demo-m0  # Plan → TRLP/RLP/AuthPolicy compilation
make demo-m1  # Capture + ledger ingest
make demo-m2  # Rating + wallet
make demo-m3  # Allowance enforcement
make demo-m4  # Stripe integration (needs test creds)
make demo-m5  # Reconciliation + budgets + invoicing
```

GPU demos (fake GPUs, no hardware):
```bash
./hack/setup-kueue.sh $CTX
make demo-g0 demo-g1 demo-g2
```

### 5. Cleanup

```bash
kind delete cluster --name platform-billing-dev
```

---

## Path B: OpenShift Deployment

### Known working stack

Tested with RHCL 1.4.2 (Kuadrant v1.5.1, wasm-shim v0.14.1) + RHOAI 3.4.2 (vLLM 0.13.0+rhai19, MaaS enabled) + OSSM 3.1.0. GPU Operator v25.3.4 optional for GPU demos.

**Known incompatibility**: Some RHOAI 3.x versions ship Authorino v1.4.2, conflicting with RHCL's v1.1.3. Symptom: RHCL operator CrashLoopBackOff. Workaround: Install RHCL before RHOAI, or use the reference ArgoCD deployment (rhoai-argo `maas` branch) which sequences correctly.

### 1. Install base stack

Install via OperatorHub: (1) RHCL, (2) RHOAI with `DataScienceCluster` MaaS enabled, (3) OSSM. Verify gateway:
```bash
oc --context=$CTX get gateway data-science-gateway -n openshift-ingress
```

Gateway class should be `data-science-gateway-class`, controller `openshift.io/gateway-controller/v1`.

Verify DevPortal CRDs:
```bash
oc --context=$CTX get crd apiproducts.devportal.kuadrant.io
```

### 2. Deploy Postgres

Apply the existing manifest:
```bash
oc --context=$CTX apply -f config/openmeter/postgres.yaml
```

**Before applying**: Edit line 5 to change namespace from `billing-system` to your target namespace if different.

**Unverified alternative**: Use `hack/setup-postgres.sh $CTX` (written for kind; OpenShift compatibility untested).

### 3. Build and deploy operator

**Mac users**: Podman build requires `--platform linux/amd64` for OpenShift x86_64 clusters:

```bash
podman build --platform linux/amd64 -t quay.io/$YOUR_REGISTRY/platform-billing-operator:v0.0.1 .
podman push quay.io/$YOUR_REGISTRY/platform-billing-operator:v0.0.1
make deploy IMG=quay.io/$YOUR_REGISTRY/platform-billing-operator:v0.0.1
```

**OpenShift BuildConfig alternative** (builds on cluster, no Mac platform concerns):

```bash
oc --context=$CTX new-project billing-system
oc --context=$CTX new-build --binary --name=controller -n billing-system
oc --context=$CTX start-build controller --from-dir=. --follow -n billing-system
oc --context=$CTX tag billing-system/controller:latest $OPERATOR_NS/controller:latest
```

Deploy:
```bash
oc --context=$CTX new-project $OPERATOR_NS
make deploy IMG=image-registry.openshift-image-registry.svc:5000/billing-system/controller:latest
```

**Cross-namespace image pull** (if operator and images in different namespaces):
```bash
oc --context=$CTX create rolebinding controller-image-puller -n billing-system \
  --clusterrole=system:image-puller \
  --serviceaccount=$OPERATOR_NS:controller-manager
```

### 4. Deploy ledger-svc

Edit `config/ledger/deployment.yaml` to set image. **Before applying**:
- Line 20: change `image: ledger-svc:latest` to your image ref (e.g., `image-registry.openshift-image-registry.svc:5000/billing-system/ledger-svc:latest` if using BuildConfig)
- Ensure Secret `billing-ledger-postgres` exists with `DSN` key pointing to your Postgres (created by `config/openmeter/postgres.yaml` or manually)

Apply with RBAC:
```bash
oc --context=$CTX apply -k config/ledger
```

This applies `deployment.yaml`, `service.yaml`, and `rbac.yaml` (Role + RoleBinding for reading PriceBook/Plan/TenantAccount CRs).

### 5. Deploy capture filter

Build:
```bash
cd wasm/capture-filter
cargo build --release --target wasm32-wasip1
```

Edit `config/capture-filter/wasmplugin.yaml` **before applying**:
- Line 5: change namespace from `billing-demo` to `openshift-ingress`
- Line 10: change gateway name from `billing-demo-gateway` to `data-science-gateway`
- Line 11: change `url:`. The repo default is `oci://localhost:5000/capture-filter:latest` (kind OCI registry). For OpenShift, options:
  - Push to OpenShift internal registry: `oci://image-registry.openshift-image-registry.svc:5000/billing-system/capture-filter:latest` (unverified: OCI push steps not documented in repo)
  - HTTP server hosting the wasm (unverified: session summaries mention wasm-server pattern but no manifests exist): create a Deployment + Service serving the `.wasm` file, use `http://wasm-server.billing-system.svc:8080/capture_filter.wasm`

Apply:
```bash
oc --context=$CTX apply -f config/capture-filter/wasmplugin.yaml
```

(Unverified: WasmPlugin delivery methods. The repo's `wasmplugin.yaml` uses OCI. Session summaries mention HTTP wasm-server. Production may require OCI registry setup.)

### 6. Expose a model through the gateway

If RHOAI serves a model, create HTTPRoute + ReferenceGrant. **Example** (unverified: written for this guide, not from tested deployment):

```bash
oc --context=$CTX apply -n <model-namespace> -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: model-route
spec:
  parentRefs:
  - name: data-science-gateway
    namespace: openshift-ingress
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /v1/chat/completions
    backendRefs:
    - name: <model-service-name>
      port: 8000
---
apiVersion: gateway.networking.k8s.io/v1beta1
kind: ReferenceGrant
metadata:
  name: allow-gateway-to-model
spec:
  from:
  - group: gateway.networking.k8s.io
    kind: HTTPRoute
    namespace: openshift-ingress
  to:
  - group: ""
    kind: Service
EOF
```

Replace `<model-namespace>` and `<model-service-name>` with your RHOAI model's namespace and Service name.

### 7. Deploy console (optional)

Copy and edit `config/console/billingconsole-dev3.yaml` (sample tied to a specific cluster):
- Line 8: change `image:` to your registry or BuildConfig ImageStream (e.g., `image-registry.openshift-image-registry.svc:5000/billing-system/billing-console:latest`)
- Line 11: set `route.host:` to your desired hostname or leave blank for auto-generation

Apply:
```bash
cp config/console/billingconsole-dev3.yaml config/console/billingconsole-yourcluster.yaml
# Edit billingconsole-yourcluster.yaml as described above
oc --context=$CTX apply -f config/console/billingconsole-yourcluster.yaml
```

Get console URL:
```bash
oc --context=$CTX get billingconsole default -n billing-system -o jsonpath='{.status.url}'
```

### 8. Run demos

Override demo script defaults:
```bash
export CLUSTER_CONTEXT=$CTX
export SKIP_BUILD=true
./hack/demo-m0.sh
./hack/demo-m1.sh
# etc.
```

For GPU demos, install Kueue:
```bash
oc --context=$CTX apply -f https://github.com/kubernetes-sigs/kueue/releases/download/v0.11.1/manifests.yaml
./hack/setup-kueue.sh $CTX
./hack/demo-g0.sh
```

---

## Verification

**Operator running**:
```bash
kubectl --context=$CTX get pods -n $OPERATOR_NS
```

**Kuadrant CRDs**:
```bash
kubectl --context=$CTX get crd tokenratelimitpolicies.kuadrant.io
```

**Gateway ready**:
```bash
kubectl --context=$CTX get gateway -n $NS          # kind
kubectl --context=$CTX get gateway -n openshift-ingress  # OpenShift
```

**Postgres tables**:
```bash
kubectl --context=$CTX exec -n $POSTGRES_NS deploy/postgres -- \
  psql -U billing -d billing_ledger -c '\dt'
```

Should show: `usage_events`, `charges`, `wallets`, `adjustments`, etc.

**Test Plan compilation**:
```bash
kubectl --context=$CTX apply -n $NS -f config/samples/billing_v1alpha1_plan.yaml
kubectl --context=$CTX get tokenratelimitpolicy -n $NS
```

TRLP `<plan>-trlp` appears within 10s.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| Operator CrashLoopBackOff: "no matches for kind TokenRateLimitPolicy" | Kuadrant CRDs missing | `kubectl --context=$CTX get crd tokenratelimitpolicies.kuadrant.io`. On kind: `make dev-cluster`. OpenShift: install RHCL. |
| No usage events after inference | WasmPlugin not applied or crashed | `kubectl --context=$CTX get wasmplugin -A`. Check Envoy logs: `kubectl --context=$CTX logs -n openshift-ingress -l gateway.networking.k8s.io/gateway-name=data-science-gateway -c istio-proxy \| grep wasm` |
| ledger-svc: "connection refused" | Postgres not ready | `kubectl --context=$CTX get pods,svc -n $POSTGRES_NS` |
| Demo fails: "TRLP not created within 30s" | Operator not reconciling | `kubectl --context=$CTX logs -n $OPERATOR_NS deployment/platform-billing-operator-controller-manager` |
| OpenShift build: "permission denied" | Files have 600 perms (Write tool) | `chmod 644 cmd/**/*.go internal/**/*.go` before `oc start-build` |
| Container crash: `PermissionError` reading files | UBI runs as UID 1001, files are 600 | Same fix as above |

---

## Demo scripts

| Script | Milestone | Validates |
|--------|-----------|-----------|
| `demo-m0.sh` | M0 | Plan → TRLP/RLP/AuthPolicy, drift re-stamp, PriceBook immutability |
| `demo-m1.sh` | M1 | Capture filter SSE parsing, ingest, dedup, disconnect/error, fail-open |
| `demo-m2.sh` | M2 | Rater determinism, wallet, ShowbackOnly, billctl usage/adjust |
| `demo-m3.sh` | M3 | Active → Warning → Frozen → Thawed, AuthPolicy freeze, overrun ≤90s |
| `demo-m4.sh` | M4 | Reload idempotency, Stripe/webhook signature, fraud controls |
| `demo-m5.sh` | M5 | Drift reconciliation, budget thresholds, FOCUS 1.2 export, invoicing |
| `demo-g0.sh` | G0 | GPU events, interval events, unified MaaS + GPUaaS wallet |
| `demo-g1.sh` | G1 | GPU enforcement (Kueue StopPolicy), Drain vs Preempt |
| `demo-g2.sh` | G2 | Reservations (Guaranteed/Preemptible), committed-use charging |

Exit code 0 = all checks passed.

---

## Teardown

**Kind**: `kind delete cluster --name platform-billing-dev`

**OpenShift**:
```bash
oc --context=$CTX delete project $OPERATOR_NS billing-system
oc --context=$CTX delete wasmplugin billing-capture-filter -n openshift-ingress
```

Leave RHOAI, RHCL, and OSSM in place (may be used by other workloads).

---

## Sources

Setup procedures from:
- `README.md`, `docs/getting-started.md`, `docs/architecture.md`
- `hack/setup-kuadrant.sh`, `hack/setup-postgres.sh`, `hack/setup-kueue.sh`
- `hack/demo-m*.sh`, `hack/demo-g*.sh` (environment variables, prerequisites)
- `config/ledger/`, `config/capture-filter/`, `config/console/`, `config/openmeter/postgres.yaml`
- `docs/sessions/m3-phase5-summary.md` (OpenShift deployment)
- Planning repo: `prerequisites.md` (RHCL 1.4.2 + RHOAI 3.4.2 stack), `RECON.md` (R5: vLLM 0.13.0+rhai19)

For console auth modes, cross-namespace RBAC, and BuildConfig details: `docs/getting-started.md`. For Authorino conflict details: planning repo `prerequisites.md` (private).
