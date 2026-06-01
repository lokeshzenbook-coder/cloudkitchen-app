# Phase 5 — DNS + GoDaddy (GCP)

**Goal:** Point your own domain (`vijaygiduthuri.in` in our case) at the
Traefik LoadBalancer IP, update the chart's IngressRoute so it accepts the
new hostname, push through the GitOps loop, and verify the app + ArgoCD UI
are reachable by hostname.

**Time:** ~15 min (5 min for DNS to propagate + 10 min for the chart change
to flow through ArgoCD).

> Where you start: app reachable at `http://<LB_IP>/`.
> Where you end:   app reachable at `http://vijaygiduthuri.in/` and
>                  ArgoCD at `http://vijaygiduthuri.in/argocd/`.

This is the **GCP counterpart** of [docs/eks/05-traefik-dns-and-godaddy.md](../eks/05-traefik-dns-and-godaddy.md).

| Concern               | EKS                                                            | GKE (this doc)                                                              |
| --------------------- | -------------------------------------------------------------- | --------------------------------------------------------------------------- |
| LB endpoint type      | NLB hostname (`a1b2c3.elb.us-east-1.amazonaws.com`) → use CNAME | TCP LB IPv4 (e.g. `35.224.38.103`) → use **A record**                       |
| Static-IP reservation | NLBs are stable by default                                     | GKE LB IPs are **ephemeral** by default; reserve as a static IP (Step 7)    |
| DNS record type       | `CNAME` (apex needs `ALIAS`/flattening)                        | `A` (works at apex with no special handling)                                |

---

## ✅ Prerequisites

| Need                                                | How to check                                                 |
| --------------------------------------------------- | ------------------------------------------------------------ |
| Phase 4 done (app reachable at `http://<LB_IP>/`)   | `kubectl -n cloudkitchen get pods` → 12 pods Running         |
| A domain you control                                | We use **`vijaygiduthuri.in`**, registered at GoDaddy        |
| `dig` and `nslookup` installed locally              | `dig +short google.com` and `nslookup google.com` both work  |

---

## What this phase changes

```
                  ┌───────────────────────────────────────┐
                  │ GoDaddy DNS for vijaygiduthuri.in     │
                  │                                       │
                  │   A   @   <YOUR-LB-IP>                │   ← new record
                  └────────────────┬──────────────────────┘
                                   ▼
                  the browser resolves the hostname,
                  hits the Traefik LB at the same IP,
                  Traefik matches Host(`vijaygiduthuri.in`)
                  on the chart's IngressRoute
                                   │
                                   ▼
        ┌────────────────────────────────────────────────┐
        │ helm/cloudkitchen/values.yaml                   │
        │   ingress.hosts:                                │
        │     - vijaygiduthuri.in       ← your hostname   │
        │     - <YOUR-LB-IP>            ← kept (debug)    │
        └────────────────────────────────────────────────┘
                                   │
                                   ▼  (commit + push → ArgoCD picks up)
        ┌────────────────────────────────────────────────┐
        │ IngressRoute `cloudkitchen` — 10 routes now     │
        │ match the hostname OR the LB IP                 │
        └────────────────────────────────────────────────┘
```

---

## Step 1 — Get the Traefik LoadBalancer IP, then add an A record at GoDaddy

### 1a — Get the LB IP dynamically (no hardcoded values)

Run this on the same machine that has kubectl pointed at your cluster:

```bash
kubectl -n traefik get svc traefik \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
echo   # adds a newline after the IP
```

Example output:
```
35.224.38.103
```

Save it to a shell variable so the other commands in this doc just work:

```bash
LB_IP=$(kubectl -n traefik get svc traefik \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "LB IP: ${LB_IP}"
```

> ⚠️ **If this command prints nothing**, Traefik's `LoadBalancer` Service
> hasn't finished provisioning. Wait 1–2 min and retry. If still empty after
> 5 min, see Phase 2 troubleshooting (firewall / health-check rules).

### 1b — Add the A record at GoDaddy

