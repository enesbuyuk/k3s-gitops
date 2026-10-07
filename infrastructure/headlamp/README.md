# Headlamp

Placeholder. Kubernetes web UI, to be exposed at `k8s.example.com` through an
HTTPRoute attached to `gateway-system/public-gateway`.

Planned contents:

- `namespace.yaml`
- Headlamp Deployment/Service (or Helm values)
- `httproute.yaml` → `k8s.example.com`

Do not expose Headlamp publicly without HTTPS and authentication.
Never commit service account tokens to Git.
