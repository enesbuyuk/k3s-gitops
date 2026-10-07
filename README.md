# k3s-gitops

Declarative manifests for a single-node K3s cluster on Debian, using
**Traefik 3** (managed by K3s) and the **Kubernetes Gateway API**
(`Gateway` + `HTTPRoute`). No `Ingress` resources are used.

Deployment is GitOps with **Argo CD** (app-of-apps pattern): Argo CD watches
this repository and syncs the cluster to it. Plain `kubectl apply` still works
for every directory and is used once to bootstrap Argo CD
(see [Argo CD](#14-argo-cd-gitops)).

---

## 1. Architecture

```text
Internet
   |
   | 80 / 443
   v
K3s ServiceLB (svclb-traefik)
   |
   | 80  -> 8000 (entryPoint "web")
   | 443 -> 8443 (entryPoint "websecure")
   v
Traefik (kube-system, managed by K3s)
   |
   | GatewayClass "traefik"  (controller: traefik.io/gateway-controller)
   v
Gateway gateway-system/public-gateway
   |
   +-- HTTPRoute -> nginx.example.com -> default/nginx      (example in this repo)
   +-- HTTPRoute -> app.example.com   -> frontend           (future)
   +-- HTTPRoute -> api.example.com   -> backend            (future)
   +-- HTTPRoute -> k8s.example.com   -> Headlamp           (future)
```

- **Traefik** is the data plane. It is installed and upgraded by K3s.
- **public-gateway** is the single shared entry point, owned by the platform
  (infrastructure) side.
- **HTTPRoutes** are owned by each application and live in the application's
  own namespace.

---

## 2. Directory structure

```text
k3s-gitops/
├── bootstrap/                   # Argo CD Application definitions
│   ├── root.yaml                # app-of-apps root (applied once by hand)
│   └── applications/
│       ├── argocd.yaml          # Argo CD manages itself
│       ├── gateway.yaml         # -> infrastructure/gateway
│       └── nginx.yaml           # -> apps/nginx
│
├── infrastructure/              # Cluster-wide, platform-owned components
│   ├── argocd/
│   │   ├── kustomization.yaml   # upstream install.yaml, pinned version
│   │   ├── namespace.yaml
│   │   ├── argocd-cmd-params-cm.yaml  # server.insecure (TLS at Gateway)
│   │   └── httproute.yaml       # argocd.example.com (enable after HTTPS)
│   ├── gateway/
│   │   ├── kustomization.yaml   # namespace + gateway (kubectl apply -k)
│   │   ├── namespace.yaml       # gateway-system namespace
│   │   ├── gateway.yaml         # shared public-gateway (HTTP now, HTTPS stub)
│   │   └── traefik-config.yaml  # K3s HelmChartConfig: enables Gateway provider
│   ├── headlamp/                # (placeholder) Kubernetes web UI
│   ├── cert-manager/            # (placeholder) TLS certificates
│   └── monitoring/              # (placeholder) Prometheus / Grafana
│
├── apps/                        # Application workloads
│   └── nginx/
│       ├── kustomization.yaml
│       ├── deployment.yaml
│       ├── service.yaml         # ClusterIP
│       └── httproute.yaml       # attaches to gateway-system/public-gateway
│
├── .gitignore
└── README.md
```

Rule of thumb: **infrastructure/** changes rarely and affects the whole
cluster; **apps/** changes often and affects one application.

---

## 3. Prerequisites

Already present in the cluster (this repo does **not** install them):

| Component | Check |
|---|---|
| K3s with bundled Traefik 3.x | `kubectl get pods -n kube-system -l app.kubernetes.io/name=traefik` |
| Gateway API CRDs | `kubectl get crd gateways.gateway.networking.k8s.io httproutes.gateway.networking.k8s.io` |
| GatewayClass `traefik`, `ACCEPTED=True` | `kubectl get gatewayclass` |
| Traefik arg `--providers.kubernetesgateway` | `kubectl get deploy traefik -n kube-system -o yaml \| grep kubernetesgateway` |
| ServiceLB exposing 80/443 | `kubectl get svc traefik -n kube-system` |

Locally: `kubectl` (v1.14+ for `-k`) with a kubeconfig pointing at the cluster.
Firewall on the VPS must allow inbound TCP 80 (and 443 for later HTTPS).

> Do not deploy another Traefik, do not install Gateway API CRDs, and do not
> create a GatewayClass — K3s/Traefik already provide them.

---

## 4. How the shared Gateway works

`infrastructure/gateway/gateway.yaml` defines one `Gateway` named
`public-gateway` in namespace `gateway-system`, using `gatewayClassName: traefik`.

Its `http` listener:

```yaml
- name: http
  protocol: HTTP
  port: 8000            # Traefik "web" entryPoint port, NOT 80
  allowedRoutes:
    kinds:
      - kind: HTTPRoute
    namespaces:
      from: All         # HTTPRoutes from any namespace may attach
```

Key points:

- **Port 8000, not 80.** Traefik maps Gateway listeners onto its own
  entryPoints, so the listener port must equal the entryPoint's internal port
  (`web` = 8000, `websecure` = 8443). ServiceLB translates external 80 → 8000.
- **No hostname on the listener.** The listener accepts any host; each
  HTTPRoute declares the hostnames it serves.
- **`allowedRoutes.namespaces.from: All`** lets application namespaces attach
  routes. To lock this down, switch to `from: Selector` with a label
  (instructions are in the comments in `gateway.yaml`).
- **HTTPS-ready.** A commented `https` listener on port 8443 is prepared. Once
  cert-manager issues a certificate Secret into `gateway-system`, uncomment it
  and add `sectionName: https` parentRefs to routes.

---

## 5. Gateway vs HTTPRoute

| | Gateway | HTTPRoute |
|---|---|---|
| Purpose | *Where* traffic enters: ports, protocols, TLS | *How* traffic is routed: hostnames, paths, backends |
| Owner | Platform / cluster admin | Application team |
| Count | One shared (`public-gateway`) | One (or more) per application |
| Namespace | `gateway-system` | Application namespace |
| Analogy | The building's front door | Signs telling visitors which room to go to |

The `GatewayClass` (`traefik`) sits above both and says *which controller*
implements Gateways — here, Traefik.

---

## 6. Attaching an application's HTTPRoute to the shared Gateway

Each HTTPRoute references the Gateway through `parentRefs`. Because the route
is in a different namespace, `namespace` is required:

```yaml
spec:
  parentRefs:
    - name: public-gateway
      namespace: gateway-system   # cross-namespace reference
      sectionName: http           # attach to the "http" listener only
  hostnames:
    - app.example.com
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: my-service        # Service in the SAME namespace as the route
          port: 80
```

Attachment succeeds when the Gateway listener's `allowedRoutes` permits the
route's namespace and kind. No `ReferenceGrant` is needed for Route → Gateway.
(A `ReferenceGrant` would only be required if a `backendRef` pointed to a
Service in a *different* namespace than the route.)

To add a new app: copy `apps/nginx/`, change names, image, port and hostname.

---

## 7. Deploy the infrastructure

**Step 1 — Traefik Gateway provider (already active on this cluster).**
Before applying, check for an existing config, because `valuesContent` is
replaced, not merged:

```bash
kubectl get helmchartconfig traefik -n kube-system -o yaml
```

If it contains other values, merge them into `infrastructure/gateway/traefik-config.yaml`, then:

```bash
kubectl apply -f infrastructure/gateway/traefik-config.yaml

# K3s re-runs the Traefik Helm install; wait for the rollout
kubectl rollout status deploy/traefik -n kube-system
```

> If the same HelmChartConfig also lives on the node in
> `/var/lib/rancher/k3s/server/manifests/`, K3s will re-apply that copy.
> Keep exactly one source of truth.

**Step 2 — namespace + shared Gateway:**

```bash
kubectl apply -k infrastructure/gateway
```

(Equivalent without kustomize: `kubectl apply -f infrastructure/gateway/namespace.yaml -f infrastructure/gateway/gateway.yaml`.)

---

## 8. Deploy nginx

**First, replace the placeholder hostname.** Edit `apps/nginx/httproute.yaml`:

```yaml
  hostnames:
    - nginx.example.com   # <-- REPLACE with your real domain, e.g. nginx.mydomain.com
```

Then:

```bash
kubectl apply -k apps/nginx
kubectl rollout status deploy/nginx -n default
```

---

## 9. Check Gateway status

```bash
kubectl get gatewayclass
kubectl get gateway -A
kubectl describe gateway -n gateway-system public-gateway
```

Healthy output: `PROGRAMMED=True`, and in `describe` the conditions
`Accepted=True` and `Programmed=True`. Under `Listeners`, `http` should show
`Attached Routes: 1` (or more) and `ResolvedRefs=True`.

---

## 10. Check HTTPRoute status

```bash
kubectl get httproute -A
kubectl describe httproute -n default nginx
```

In `Status.Parents`, for parent `gateway-system/public-gateway`, expect:

- `Accepted=True` — the Gateway accepted the route
- `ResolvedRefs=True` — the backend Service `nginx:80` exists

Common failure reasons: `NotAllowedByListeners` (allowedRoutes blocks the
namespace), `NoMatchingParent` (wrong Gateway name/namespace/sectionName),
`BackendNotFound` (wrong Service name/port).

---

## 11. Test routing with curl

Before DNS exists, fake the Host header against the server IP:

```bash
curl -i -H "Host: nginx.example.com" http://<SERVER_PUBLIC_IP>/
```

Or resolve the name explicitly (closest to real traffic):

```bash
curl -i --resolve nginx.example.com:80:<SERVER_PUBLIC_IP> http://nginx.example.com/
```

From the node itself:

```bash
curl -i -H "Host: nginx.example.com" http://127.0.0.1/
```

After DNS is set:

```bash
curl -i http://nginx.example.com/
```

Expected: `HTTP/1.1 200 OK` with the "Welcome to nginx!" page.
A `404 page not found` (Traefik's default) means the request reached Traefik
but no route matched the Host — check the hostname and HTTPRoute status.

---

## 12. DNS requirements

Create DNS records pointing your hostnames at the VPS public IP:

| Type | Name | Value |
|---|---|---|
| A | `nginx.example.com` | `<SERVER_PUBLIC_IP>` |
| A | `app.example.com` | `<SERVER_PUBLIC_IP>` |
| A | `api.example.com` | `<SERVER_PUBLIC_IP>` |
| A | `k8s.example.com` | `<SERVER_PUBLIC_IP>` |

Alternatively one wildcard record: `A *.example.com → <SERVER_PUBLIC_IP>`.
Add `AAAA` records if the VPS has IPv6. Verify with:

```bash
dig +short nginx.example.com
```

For later HTTPS with Let's Encrypt: HTTP-01 needs port 80 reachable; a
wildcard certificate requires DNS-01 via your DNS provider's API (store the
API token as a Secret created out-of-band, never in Git).

---

## 13. Troubleshooting

```bash
# Overview
kubectl get pods -A
kubectl get svc -A
kubectl get gatewayclass
kubectl get gateway -A
kubectl get httproute -A

# Details and conditions
kubectl describe gateway -n gateway-system public-gateway
kubectl describe httproute -n default nginx
kubectl get gateway public-gateway -n gateway-system -o jsonpath='{.status}' | jq
kubectl get httproute nginx -n default -o jsonpath='{.status.parents}' | jq

# Backend health
kubectl get deploy,pods,svc,endpointslices -n default -l app.kubernetes.io/name=nginx
kubectl get endpointslices -n default -l kubernetes.io/service-name=nginx
kubectl logs -n default deploy/nginx

# Bypass the Gateway: talk to the Service directly
kubectl port-forward -n default svc/nginx 8080:80
curl -i http://127.0.0.1:8080/

# Traefik
kubectl get pods -n kube-system -l app.kubernetes.io/name=traefik
kubectl logs -n kube-system deploy/traefik --tail=100
kubectl get deploy traefik -n kube-system -o jsonpath='{.spec.template.spec.containers[0].args}' | tr ',' '\n'
kubectl get svc traefik -n kube-system
kubectl get helmchartconfig traefik -n kube-system -o yaml
kubectl get pods -n kube-system | grep svclb

# Events (newest last)
kubectl get events -A --sort-by=.lastTimestamp | tail -30
```

| Symptom | Likely cause |
|---|---|
| Gateway `PROGRAMMED=False` / `Unknown` | Listener port not matching a Traefik entryPoint (use 8000/8443), or Gateway provider disabled |
| HTTPRoute `Accepted=False` | Wrong `parentRefs` (name / namespace / sectionName) or blocked by `allowedRoutes` |
| HTTPRoute `ResolvedRefs=False` | Service name or port wrong, or Service in another namespace |
| curl → `404 page not found` | Host header does not match `hostnames` |
| curl → connection refused / timeout | Firewall, ServiceLB pod not running, or DNS pointing elsewhere |
| curl → `503` / `Service Unavailable` | Pods not Ready (check readiness probe / endpoints) |

> Note: the Traefik Helm chart can create its own default Gateway
> (`kube-system/traefik-gateway`) when the Gateway provider is enabled. It does
> not conflict with `public-gateway`. To disable it, add `gateway: {enabled: false}`
> to `traefik-config.yaml` — check first that nothing attaches routes to it.

---

## 14. Argo CD (GitOps)

### How it works

```text
Git repo (this repository)
   |
   v
Argo CD (namespace argocd)
   |
   +-- Application "root"  -> bootstrap/applications/
          |
          +-- Application "argocd"  -> infrastructure/argocd   (wave -10, no prune)
          +-- Application "gateway" -> infrastructure/gateway  (wave 0)
          +-- Application "nginx"   -> apps/nginx              (wave 10)
```

- **App of apps.** You apply only `bootstrap/root.yaml` by hand. It creates
  one `Application` per file in `bootstrap/applications/`.
- **Automated sync** with `prune` + `selfHeal`: changes merged into `main`
  are applied automatically; manual `kubectl edit` drift is reverted.
- **Argo CD manages itself** (`argocd` Application). `prune` is off there so a
  bad commit cannot delete Argo CD.
- **Not managed by Argo CD:** `infrastructure/gateway/traefik-config.yaml`.
  It modifies the K3s-managed Traefik and stays a deliberate manual step.

### Step 0 — Push this repository

All Applications under `bootstrap/` point to
`https://github.com/enesbuyuk/k3s-gitops.git` (branch `main`). Argo CD syncs
from GitHub, not from your local copy, so push before bootstrapping:

```bash
git remote add origin https://github.com/enesbuyuk/k3s-gitops.git
git push -u origin main
```

If you fork or rename the repo, update `repoURL` in every file under `bootstrap/`.

### Step 1 — Install Argo CD (once)

```bash
kubectl apply -k infrastructure/argocd --server-side --force-conflicts
kubectl wait -n argocd --for=condition=Available deploy --all --timeout=300s
```

`--server-side` is required: some Argo CD CRDs are too large for client-side apply.

### Step 2 — Give Argo CD access to the repo (private repos only)

Public repos need nothing. For a private repo, create the credential
**out-of-band** — never commit it:

```bash
kubectl create secret generic repo-k3s-gitops -n argocd \
  --from-literal=type=git \
  --from-literal=url=https://github.com/enesbuyuk/k3s-gitops.git \
  --from-literal=username=enesbuyuk \
  --from-literal=password=<github-fine-grained-token-read-only>
kubectl label secret repo-k3s-gitops -n argocd argocd.argoproj.io/secret-type=repository
```

(SSH alternative: use `url=git@github.com:enesbuyuk/k3s-gitops.git` and
`--from-file=sshPrivateKey=<path-to-deploy-key>`.)

### Step 3 — Bootstrap the root app

```bash
kubectl apply -f bootstrap/root.yaml
kubectl get applications -n argocd
```

All Applications should reach `Synced` / `Healthy`. If you already deployed
the gateway and nginx with `kubectl apply`, Argo CD simply adopts them.

### Access the UI

Until HTTPS is enabled on the Gateway, use port-forward (do not send the Argo
CD login over plain HTTP):

```bash
# Initial admin password (generated in-cluster, never in Git)
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d; echo

kubectl port-forward -n argocd svc/argocd-server 8080:80
# open http://localhost:8080  (user: admin)
```

After first login, change the password and delete the initial secret:
`kubectl -n argocd delete secret argocd-initial-admin-secret`.

**Later, with HTTPS:** enable the `https` listener on `public-gateway`, set the
real hostname in `infrastructure/argocd/httproute.yaml`, uncomment
`- httproute.yaml` in `infrastructure/argocd/kustomization.yaml`, and push.

### Day-to-day workflow

| Task | What to do |
|---|---|
| Change an app | Edit files under `apps/<name>/`, commit, push |
| Add an app | Create `apps/<name>/` + `bootstrap/applications/<name>.yaml`, push |
| Remove an app | Delete its `bootstrap/applications/<name>.yaml`, push (finalizer prunes resources) |
| Upgrade Argo CD | Bump version in `infrastructure/argocd/kustomization.yaml`, push |

### Argo CD troubleshooting

```bash
kubectl get applications -n argocd
kubectl describe application -n argocd nginx
kubectl get pods -n argocd
kubectl logs -n argocd deploy/argocd-repo-server --tail=100
kubectl logs -n argocd statefulset/argocd-application-controller --tail=100

# Force a refresh from Git
kubectl annotate application -n argocd nginx argocd.argoproj.io/refresh=hard --overwrite
```

---

## Security

- Never commit Secrets, tokens, private keys, certificates or kubeconfigs.
  `.gitignore` blocks common patterns, but review every commit.
- Create Secrets out-of-band (`kubectl create secret ...`) or adopt
  Sealed Secrets / SOPS before moving to GitOps.
- Do not expose admin UIs (Headlamp, Grafana) without HTTPS and authentication.
