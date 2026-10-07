# cert-manager

Placeholder. Will issue TLS certificates (e.g. Let's Encrypt) for the
`https` listener of `gateway-system/public-gateway`.

Planned contents:

- cert-manager installation (with Gateway API support enabled)
- `ClusterIssuer` (ACME / Let's Encrypt)
- `Certificate` whose Secret lives in `gateway-system`

Certificates and private keys are generated in-cluster. Never commit them to Git.
