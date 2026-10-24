# Monitoring Stack (Prometheus + Grafana)

## Overview

The homelab runs the `prometheus-community/kube-prometheus-stack` Helm chart in the
`monitoring` namespace. It bundles Prometheus, Alertmanager, Node Exporter
(DaemonSet), kube-state-metrics, and Pushgateway, plus Grafana with preconfigured
dashboards and a Prometheus data source.

Prometheus scrapes metrics from every node, the Kubernetes control plane, and the
cluster objects themselves. Grafana consumes those metrics behind a data source, so
the two services are already wired together — no manual configuration is needed to
open a dashboard.

## Stack Layout

| Namespace | Component | Kind | Purpose |
|-----------|-----------|------|---------|
| `monitoring` | `prometheus-stack-kube-prometheus-operator` | Deployment | Watches Prometheus CRDs |
| `monitoring` | `prometheus-prometheus-stack-kube-prom-prometheus` (StatefulSet) | StatefulSet | Prometheus server, 1000s persistence |
| `monitoring` | `alertmanager-prometheus-stack-kube-prom-alertmanager` (StatefulSet) | StatefulSet | Alert routing and notifications |
| `monitoring` | `prometheus-stack-kube-prometheus-node-exporter` | DaemonSet | Host metrics on every node |
| `monitoring` | `prometheus-stack-kube-state-metrics` | Deployment | Kubernetes object metrics |
| `monitoring` | `prometheus-stack-grafana` | Deployment | Dashboards + Prometheus datasource |
| `default` | `dynamic-404`, `landing-page`, … | Deployment | Application pods (no Prometheus targets) |

## How It Was Installed

The stack replaced an earlier, manually-applied Prometheus deployment (pre-stack
backups live in `manifests/_bak-monitoring-pre-stack/`). It was installed from the
`prometheus-community` chart repo after adding Helm to the workstation.

```bash
# 1. Install Helm 4.3.1 to a user bin on Windows (get.helm.sh only serves binaries,
#    not GitHub release assets — the real zip is https://get.helm.sh/helm-v4.3.1-windows-amd64.zip)
unzip helm-v4.3.1-windows-amd64.zip -d "$HOME/.local/bin"
# $HOME/.local/bin/windows-amd64 is added to PATH in ~/.bashrc

# 2. Add the chart repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# 3. Install the stack (Grafana Ingress routed by Traefik)
helm upgrade --install prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set grafana.ingress.enabled=true \
  --set grafana.ingress.hosts[0]=grafana.jdools.com \
  --wait --timeout 10m

# 4. Force the bundled Prometheus data source into Grafana at boot
helm upgrade --install prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set prometheus.prometheusSpec.enabled=true \
  --set grafana.ingress.enabled=true \
  --set grafana.ingress.hosts[0]=grafana.jdools.com \
  --set prometheus.nodeExporter.enabled=true \
  --set grafana.forceDeployDatasources=true \
  --set grafana.defaultDashboardsEnabled=true \
  --wait --timeout 10m
```

### Gotcha: the `routePrefix` MSYS2 path trap

The first install passed `--set prometheus.prometheusSpec.routePrefix=/` to force
Prometheus to sit behind a single Traefik path. On a MinGW/MSYS2 workstation that
bare `/` argument gets rewritten to the host's Git path — the pod spec picked up
`/C:/Program Files/Git/` and Prometheus panicked on boot:

```
panic: wildcards must be named with a non-empty name in path '/C:/Program Files/Git/'
```

The fix was to drop `routePrefix` entirely. With no prefix, Prometheus serves at `/`
inside the cluster and Traefik handles the routing. Do not pass a `routePrefix`
unless it is written as a Windows-style string (e.g. `/monitoring/`) to avoid the
MSYS2 rewrite.

## Viewing the Stack

Prometheus, Alertmanager, and Grafana all expose services inside the cluster.
External access is via Ingress → Traefik → pod, on the MetalLB IP (`10.10.30.100`).

For devices on the tailnet, Grafana and Prometheus are also exposed directly over
the tailnet via the Tailscale Kubernetes operator — see
[tailscale-operator.md](./tailscale-operator.md). That route uses MagicDNS names
(`monitoring-prometheus-stack-grafana.taile4e2f7.ts.net`) and requires no ingress
or TLS; it only works from devices already on the tailnet.

### Grafana (dashboards)

- **External**: `http://grafana.jdools.com` (Ingress `prometheus-stack-grafana`)
- **Local port-forward**:

```bash
kubectl -n monitoring port-forward svc/prometheus-stack-grafana 3000:80
# open http://localhost:3000
```

Log in with `admin`. The password is stored in a Kubernetes Secret:

```bash
kubectl -n monitoring get secret prometheus-stack-grafana \
  -o jsonpath='{.data.admin-password}' | base64 --decode ; echo
```

Grafana comes with the standard Kubernetes and node dashboards pre-imported and the
Prometheus data source already defined in the
`prometheus-stack-kube-prom-grafana-datasource` ConfigMap.

### Prometheus (queries, targets, rules)

- **Tailnet**: `monitoring-prometheus-stack-kube-prom-prometheus.taile4e2f7.ts.net` (see [tailscale-operator.md](./tailscale-operator.md))
- **Local port-forward**:

```bash
kubectl -n monitoring port-forward svc/prometheus-operated 9090:9090
# open http://localhost:9090
```

From here you can browse metrics, query `up` or any metric, and inspect targets and
alerting rules.

### Alertmanager

- **Local port-forward**:

```bash
kubectl -n monitoring port-forward svc/prometheus-stack-kube-prom-alertmanager 9093:9093
# open http://localhost:9093
```

## Helm Release Management

The release is named `prometheus-stack` in `monitoring`.

```bash
# Current release status
helm list -n monitoring

# Inspect the installed values
helm get values prometheus-stack -n monitoring

# Roll back if a change breaks the stack
helm rollback prometheus-stack -n monitoring 1
```

Helm lives at `~/.local/bin/windows-amd64/helm.exe` on the workstation (added to
`PATH` in `~/.bashrc`).

## Current State

- `prometheus-stack` release: **deployed** (chart `kube-prometheus-stack-92.11.0`,
  app `v0.94.1`).
- All Prometheus, Alertmanager, and Grafana pods: `Running`, endpoints ready.
- Node Exporter running on all three nodes; kube-state-metrics and the operator up.
- Grafana exposed at `grafana.jdools.com` through Traefik on the MetalLB IP.
- Prometheus data source and default dashboards present in the Grafana ConfigMaps.
