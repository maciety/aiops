# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Purpose

This is an ops workspace for investigating issues with services running on the user's local network. Work here is primarily interactive investigation using the Bash tool — no build system or test suite.

## Infrastructure

### ollama.moleda.io
Runs Ollama (systemd, port 11434) and OpenClaw gateway (user systemd service, port 18789). SSH access via `ssh ollama` or `ssh ollama.moleda.io`. Config at `/root/.openclaw/openclaw.json`; credentials override at `/root/.config/systemd/user/openclaw-gateway.service.d/anthropic.conf`.

### Prometheus
Metrics available at https://prometheus.moleda.io — use the WebFetch tool to query the HTTP API (e.g. `/api/v1/query`, `/api/v1/query_range`) for cluster and node metrics during investigations.

### Loki
Logs are stored in Loki, accessible at https://loki-api.moleda.io — use the HTTP API directly:
```
curl 'https://loki-api.moleda.io/loki/api/v1/query_range?query=...'
```

### k3s Kubernetes cluster
Home cluster managed via ArgoCD GitOps. Full details — app layout, access, conventions, gotchas — are in `/Users/maciek/src/home-k8s/CLAUDE.md`. Key points:
- `kubectl` default context points at this cluster; use it freely for read-only investigation.
- ArgoCD login: `argocd login argocd.moleda.io:443 --username $ARGOCD_USERNAME --password $ARGOCD_PASSWORD`
- Apps live under `moleda.io` domain; ingress via Traefik + cert-manager.
- Changes to `applications/*.yaml` must be `kubectl apply`'d manually — ArgoCD does not self-manage its own Application CRDs.

## Notes and Runbooks

Save all troubleshooting notes, runbooks, and hard-won fixes as Markdown files in this repo (e.g. `openclaw-gateway.md`). Do not save them only to an assistant memory directory — they need to live here so they can move between machines.

Start with:
- `TARS.md` — Tars/OpenClaw assistant operating notes.
- `home-network-map.md` — living topology and service map for Maciek's home/lab network.
- `openclaw-gateway.md` — OpenClaw gateway troubleshooting runbook.
- `k3s.md` — Kubernetes/k3s health checks, Prometheus alerts, and Longhorn cleanup notes.

## Working Style

- The primary tool is Bash: use it to inspect services, check logs, probe network endpoints, and diagnose issues.
- Prefer targeted, non-destructive commands. Read before writing, observe before changing.
- When investigating an issue, build up a picture of the system state before proposing fixes.
- Confirm with the user before making any change that affects a running service or modifies system configuration.