1. Sign into https://account.godaddy.com/products
2. Click your domain → **DNS** (or **Manage DNS**)
3. Click **Add New Record**
4. Fill the form with the value of `$LB_IP` from Step 1a:

   | Field        | Value             | Notes                                                                                       |
   | ------------ | ----------------- | ------------------------------------------------------------------------------------------- |
   | **Type**     | `A`               | IPv4 Address record. Don't use `AAAA` (IPv6) or `CNAME` (hostname → hostname, not IP).      |
   | **Name**     | `@` (apex) or a label like `cloudkitchen` | `@` makes the bare `vijaygiduthuri.in` resolve. A label makes `<label>.vijaygiduthuri.in`. |
   | **Value**    | The IP from `echo $LB_IP` | Paste it. **Do not** type out an example value — the IP is yours specifically. |
   | **TTL**      | `600` (10 min)    | Keep low during setup so changes propagate fast. Raise to `1 Hour` later.                   |

5. Save.

> 💡 In this guide we used **Name = `@`** so requests to the apex domain
> `vijaygiduthuri.in` resolve. If you'd used `Name = cloudkitchen` instead,
> the URL becomes `cloudkitchen.vijaygiduthuri.in`. Both work; the chart
> accepts a list of hosts (see Step 3) — adjust the hostnames there to
> match what you put in GoDaddy.

---

## Step 2 — Verify DNS propagation

GoDaddy → public resolvers usually takes 1–10 min. Three independent ways
to check, in order of usefulness:

### 2a — `dig` (returns just the IP)

```bash
dig +short @8.8.8.8 vijaygiduthuri.in
# Expected output (a single line):
# 35.224.38.103       (← should equal your $LB_IP)
```

Why `@8.8.8.8`? It forces the query to Google's public DNS (8.8.8.8),
bypassing your ISP's resolver. GoDaddy → Google DNS usually propagates
faster than GoDaddy → your-ISP → your-laptop.

### 2b — `nslookup` (more verbose, friendlier output)

```bash
nslookup vijaygiduthuri.in 8.8.8.8
```

Expected output:
```
Server:		8.8.8.8
Address:	8.8.8.8#53

Non-authoritative answer:
Name:	vijaygiduthuri.in
Address: 35.224.38.103       ← should equal your $LB_IP
```

Both `dig` and `nslookup` ask the same question; `nslookup` formats the
answer more chattily. If they disagree, something's broken.

### 2c — https://dnschecker.org (when the above two don't agree)

Paste your hostname. The site queries DNS servers from ~25 regions around
the world and shows which ones see the new record yet. Useful when:
- `dig` against `@8.8.8.8` works but your browser doesn't (your local
  resolver hasn't refreshed)
- You want to know "have AT&T, Comcast, Airtel resolvers picked it up?"

### 2d — Quick HTTP probe

```bash
curl -sI -o /dev/null -w "%{http_code}\n" "http://vijaygiduthuri.in/"
# Will be 404 until Step 3 lands the chart change — that's expected and OK.
# 404 means "Traefik received the request but no IngressRoute matched the
# Host: header" — which is exactly the problem Step 3 fixes.
```

---

## Step 3 — Update the chart so it accepts the hostname

Two files change. We'll show **the entire file** of each, with the lines
you actually edit clearly marked.

### Step 3a — Edit `helm/cloudkitchen/values.yaml`

**File to edit:** [helm/cloudkitchen/values.yaml](../../helm/cloudkitchen/values.yaml)

**What changes:** the **`ingress:` block** (lines 143–157). Nothing else in
the file changes. The entire file is shown below so you can locate the
block visually; the only edits are inside the `# ====== ↓ EDIT THIS BLOCK ↓ ======`
markers.

