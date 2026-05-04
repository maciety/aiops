# Home Network Map

This is the living map of Maciek's home/lab network as Tars learns it.

## Known domains and entrypoints

- Public/internal service domain: `moleda.io`
- OpenClaw UI: `https://openclaw.moleda.io`
- Prometheus: `https://prometheus.moleda.io`
- Loki API: `https://loki-api.moleda.io`
- ArgoCD: `argocd.moleda.io:443`

## Known hosts

### `ollama.moleda.io` / `ollama`

Known access paths:

- SSH: `ssh ollama` or `ssh ollama.moleda.io`

Known services:

- Ollama via systemd on port `11434`
- OpenClaw gateway as a user systemd service on port `18789`

Important paths:

- OpenClaw config: `/root/.openclaw/openclaw.json`
- OpenClaw gateway service override/credentials: `/root/.config/systemd/user/openclaw-gateway.service.d/anthropic.conf`
- OpenClaw paired devices: `/root/.openclaw/devices/paired.json`
- OpenClaw pending pairing requests: `/root/.openclaw/devices/pending.json`

### Traefik ingress

- OpenClaw is proxied through Traefik at `192.168.50.12`.
- Traefik must be trusted by OpenClaw via `gateway.trustedProxies` when proxy headers are present.

## Kubernetes / k3s

- Home cluster runs k3s and is managed by ArgoCD GitOps.
- `kubectl` default context is expected to point at this cluster for read-only investigation.
- Apps generally live under the `moleda.io` domain.
- Ingress stack: Traefik + cert-manager.
- Full app layout and conventions are documented on Maciek's machine at `/Users/maciek/src/home-k8s/CLAUDE.md`.
- Gotcha: changes to `applications/*.yaml` must be applied manually with `kubectl apply`; ArgoCD does not self-manage its own Application CRDs.

## Observability

### Prometheus

- URL: `https://prometheus.moleda.io`
- Use HTTP API endpoints such as `/api/v1/query` and `/api/v1/query_range` for metrics investigation.

### Loki

- URL: `https://loki-api.moleda.io`
- Query API example:

```sh
curl 'https://loki-api.moleda.io/loki/api/v1/query_range?query=...'
```

## Troubleshooting workflow

1. Define the failing user-visible endpoint or service.
2. Check DNS/HTTP reachability from the current environment.
3. Check ingress/proxy path: domain → Traefik → Kubernetes service/pod, or domain → host service.
4. Check Prometheus for health, restarts, saturation, and recent changes.
5. Check Loki or service logs for the exact failing component.
6. Only then propose or make a fix, and document the resulting runbook.

## Needs confirmation / to discover

- LAN subnet(s), VLANs, and router/firewall model.
- k3s node names, IPs, roles, storage classes, and backup strategy.
- Home Assistant URL, host, install type, and integration boundaries.
- NAS/storage layout, if any.
- Alerting path and notification channels.
