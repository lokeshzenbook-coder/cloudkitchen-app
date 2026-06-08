# Phase 7 — HTTPS with Let's Encrypt + Path-Routed Sub-Apps (GCP)

**Goal:** Install **cert-manager**, mint a free **Let's Encrypt** certificate
for your domain, flip the chart to **HTTPS** (`websecure` entryPoint), and
move ArgoCD + Grafana + Prometheus + Alertmanager over to HTTPS too — so
you finish with:

```
https://vijaygiduthuri.in                — the app UI
https://vijaygiduthuri.in/argocd         — ArgoCD UI
https://vijaygiduthuri.in/grafana        — Grafana
https://vijaygiduthuri.in/prometheus     — Prometheus
https://vijaygiduthuri.in/alertmanager   — Alertmanager
```

**Time:** ~15 minutes (most of it Let's Encrypt issuing the cert).

This is the **GCP counterpart** of [docs/eks/07-https-letsencrypt-and-routes.md](../eks/07-https-letsencrypt-and-routes.md).
Same architecture; only the install paths + service names differ.

---

## What & why

After Phase 5 you have HTTP working. Modern browsers warn on HTTP and many
auth flows (Grafana login form autofill, ArgoCD OAuth, etc.) refuse to run
over HTTP. So HTTPS is the next step.

**cert-manager** is the standard Kubernetes operator that talks to ACME
servers (Let's Encrypt, ZeroSSL, internal CA) to obtain + renew TLS
certificates automatically. We use Let's Encrypt's free public service
with the **HTTP-01** challenge: cert-manager creates a temporary IngressRoute
serving a token at `http://<your-domain>/.well-known/acme-challenge/…`,
Let's Encrypt fetches it to prove you control the domain, and emits a
90-day cert (renewed automatically at day 75).

```
        ┌──────────────────────────────────────────────────────┐
        │  cert-manager (cert-manager ns)                       │
        │   ┌─────────────────────────────────────────┐         │
        │   │  ClusterIssuer "letsencrypt-staging"     │         │
        │   │  ClusterIssuer "letsencrypt-prod"        │         │
        │   └─────────────────────────────────────────┘         │
        │                       │                              │
        │                       ▼                              │
        │   Certificate "cloudkitchen-tls"  in cloudkitchen ns │
        │   → produces Secret "cloudkitchen-tls"                │
        └──────────────────────┬───────────────────────────────┘
                               │
                               ▼  (the Secret holds tls.crt + tls.key)
        Traefik IngressRoute (chart-rendered, ingress.tls=true)
        references secretName: cloudkitchen-tls  →  HTTPS!
```

---

## ⚠️ Heads-up — the Certificate is created manually

The cert-manager `Certificate` resource is **NOT** in the cloudkitchen
helm chart on purpose. The cert's lifecycle is tied to your **domain**,
not the chart's release lifecycle — keeping it separate makes it safer
to re-deploy the chart without churning the cert (Let's Encrypt
rate-limits real-cert issuance to 5/week per registered domain).

So in this phase we **apply the Certificate manifest by hand** from
`security/cert-manager/certificate.yaml`.

---

## ✅ Prerequisites

| Check | How |
|---|---|
| Phase 5 done (your domain resolves to the LB IP) | `dig +short vijaygiduthuri.in` → returns your LB IP (e.g. `136.112.45.103`) |
| Phase 6 done (monitoring + logging) | `kubectl -n monitoring get pods` healthy |
| Port 80 reachable from the internet | Used by Let's Encrypt for HTTP-01 challenge. Traefik listens on 80 by default. |
| `kubectl` + `helm` work | `kubectl get nodes` and `helm version` succeed |

> 💡 **Why port 80 has to stay open** — even after we flip the app to
> HTTPS, we keep port 80 reachable so cert-manager can solve the HTTP-01
> challenge during renewals. Traefik will redirect normal traffic from
> `:80` → `:443` (configured in Step 4) while still passing
> `/.well-known/acme-challenge/…` through to cert-manager's solver.

---

## Step 1 — Install cert-manager

Two equally valid paths. Pick one.

### Option A — As an ArgoCD Application (matches Phase 6 pattern)

The repo already ships [argocd/apps/app-cert-manager.yaml](../../argocd/apps/app-cert-manager.yaml).
**One edit needed first**: it installs cert-manager into the `ingress`
namespace, but our Traefik lives in `traefik` (not `ingress`) and cert-manager
typically lives in its own `cert-manager` namespace. Open the file and change:

```yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: cert-manager          # was: ingress
```

Then apply:

```bash
kubectl apply -f argocd/apps/app-cert-manager.yaml
```

ArgoCD auto-syncs, creates the `cert-manager` namespace, and installs the
chart. Wait until all 3 cert-manager Deployments are Ready:

```bash
kubectl -n cert-manager rollout status deploy/cert-manager
kubectl -n cert-manager rollout status deploy/cert-manager-cainjector
kubectl -n cert-manager rollout status deploy/cert-manager-webhook
```

### Option B — Direct `helm install` (simpler one-off, no ArgoCD)

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update jetstack

helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager --create-namespace \
  --set crds.enabled=true \
  --set replicaCount=2 \
  --set global.leaderElection.namespace=cert-manager \
  --wait --timeout=5m
```

Either way: 3 cert-manager pods Running + 5 CRDs installed
(`certificates`, `certificaterequests`, `clusterissuers`, `issuers`,
`orders`, `challenges`).

```bash
kubectl get crd | grep cert-manager.io
kubectl -n cert-manager get pods
```

---

## Step 2 — Apply the ClusterIssuers (staging + prod)

Both Issuers are defined in **one file** in the repo:
[security/cert-manager/clusterissuer.yaml](../../security/cert-manager/clusterissuer.yaml).

- `letsencrypt-staging` → Let's Encrypt **staging** API (not browser-trusted,
  but no rate limits — use this until issuance works)
- `letsencrypt-prod` → Let's Encrypt **production** API (real browser-trusted cert)

Both use the **HTTP-01** solver via the `traefik` IngressClass.

### 2a — Set your email (required by Let's Encrypt for expiry notices)

```bash
EMAIL="vijaygiduthuri67@gmail.com"   # 👈 use your real email
sed -i "s/admin@cloudkitchen.example.com/${EMAIL}/g" \
  security/cert-manager/clusterissuer.yaml
```

### 2b — Apply

```bash
kubectl apply -f security/cert-manager/clusterissuer.yaml
```

### 2c — Verify both are Ready

```bash
kubectl get clusterissuer
# NAME                  READY   AGE
# letsencrypt-staging   True    30s
# letsencrypt-prod      True    30s
```

If they show `READY=False`, check the conditions:
```bash
kubectl describe clusterissuer letsencrypt-staging | tail -20
```
Common cause: cert-manager pods not Ready yet (Step 1 wait).

---

## Step 3 — Create the Certificate (start with STAGING)

The repo ships [security/cert-manager/certificate.yaml](../../security/cert-manager/certificate.yaml)
— but the defaults reference `cloudkitchen.example.com` and a subdomain
list we don't need. Edit it to match your setup:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: cloudkitchen-tls
  namespace: cloudkitchen
spec:
  secretName: cloudkitchen-tls         # the Secret the chart will reference
  duration: 2160h                       # 90 days
  renewBefore: 360h                     # renew 15 days before expiry
  privateKey:
    algorithm: ECDSA
    size: 256
    rotationPolicy: Always
  issuerRef:
    name: letsencrypt-staging           # 👈 start with STAGING (no rate limits)
    kind: ClusterIssuer
    group: cert-manager.io
  commonName: vijaygiduthuri.in         # 👈 YOUR domain
  dnsNames:
    - vijaygiduthuri.in                 # 👈 YOUR domain
```

The chart you wrote earlier ships a multi-domain example. Replace its
`dnsNames:` block with just your single host and `commonName:` to match.

Apply:
```bash
kubectl apply -f security/cert-manager/certificate.yaml
```

Watch:
```bash
kubectl -n cloudkitchen get certificate cloudkitchen-tls -w
# Wait for READY=True. Typically <60s.

# If it's stuck:
kubectl -n cloudkitchen describe certificate cloudkitchen-tls | tail -30
kubectl -n cloudkitchen get challenges            # the active HTTP-01 challenge
```

### Switch to PROD once staging works

Once `READY=True` on staging, flip the issuer to **prod** so browsers
trust the cert:

```bash
sed -i 's|letsencrypt-staging|letsencrypt-prod|' security/cert-manager/certificate.yaml
kubectl apply -f security/cert-manager/certificate.yaml

# cert-manager re-issues with the prod issuer (the old staging cert in the
# Secret is replaced). Watch for READY=True again:
kubectl -n cloudkitchen get certificate cloudkitchen-tls -w
```

Verify the Secret now holds a prod cert:

```bash
kubectl -n cloudkitchen get secret cloudkitchen-tls \
  -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -issuer
# Issuer: C=US, O=Let's Encrypt, CN=R10        ← real prod cert
```

(Staging issuer would show `O=(STAGING) Let's Encrypt`.)

---

## Step 4 — Flip the chart to HTTPS

Edit [helm/cloudkitchen/values.yaml](../../helm/cloudkitchen/values.yaml) — change exactly two fields in the `ingress:` block:

```yaml
ingress:
  enabled: true
  tls: true                       # 👈 was: false
  host: vijaygiduthuri.in          # (unchanged from Phase 5)
  lbIP: 136.112.45.103              # (unchanged)
  entryPoint: websecure            # 👈 was: web
  tlsSecretName: cloudkitchen-tls   # (unchanged — points at the Secret from Step 3)
  clusterIssuer: letsencrypt-prod   # (unchanged)
```

Commit + push:
```bash
git add helm/cloudkitchen/values.yaml
git commit -m "phase 7: flip cloudkitchen ingress to HTTPS"
git push origin main
```

The pipeline rebuilds + pushes images and bumps tags (a `helm/**`
change triggers the full CI per our consolidated workflow — see Phase 3).
Then ArgoCD picks up the new values.yaml and reconciles. To skip the
3-minute poll wait:

```bash
kubectl -n argocd annotate app cloudkitchen \
  argocd.argoproj.io/refresh=hard --overwrite
```

Verify the IngressRoute now has TLS:

```bash
kubectl -n cloudkitchen get ingressroute cloudkitchen -o yaml \
  | grep -A 2 "tls:"
# Should show:  tls:
#                  secretName: cloudkitchen-tls
```

### Smoke test

```bash
curl -sI "https://vijaygiduthuri.in/" | head -1
# Expect: HTTP/2 200

curl -s "https://vijaygiduthuri.in/api/restaurants" | head -c 200 ; echo
# Expect: JSON array of restaurants
```

🎉 The app is now on **HTTPS with a real browser-trusted certificate**.

> ⚠️ HTTP-to-HTTPS redirect (port 80 → 443) is NOT automatic with the
> current chart. If you want plain `http://vijaygiduthuri.in/` to bounce
> to https, add this Traefik redirect Middleware and reference it on the
> route's `web` entrypoint — out of scope for this phase. Optional.

---

## Step 5 — Move ArgoCD / Grafana / Prometheus / Alertmanager to HTTPS

Right now those four UIs are reachable at `http://vijaygiduthuri.in/argocd/`,
`/grafana/`, `/prometheus/`, `/alertmanager/`. To move them to HTTPS, two
things change:

1. Each IngressRoute switches from `entryPoints: [web]` → `[websecure]` and
   gets a `tls: { secretName: cloudkitchen-tls }` block.
2. Because each IngressRoute lives in a **different namespace**
   (`monitoring`, `argocd`) and Traefik can only read the TLS Secret from
   the IngressRoute's own namespace, **we duplicate the Secret into each
   target namespace**.

### 5a — Duplicate the TLS Secret into `monitoring` and `argocd`

```bash
kubectl get secret cloudkitchen-tls -n cloudkitchen -o yaml \
  | sed 's/namespace: cloudkitchen/namespace: monitoring/' \
  | kubectl apply -f -

kubectl get secret cloudkitchen-tls -n cloudkitchen -o yaml \
  | sed 's/namespace: cloudkitchen/namespace: argocd/' \
  | kubectl apply -f -

# Verify
kubectl get secret cloudkitchen-tls -n monitoring
kubectl get secret cloudkitchen-tls -n argocd
```

> 💡 **About renewal** — cert-manager only refreshes the **original** Secret
> in the `cloudkitchen` namespace (the one tied to the `Certificate`
> resource). When the cert auto-renews (~day 75), you'll need to re-run
> the two `kubectl get | sed | apply` commands above to refresh the
> duplicates. For a learning project this is fine.
> If you want fully-automatic propagation, install
> [reflector](https://github.com/emberstack/kubernetes-reflector) and
> annotate the source Secret — out of scope here.

### 5b — Edit the 3 observability IngressRoutes

Open each file in [monitoring/ingressroutes/](../../monitoring/ingressroutes/)
and:

- Change `entryPoints: - web` → `entryPoints: - websecure`
- Add a `tls:` block at the bottom of `spec:`

The final shape for `monitoring/ingressroutes/grafana.yaml`:

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: grafana
  namespace: monitoring
  annotations:
    argocd.argoproj.io/sync-options: Prune=false
spec:
  entryPoints:
    - websecure                   # 👈 was: web
  routes:
    - match: Host(`vijaygiduthuri.in`) && PathPrefix(`/grafana`)
      kind: Rule
      services:
        - name: monitoring-grafana
          port: 80
  tls:                            # 👈 NEW block
    secretName: cloudkitchen-tls
```

Apply the same two edits to `prometheus.yaml` and `alertmanager.yaml`,
then `kubectl apply` all three:

```bash
kubectl apply -f monitoring/ingressroutes/
```

### 5c — Update the ArgoCD IngressRoute

The ArgoCD IngressRoute lives in the cluster (created via kubectl in
Phase 4, not in any chart). The easiest way to flip it to HTTPS is to
re-apply a corrected version:

```bash
cat > /tmp/argocd-ingressroute.yaml <<'EOF'
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: argocd
  namespace: argocd
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`vijaygiduthuri.in`) && PathPrefix(`/argocd`)
      kind: Rule
      services:
        - name: argocd-server
          port: 80
  tls:
    secretName: cloudkitchen-tls
EOF
kubectl apply -f /tmp/argocd-ingressroute.yaml
```

No `StripPrefix` middleware — ArgoCD was installed with
`configs.params.server.rootpath=/argocd` (Phase 4), so it already
expects the `/argocd` prefix on incoming requests.

---

## Step 6 — Verify all 5 URLs over HTTPS

```bash
DOMAIN=vijaygiduthuri.in

# 1. App UI
curl -sI "https://$DOMAIN/"                       | head -1
# Expect: HTTP/2 200

# 2. ArgoCD UI
curl -sIL "https://$DOMAIN/argocd/"               | head -1
# Expect: HTTP/2 200 (Argo redirects 307 → 200 with -L)

# 3. Grafana
curl -sIL "https://$DOMAIN/grafana/login"         | head -1
# Expect: HTTP/2 200

# 4. Prometheus
curl -sIL "https://$DOMAIN/prometheus/-/ready"    | head -1
# Expect: HTTP/2 200

# 5. Alertmanager
curl -sL  "https://$DOMAIN/alertmanager/-/ready"
# Expect: "OK"
```

Open each in a browser — you should see the **🔒 padlock** with a
browser-trusted (Let's Encrypt R10/R11 issuer) cert:

- https://vijaygiduthuri.in/                  → React UI
- https://vijaygiduthuri.in/argocd/           → ArgoCD UI (admin / your bootstrap password)
- https://vijaygiduthuri.in/grafana/          → Grafana (admin / prom-operator)
- https://vijaygiduthuri.in/prometheus/       → Prometheus
- https://vijaygiduthuri.in/alertmanager/     → Alertmanager

All five served over **HTTPS with a real browser-trusted certificate**. 🔒

---

## 🐛 Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `Certificate` stuck `READY=False` for >5 min | Let's Encrypt HTTP-01 can't reach `http://vijaygiduthuri.in/.well-known/acme-challenge/…` | Confirm port 80 is open: `curl http://vijaygiduthuri.in/.well-known/acme-challenge/test` should return 404 (not "connection refused"). `kubectl get challenges -A` shows the exact URL Let's Encrypt is probing — curl it from your laptop. |
| Browser shows "NET::ERR_CERT_AUTHORITY_INVALID" or warns | You actually have the staging cert | Edit `security/cert-manager/certificate.yaml` `issuerRef.name` from `letsencrypt-staging` → `letsencrypt-prod`, `kubectl apply`, wait. Verify with the `openssl x509 -noout -issuer` command above. |
| Rate-limited by Let's Encrypt | You requested too many real certs (5/week/domain) | Stay on staging until everything works, then flip to prod **once**. The chart's `tls:` block keys off the Secret, not the cert source — staging vs prod is just `issuerRef`. |
| `kubectl get challenges` shows the challenge stuck "pending" | cert-manager's HTTP-01 solver IngressRoute clashed with something | `kubectl -n cert-manager delete order --all` (forces a fresh attempt). If it keeps failing, check `kubectl -n cert-manager logs deploy/cert-manager` for the specific error. |
| `/grafana` works but the page renders un-styled / blank | TLS secret not present in `monitoring` ns, or the IngressRoute still has `entryPoints: [web]` | Re-run Step 5a (duplicate Secret) + re-verify the edits in 5b landed: `kubectl -n monitoring get ingressroute grafana -o jsonpath='{.spec.entryPoints}'` should print `[websecure]`. |
| `/argocd` returns "ERR_TOO_MANY_REDIRECTS" | ArgoCD's `server.insecure=true` got lost during a chart upgrade — it now tries to redirect from `/argocd` to `https://localhost/argocd` | `helm upgrade argocd argo/argo-cd -n argocd --reuse-values --set 'configs.params.server\.insecure=true' --set 'configs.params.server\.rootpath=/argocd'`, then `kubectl -n argocd rollout restart deploy argocd-server`. |
| Certificate renews but `/grafana` and `/argocd` still serve the OLD cert | The duplicated Secrets in `monitoring` + `argocd` are stale — cert-manager only refreshed `cloudkitchen/cloudkitchen-tls` | Re-run the two `kubectl get | sed | apply` commands from Step 5a. To automate, install [reflector](https://github.com/emberstack/kubernetes-reflector). |
| `error from server: no matches for kind "Certificate"` | cert-manager CRDs didn't install | `kubectl get crd | grep cert-manager.io` should list 6 CRDs. If empty, re-run Step 1 with `--set crds.enabled=true`. |

---

## 📋 Phase 7 cheatsheet

```bash
# 1. cert-manager
kubectl apply -f argocd/apps/app-cert-manager.yaml          # Option A
# OR: helm install cert-manager jetstack/cert-manager -n cert-manager \
#       --create-namespace --set crds.enabled=true --wait

# 2. ClusterIssuers (set your email first)
sed -i "s/admin@cloudkitchen.example.com/YOUR_EMAIL/g" security/cert-manager/clusterissuer.yaml
kubectl apply -f security/cert-manager/clusterissuer.yaml

# 3. Certificate (start staging, then prod)
# Edit security/cert-manager/certificate.yaml: commonName + dnsNames = your domain
kubectl apply -f security/cert-manager/certificate.yaml
kubectl -n cloudkitchen get cert cloudkitchen-tls -w        # wait READY=True
sed -i 's|letsencrypt-staging|letsencrypt-prod|' security/cert-manager/certificate.yaml
kubectl apply -f security/cert-manager/certificate.yaml
kubectl -n cloudkitchen get cert cloudkitchen-tls -w        # wait READY=True again

# 4. Flip the chart to HTTPS
sed -i 's/tls: false/tls: true/' helm/cloudkitchen/values.yaml
sed -i 's/entryPoint: web$/entryPoint: websecure/' helm/cloudkitchen/values.yaml
git add helm/cloudkitchen/values.yaml
git commit -m "phase 7: flip cloudkitchen ingress to HTTPS"
git push origin main
kubectl -n argocd annotate app cloudkitchen argocd.argoproj.io/refresh=hard --overwrite

# 5. Sub-app routes to HTTPS
# 5a — copy the TLS Secret across namespaces
for NS in monitoring argocd; do
  kubectl get secret cloudkitchen-tls -n cloudkitchen -o yaml \
    | sed "s/namespace: cloudkitchen/namespace: ${NS}/" \
    | kubectl apply -f -
done
# 5b — edit monitoring/ingressroutes/*.yaml (web → websecure, add tls: block), apply:
kubectl apply -f monitoring/ingressroutes/
# 5c — re-apply the ArgoCD IngressRoute with websecure + tls (see Step 5c)

# 6. Verify
for URL in / /argocd/ /grafana/login /prometheus/-/ready /alertmanager/-/ready; do
  printf "%-30s  -> " "$URL"
  curl -sIL -o /dev/null -w "HTTP %{http_code}\n" "https://vijaygiduthuri.in$URL"
done
```

---

## 🎉 What you accomplished

- ✅ **cert-manager** running with staging + prod Let's Encrypt issuers
- ✅ A real **browser-trusted TLS certificate** in the cluster, auto-renewing every 75 days
- ✅ Chart flipped to HTTPS via the existing `ingress.tls` toggle — no chart-template edits needed
- ✅ **All 5 UIs** (app, ArgoCD, Grafana, Prometheus, Alertmanager) reachable under the **same domain** over HTTPS

You now have a **production-shape** GKE deployment:

```
https://vijaygiduthuri.in                📱 the app
https://vijaygiduthuri.in/argocd         🚀 GitOps controller
https://vijaygiduthuri.in/grafana        📊 dashboards
https://vijaygiduthuri.in/prometheus     📈 metrics
https://vijaygiduthuri.in/alertmanager   🚨 alerts
```

---

## 🧹 Tearing it all down

When you finish the learning journey:

```bash
# 1. (Optional) helm uninstalls — quick
helm uninstall cloudkitchen   -n cloudkitchen
helm uninstall argocd         -n argocd
helm uninstall traefik        -n traefik
helm uninstall cert-manager   -n cert-manager
# (monitoring + logging Apps will go away with the cluster — they're
#  ArgoCD-managed so you can ALSO `kubectl -n argocd delete app monitoring logging`)

# 2. Delete StatefulSet PVCs (NOT auto-deleted)
kubectl -n cloudkitchen delete pvc -l app=postgres
kubectl -n cloudkitchen delete pvc -l app=nats
kubectl -n monitoring  delete pvc --all
kubectl -n logging     delete pvc --all

# 3. Release the static LB IP (otherwise GCP keeps billing ~$7/month)
gcloud compute addresses delete traefik-lb-ip --region=us-central1

# 4. Destroy the GCP infra
cd gcp-terraform && terraform destroy
```

Billing drops to ~$0/day within ~10 minutes once `terraform destroy` finishes.

---

🏁 **You did it.** The full DevOps lifecycle in seven phases on GKE:
**infra → ingress → CI → CD → DNS → observability → HTTPS**.

Go put this on your resume.
