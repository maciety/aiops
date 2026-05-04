# k3s Cluster Runbook

Maciek's home Kubernetes cluster is k3s, managed by ArgoCD GitOps.

## Known nodes

Current Kubernetes nodes observed on 2026-05-04:

| Node | Role | Internal IP | Notes |
| --- | --- | --- | --- |
| `node2` | control-plane, etcd, master | `192.168.50.23` | Raspberry Pi / Ubuntu 22.04 |
| `node3` | control-plane, etcd, master | `192.168.50.133` | Raspberry Pi / Ubuntu 22.04 |
| `rk1` | control-plane, etcd, master | `192.168.50.127` | Rockchip / Ubuntu 24.04 |

Historical/stale node discovered and removed from Longhorn: `jetson1`.

## Quick cluster health check

```sh
kubectl config current-context
kubectl cluster-info
kubectl get nodes -o wide
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded -o wide
kubectl get events -A --field-selector type=Warning --sort-by=.lastTimestamp | tail -80
kubectl top nodes
kubectl top pods -A --sort-by=cpu | head -25
```

Useful extra checks:

```sh
kubectl get deploy -A -o jsonpath='{range .items[?(@.status.unavailableReplicas)]}{.metadata.namespace}/{.metadata.name} unavailable={.status.unavailableReplicas} ready={.status.readyReplicas} desired={.spec.replicas}{"\n"}{end}'
kubectl get ds -A
kubectl get pvc -A
kubectl get applications.argoproj.io -A
kubectl get certificates -A
```

## Prometheus active alerts

Do not stop after verifying that the monitoring pods are healthy. Check the actual active alert state.

```sh
pod=$(kubectl get pod -n monitoring -l app.kubernetes.io/name=prometheus -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n monitoring "$pod" -c prometheus -- \
  sh -c "wget -qO- 'http://127.0.0.1:9090/api/v1/alerts'" > /tmp/prometheus-alerts.json

python3 - /tmp/prometheus-alerts.json <<'PY'
import sys, json
with open(sys.argv[1]) as f:
    data = json.load(f)
alerts = [a for a in data.get('data', {}).get('alerts', []) if a.get('state') in ('firing', 'pending')]
print(f'total_active={len(alerts)}')
for a in alerts:
    labels = a.get('labels', {})
    print(f"{a.get('state','').upper()} severity={labels.get('severity','')} alert={labels.get('alertname','')} ns={labels.get('namespace','')} pod={labels.get('pod','')} pdb={labels.get('poddisruptionbudget','')}")
PY
```

`Watchdog` with severity `none` is expected to be always firing; it verifies the alerting pipeline.

## Longhorn health

```sh
kubectl get volumes.longhorn.io -n longhorn-system
kubectl get nodes.longhorn.io -n longhorn-system
kubectl get pdb -n longhorn-system -o wide
kubectl get pods -n longhorn-system -l longhorn.io/component=instance-manager -o wide --show-labels
```

## Fix: stale Longhorn node causing `KubePdbNotEnoughHealthyPods`

Symptom observed on 2026-05-04:

- Prometheus firing alert: `KubePdbNotEnoughHealthyPods`
- Namespace: `longhorn-system`
- PDB: `instance-manager-5350dec33e0ff695847be50e2998949d`
- PDB selector included `longhorn.io/node=jetson1`
- `jetson1` was not present in `kubectl get nodes`, but still existed as `nodes.longhorn.io/jetson1`
- Longhorn node had 0 replicas and 0 engines, but `spec.allowScheduling: true`, which blocked deletion

This is runtime cleanup, not a desired-state GitOps change. Do not make a repo PR for it unless there is persistent declarative config to change.

Cleanup sequence:

```sh
kubectl get nodes.longhorn.io -n longhorn-system jetson1 -o yaml

# Longhorn may reject deletion while allowScheduling=true, even for a stale KubernetesNodeGone node.
kubectl patch nodes.longhorn.io -n longhorn-system jetson1 \
  --type=merge \
  -p '{"spec":{"allowScheduling":false}}'

kubectl delete nodes.longhorn.io -n longhorn-system jetson1 --ignore-not-found=true
kubectl delete poddisruptionbudget -n longhorn-system instance-manager-5350dec33e0ff695847be50e2998949d --ignore-not-found=true
```

Verify:

```sh
kubectl get nodes.longhorn.io -n longhorn-system
kubectl get pdb -n longhorn-system instance-manager-5350dec33e0ff695847be50e2998949d
# Then re-check Prometheus active alerts.
```

Expected result after cleanup: only the `Watchdog` alert remains firing.