```yaml
# =============================================================================
# CloudKitchen umbrella chart — default values
# -----------------------------------------------------------------------------
# Flat, explicit, one block per service. Each microservice block drives its own
# <svc>-deployment.yaml / -service.yaml / -configmap.yaml / -secret.yaml / -hpa.yaml.
# This is THE values file ArgoCD renders with and CI patches (per-service image
# strings) — there are no separate per-environment override files.
# =============================================================================

namespace: cloudkitchen
imageRegistry: us-central1-docker.pkg.dev/project-d31a3358-346c-40e8-bda/cloudkitchen-registry

# -----------------------------------------------------------------------------
# Backend microservices (all listen on :8080, expose /healthz and /readyz)
# -----------------------------------------------------------------------------
authService:
  deploymentname: auth-service-deployment
  servicename: auth-service
  image: us-central1-docker.pkg.dev/.../cloudkitchen-registry/auth-service:<sha>
  replicas: 1
  port: 8080
  schema: auth
  hpa: auth-service-autoscaler
  minReplicas: 1
  maxReplicas: 5

# ... userService / restaurantService / menuService / orderService /
#     paymentService / deliveryService / notificationService — same shape,
#     all unchanged ...

frontend:
  deploymentname: frontend-deployment
  servicename: frontend-service
  image: us-central1-docker.pkg.dev/.../cloudkitchen-registry/frontend:<sha>
  replicas: 1
  port: 8080
  hpa: frontend-autoscaler
  minReplicas: 1
  maxReplicas: 4

# -----------------------------------------------------------------------------
# In-cluster data stores
# -----------------------------------------------------------------------------
postgres:
  name: postgres
  servicename: postgres
  image: postgres:16-alpine
  port: 5432
  storage: 10Gi
  storageClass: standard-rwo
redis:
  name: redis
  servicename: redis
  image: redis:7-alpine
  port: 6379
nats:
  name: nats
  servicename: nats
  image: nats:2.10-alpine
  clientPort: 4222
  monitorPort: 8222
  storage: 5Gi
  storageClass: standard-rwo

db:
  host: postgres
  port: "5432"
  name: cloudkitchen
  sslmode: disable

# ====== ↓ EDIT THIS BLOCK ↓ ============================================
# -----------------------------------------------------------------------------
# Ingress (Traefik) + TLS (cert-manager)
# -----------------------------------------------------------------------------
ingress:
  enabled: true
  # HTTP-only vs HTTPS:
  #   tls: false + entryPoint: web        -> first-day deploy, no cert needed.
  #   tls: true  + entryPoint: websecure  -> HTTPS via cert-manager (Phase 7).
  tls: false

  # 👉 CHANGE THESE LINES
  #   Replace the OLD single 'domain: ...' field with a LIST of hosts.
  #   Order matters slightly: put the production hostname first.
  hosts:
    - vijaygiduthuri.in       # 👉 replace with YOUR hostname
    - 35.224.38.103           # 👉 replace with YOUR LB IP (kept for curl-by-IP debugging)

  entryPoint: web
  tlsSecretName: cloudkitchen-tls
  clusterIssuer: letsencrypt-prod
# ====== ↑ EDIT THIS BLOCK ↑ ============================================

# -----------------------------------------------------------------------------
# Secrets (DEV DEFAULTS ONLY)
# -----------------------------------------------------------------------------
secrets:
  dbUser: cloudkitchen
  dbPassword: cloudkitchen-dev-password
  jwtSecret: changeme-use-a-strong-secret-in-production
  jwtExpiry: 24h
  redisPassword: ""
```

#### Old vs new — just the bit that changes

**Before** (single hostname, hard-coded to the IP):
```yaml
ingress:
  enabled: true
  tls: false
  domain: 35.224.38.103       # 👈 was: a single string
  entryPoint: web
  tlsSecretName: cloudkitchen-tls
  clusterIssuer: letsencrypt-prod
```

**After** (list of hostnames, hostname + IP both accepted):
```yaml
ingress:
  enabled: true
  tls: false
  hosts:                       # 👈 now: a list (one item per accepted hostname)
    - vijaygiduthuri.in
    - 35.224.38.103
  entryPoint: web
  tlsSecretName: cloudkitchen-tls
  clusterIssuer: letsencrypt-prod
```

**Key idea:** `domain` (singular string) became `hosts` (list). The
template in Step 3b iterates over this list to build the Traefik matcher.

#### How to get the right values for the two hosts

```bash
# Put YOUR hostname here:
HOSTNAME='vijaygiduthuri.in'

# LB IP — read it from the cluster, don't hard-code:
LB_IP=$(kubectl -n traefik get svc traefik -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

echo "  hosts:"
echo "    - ${HOSTNAME}"
echo "    - ${LB_IP}"
```

Copy the 3 lines into your values.yaml inside the `ingress:` block.

---

### Step 3b — Edit `helm/cloudkitchen/templates/ingressroute.yaml`

**File to edit:** [helm/cloudkitchen/templates/ingressroute.yaml](../../helm/cloudkitchen/templates/ingressroute.yaml)

The template needs two changes:

1. **Add a helper at the top** that renders the multi-host clause.
2. **Replace every `Host(...)` matcher** in the 10 routes with a call to
   that helper, wrapped in parens.

The entire **before** and **after** files are below. Copy-paste the
"after" version verbatim into your repo.

#### Before

```yaml
{{- if .Values.ingress.enabled }}
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: cloudkitchen
  namespace: {{ .Values.namespace }}
  labels:
    app: cloudkitchen
spec:
  entryPoints:
    - {{ .Values.ingress.entryPoint }}
  routes:
    # --- menu sub-paths under /api/restaurants (higher priority) ---
    - match: Host(`{{ .Values.ingress.domain }}`) && PathRegexp(`^/api/restaurants/[^/]+/(menu|categories|items)`)
      kind: Rule
      priority: 200
      services:
        - name: {{ .Values.menuService.servicename }}
          port: {{ .Values.menuService.port }}
    - match: Host(`{{ .Values.ingress.domain }}`) && PathPrefix(`/api/menu`)
      kind: Rule
      priority: 200
      services:
        - name: {{ .Values.menuService.servicename }}
          port: {{ .Values.menuService.port }}
    # --- one rule per service ---
    - match: Host(`{{ .Values.ingress.domain }}`) && PathPrefix(`/api/auth`)
      kind: Rule
      priority: 100
      services:
        - name: {{ .Values.authService.servicename }}
          port: {{ .Values.authService.port }}
    - match: Host(`{{ .Values.ingress.domain }}`) && PathPrefix(`/api/users`)
      kind: Rule
      priority: 100
      services:
        - name: {{ .Values.userService.servicename }}
          port: {{ .Values.userService.port }}
    - match: Host(`{{ .Values.ingress.domain }}`) && PathPrefix(`/api/restaurants`)
      kind: Rule
      priority: 100
      services:
        - name: {{ .Values.restaurantService.servicename }}
          port: {{ .Values.restaurantService.port }}
    - match: Host(`{{ .Values.ingress.domain }}`) && (PathPrefix(`/api/cart`) || PathPrefix(`/api/orders`))
      kind: Rule
      priority: 100
      services:
        - name: {{ .Values.orderService.servicename }}
          port: {{ .Values.orderService.port }}
    - match: Host(`{{ .Values.ingress.domain }}`) && PathPrefix(`/api/payments`)
      kind: Rule
      priority: 100
      services:
        - name: {{ .Values.paymentService.servicename }}
          port: {{ .Values.paymentService.port }}
    - match: Host(`{{ .Values.ingress.domain }}`) && PathPrefix(`/api/deliveries`)
      kind: Rule
      priority: 100
      services:
        - name: {{ .Values.deliveryService.servicename }}
          port: {{ .Values.deliveryService.port }}
    - match: Host(`{{ .Values.ingress.domain }}`) && PathPrefix(`/api/notifications`)
      kind: Rule
      priority: 100
      services:
        - name: {{ .Values.notificationService.servicename }}
          port: {{ .Values.notificationService.port }}
    # --- frontend catch-all (lowest priority) ---
    - match: Host(`{{ .Values.ingress.domain }}`) && PathPrefix(`/`)
      kind: Rule
      priority: 1
      services:
        - name: {{ .Values.frontend.servicename }}
          port: 80
  {{- if .Values.ingress.tls }}
  tls:
    secretName: {{ .Values.ingress.tlsSecretName }}
  {{- end }}
{{- end }}
```

#### After

```yaml
{{- /*
   👉 NEW: hostsMatcher helper. Renders Traefik's host clause from
   .Values.ingress.hosts.

   Traefik v3's Host() function only accepts ONE hostname argument; the
   comma form Host(`a`,`b`) is rejected with
     "Host: unexpected number of parameters; got 2, expected one of [1]"
   and the entire IngressRoute gets silently disabled.

   The correct multi-host form is one Host() per name, OR'd together:
     Host(`a`) || Host(`b`) || Host(`c`)

   We render exactly that, and every caller below wraps the expression
   in parens so the surrounding `&& PathPrefix(...)` precedence works.
*/ -}}
{{- define "cloudkitchen.hostsMatcher" -}}
{{- range $i, $h := .Values.ingress.hosts -}}{{- if $i }} || {{ end }}Host(`{{ $h }}`){{- end -}}
{{- end -}}
{{- if .Values.ingress.enabled }}
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: cloudkitchen
  namespace: {{ .Values.namespace }}
  labels:
    app: cloudkitchen
spec:
  entryPoints:
    - {{ .Values.ingress.entryPoint }}
  routes:
    # --- menu sub-paths under /api/restaurants (higher priority) ---
    - match: ({{ include "cloudkitchen.hostsMatcher" . }}) && PathRegexp(`^/api/restaurants/[^/]+/(menu|categories|items)`)
      kind: Rule
      priority: 200
      services:
        - name: {{ .Values.menuService.servicename }}
          port: {{ .Values.menuService.port }}
    - match: ({{ include "cloudkitchen.hostsMatcher" . }}) && PathPrefix(`/api/menu`)
      kind: Rule
      priority: 200
      services:
        - name: {{ .Values.menuService.servicename }}
          port: {{ .Values.menuService.port }}
    # --- one rule per service ---
    - match: ({{ include "cloudkitchen.hostsMatcher" . }}) && PathPrefix(`/api/auth`)
      kind: Rule
      priority: 100
      services:
        - name: {{ .Values.authService.servicename }}
          port: {{ .Values.authService.port }}
    - match: ({{ include "cloudkitchen.hostsMatcher" . }}) && PathPrefix(`/api/users`)
      kind: Rule
      priority: 100
      services:
        - name: {{ .Values.userService.servicename }}
          port: {{ .Values.userService.port }}
    - match: ({{ include "cloudkitchen.hostsMatcher" . }}) && PathPrefix(`/api/restaurants`)
      kind: Rule
      priority: 100
      services:
        - name: {{ .Values.restaurantService.servicename }}
          port: {{ .Values.restaurantService.port }}
    - match: ({{ include "cloudkitchen.hostsMatcher" . }}) && (PathPrefix(`/api/cart`) || PathPrefix(`/api/orders`))
      kind: Rule
      priority: 100
      services:
        - name: {{ .Values.orderService.servicename }}
          port: {{ .Values.orderService.port }}
    - match: ({{ include "cloudkitchen.hostsMatcher" . }}) && PathPrefix(`/api/payments`)
      kind: Rule
      priority: 100
      services:
        - name: {{ .Values.paymentService.servicename }}
          port: {{ .Values.paymentService.port }}
    - match: ({{ include "cloudkitchen.hostsMatcher" . }}) && PathPrefix(`/api/deliveries`)
      kind: Rule
      priority: 100
      services:
        - name: {{ .Values.deliveryService.servicename }}
          port: {{ .Values.deliveryService.port }}
    - match: ({{ include "cloudkitchen.hostsMatcher" . }}) && PathPrefix(`/api/notifications`)
      kind: Rule
      priority: 100
      services:
        - name: {{ .Values.notificationService.servicename }}
          port: {{ .Values.notificationService.port }}
    # --- frontend catch-all (lowest priority) ---
    - match: ({{ include "cloudkitchen.hostsMatcher" . }}) && PathPrefix(`/`)
      kind: Rule
      priority: 1
      services:
        - name: {{ .Values.frontend.servicename }}
          port: 80
  {{- if .Values.ingress.tls }}
  tls:
    secretName: {{ .Values.ingress.tlsSecretName }}
  {{- end }}
{{- end }}
```

#### Two changes, summarised

| Change | Before | After |
| --- | --- | --- |
| **Helper at top of file** | (none) | New `{{- define "cloudkitchen.hostsMatcher" -}}` block (the first ~16 lines of the file) |
| **Each of the 10 `match:` lines** | `Host(\`{{ .Values.ingress.domain }}\`) && ...` | `({{ include "cloudkitchen.hostsMatcher" . }}) && ...` |

After your edits, **count the occurrences** to be sure you didn't miss any:

```bash
# OLD pattern should be 0:
grep -c 'Host(`{{ .Values.ingress.domain }}`)' helm/cloudkitchen/templates/ingressroute.yaml
# NEW pattern should be 11 (10 routes + 1 helper define line):
grep -c 'cloudkitchen.hostsMatcher' helm/cloudkitchen/templates/ingressroute.yaml
```

#### Render-test before pushing

Don't push until you've confirmed the chart renders correctly:

```bash
helm template cloudkitchen ./helm/cloudkitchen | grep -E "match:" | head -5
```

Expected output (each line starts with the same `(Host(...) || Host(...))`):
```
    - match: (Host(`vijaygiduthuri.in`) || Host(`35.224.38.103`)) && PathRegexp(`^/api/restaurants/...`)
    - match: (Host(`vijaygiduthuri.in`) || Host(`35.224.38.103`)) && PathPrefix(`/api/menu`)
    - match: (Host(`vijaygiduthuri.in`) || Host(`35.224.38.103`)) && PathPrefix(`/api/auth`)
    - match: (Host(`vijaygiduthuri.in`) || Host(`35.224.38.103`)) && PathPrefix(`/api/users`)
    - match: (Host(`vijaygiduthuri.in`) || Host(`35.224.38.103`)) && PathPrefix(`/api/restaurants`)
```

---

## Step 4 — Push through the GitOps loop

```bash
git add helm/cloudkitchen/values.yaml helm/cloudkitchen/templates/ingressroute.yaml
git commit -m "phase 5: support hostname-based access (vijaygiduthuri.in)"
git push origin main
```

**Important:** the workflow `.github/workflows/ci-gcp.yaml` does **not**
trigger on changes under `helm/**` — its `paths:` filter is just the
per-service directories. So this push **does not** rebuild any images
and **does not** land a `cloudkitchen-ci[bot]` commit. ArgoCD picks the
change up directly from your commit, via its 3-minute repo poll.

To skip the 3-minute wait, force a refresh:

```bash
kubectl -n argocd annotate app cloudkitchen \
  argocd.argoproj.io/refresh=hard --overwrite
```

Watch the Application transition:

```bash
kubectl -n argocd get app cloudkitchen -w
# Expect:  Sync Status: OutOfSync -> Synced ; Health: Healthy
# (Ctrl+C when it reaches Synced + Healthy)
```

---

## Step 5 — Verify hostname-based access

```bash
HOSTNAME='vijaygiduthuri.in'
LB_IP=$(kubectl -n traefik get svc traefik -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

echo "=== via hostname ==="
curl -s    -o /dev/null -w "  /                  -> HTTP %{http_code}\n" "http://${HOSTNAME}/"
curl -sI   -o /dev/null -w "  /argocd/           -> HTTP %{http_code}\n" "http://${HOSTNAME}/argocd/"
curl -s    -o /dev/null -w "  /api/restaurants   -> HTTP %{http_code}\n" "http://${HOSTNAME}/api/restaurants"

echo "=== via raw IP (must still work — second host in the list) ==="
curl -s    -o /dev/null -w "  /                  -> HTTP %{http_code}\n" "http://${LB_IP}/"
curl -s    -o /dev/null -w "  /api/restaurants   -> HTTP %{http_code}\n" "http://${LB_IP}/api/restaurants"
```

All should return **HTTP 200**.

> ⚠️ **HEAD vs GET quirk**: some Gin routes don't define an explicit HEAD
> handler. If `curl -I` (HEAD) returns 404 but `curl` (GET) returns 200,
> that's expected and harmless — Gin treats undeclared HEAD as a separate
> method that wasn't registered. Real browser traffic uses GET, so this
> doesn't affect users.

You should now be able to open these in a browser:
- **App:** http://vijaygiduthuri.in/
- **ArgoCD:** http://vijaygiduthuri.in/argocd/  (trailing slash required)

---

## Step 6 — Sanity-check Traefik's parsed routers

If Step 5 fails, this tells you whether Traefik even **accepted** the new
IngressRoute — a syntax error in the matcher silently disables the router
and every request 404s.

```bash
kubectl -n traefik exec deploy/traefik -- wget -qO- http://localhost:8080/api/http/routers \
  | python3 -c "
import json, sys
ck = [r for r in json.load(sys.stdin) if 'cloudkitchen' in r.get('name','').lower()]
enabled = [r for r in ck if r.get('status')=='enabled']
print(f'  {len(enabled)} / {len(ck)} cloudkitchen routers ENABLED')
for r in ck:
  if r.get('status') != 'enabled':
    print(f'  ❌  {r[\"name\"][:60]}  err={r.get(\"error\")}')"
```

Expected: `10 / 10 cloudkitchen routers ENABLED`.

If any are not enabled, the `err=` field tells you exactly which rule
failed parsing — see the Troubleshooting table.

---

## Step 7 — (Optional but recommended) Reserve the LB IP as a static address

By default GKE assigns an **ephemeral** IP to the Traefik LoadBalancer.
If you ever delete and recreate the Traefik Service, the IP changes and
your DNS goes stale until you update GoDaddy.

Promote the current IP to a regional **static** address so it survives
Service recreation:

```bash
PROJECT=$(gcloud config get-value project)
REGION=us-central1
LB_IP=$(kubectl -n traefik get svc traefik -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

# 1. Reserve the current ephemeral IP — this 'upgrades' it to a static.
gcloud compute addresses create traefik-lb-ip \
  --addresses="${LB_IP}" \
  --region="${REGION}" \
  --project="${PROJECT}"

# 2. Pin the Traefik Service to that static IP (helm upgrade — Traefik is
#    still managed by helm, NOT by ArgoCD, since we installed it directly
#    via helm install in Phase 2).
helm upgrade traefik traefik/traefik \
  --namespace traefik --reuse-values \
  --set service.loadBalancerIP="${LB_IP}"

# 3. Confirm the IP didn't change after the upgrade.
kubectl -n traefik get svc traefik -o jsonpath='{.status.loadBalancer.ingress[0].ip}{"\n"}'
```

If you ever **destroy the cluster**, also release the static IP so you
don't get billed for an unused address (~$7/month):

```bash
gcloud compute addresses delete traefik-lb-ip --region=us-central1
```

---

## Troubleshooting

These are the **real failures we hit** on `cloudkitchen-dev-01` while
landing Phase 5, in order:

| Symptom                                                                                                                                                                                                                                                                          | Root cause                                                                                                                                                                       | Fix                                                                                                                                                                                                                            |
| ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `dig vijaygiduthuri.in` returns nothing for 15+ minutes                                                                                                                                                                                                                          | GoDaddy's authoritative NS hasn't pushed the record yet, OR the record was saved against the wrong domain in your GoDaddy account                                                | Open https://dnschecker.org and paste your hostname — distinguishes "still propagating" from "GoDaddy config wrong". Recheck the DNS page; ensure you saved on the right domain.                                              |
| `nslookup` and `dig` disagree (one returns the IP, the other "no answer")                                                                                                                                                                                                       | One of them is querying a different DNS server. `nslookup` defaults to your system resolver; `dig +short @8.8.8.8` is explicitly Google.                                          | Use the same server in both: `nslookup vijaygiduthuri.in 8.8.8.8` AND `dig +short @8.8.8.8 vijaygiduthuri.in`. If they agree on the IP, you're good.                                                                            |
| Hostname resolves correctly, but `curl http://<host>/` returns 404 even though `curl http://<LB_IP>/` worked before                                                                                                                                                              | Chart's IngressRoute has `Host(\`<LB_IP>\`)` hard-coded; the browser/curl sends `Host: <hostname>` and Traefik doesn't match.                                                    | Phase 5 Step 3: parameterize the chart with `ingress.hosts: [...]` and rewrite the template to OR them with `||`.                                                                                                              |
| After Step 3 push, ALL routes (including IP-by-curl) return 404. Traefik logs show `error while adding rule Host: unexpected number of parameters; got 2, expected one of [1]` and `http://traefik:8080/api/http/routers` shows all `cloudkitchen-*` routers `status=disabled`.   | The multi-host syntax `Host(\`a\`,\`b\`)` is invalid in Traefik 3. `Host()` accepts only one argument; multi-host must be written as `Host(\`a\`) || Host(\`b\`)`.                | Rewrite the helper to render `||`-joined `Host()` calls + wrap each rule's matcher in `(...)` so `||` binds tighter than the surrounding `&&`. This is what the **After** template in Step 3b does.                            |
| `curl -I` returns 404 but `curl` (GET) returns 200                                                                                                                                                                                                                                | Gin doesn't auto-register HEAD handlers for routes defined only as `GET`. Traefik forwards the HEAD verbatim, the service has no HEAD route, replies 404.                        | Not a bug, expected. Real traffic is GET. If you ever need HEAD support, register a `HEAD` handler in the Gin router.                                                                                                          |
| ArgoCD shows `Synced` but the cluster still serves old behaviour                                                                                                                                                                                                                  | The Application reported "Synced" against an old git revision; the new commit hasn't been polled yet.                                                                            | `kubectl -n argocd annotate app cloudkitchen argocd.argoproj.io/refresh=hard --overwrite` — forces an immediate repo refresh + sync. Or just click "Refresh" in the ArgoCD UI.                                                  |
| Chart change works for some routes but not others                                                                                                                                                                                                                                 | One stale rule with the broken syntax was left in the template (sed/edit missed it)                                                                                              | `grep -c 'Host(\`{{ .Values.ingress.domain }}\`)' helm/cloudkitchen/templates/ingressroute.yaml` — should return 0. If not, replace the leftovers.                                                                              |
| You deleted and recreated the Traefik Service and now DNS resolves to a dead IP                                                                                                                                                                                                  | GKE assigned a new ephemeral IP to the new Service; the old IP is gone.                                                                                                          | Update the GoDaddy A record to the new IP, **or** (better) do Step 7 — reserve a static IP so this never happens again.                                                                                                        |

---

## What was committed

| Commit    | Files changed                                                  | What it does                                                                                                                  |
| --------- | -------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| `5402106` | `helm/cloudkitchen/values.yaml`, `templates/ingressroute.yaml` | First attempt: introduced `ingress.hosts` (list) and the `cloudkitchen.hostsMatcher` helper. Used the comma syntax — broken.   |
| `f18fb4f` | `helm/cloudkitchen/templates/ingressroute.yaml`                | Fixed: replaced `Host(\`a\`,\`b\`)` with `Host(\`a\`) || Host(\`b\`)` and wrapped each call site in parens for precedence.    |

Both commits stay in history on purpose — the bad → good progression is
the same instructive moment a reader following this doc lives through.

---

➡️ **Next:** Phase 6 — Monitoring & Logging (Prometheus + Grafana + Loki).
