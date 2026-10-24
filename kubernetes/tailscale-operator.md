# Tailscale Kubernetes Operator

## Overview

The Tailscale Kubernetes Operator is deployed in the `tailscale-system` namespace.
It registers a single operator device on the tailnet and, from there, exposes
designated in-cluster Services as first-class Tailnet hosts via MagicDNS. No
Ingress resources, TLS certs, or external IP routing are involved — traffic flows
directly over the tailnet between devices that are already on it.

This complements the Traefik Ingress in [ingress.md](./ingress.md). Traefik serves
the public site over the Internet via MetalLB; the Tailscale operator serves
selected internal services (Grafana, Prometheus) to devices on the tailnet only.
The two routes are independent and can coexist.

## Architecture

```
                          Tailnet (100.100.x.x / MagicDNS *.ts.net)
                              |                         |
                              |                         |
              ┌───────────────▼───────────────┐  ┌──────▼───────────────────┐
              │  operator device (k8s-control) │  │  client device (laptop)  │
              │  tailscale-operator pod        │  │  MagicDNS enabled        │
              └───────────────┬───────────────┘  └──────┬───────────────────┘
                              │ 100.x.x.x                │ 100.x.x.x
                              │   ^                      │   |
                              │   | egress proxy (tailscale.com/expose)
                              │   ▼                      │   |
              ┌───────────────┴──────────────────────────┴───────────────┐
              │  k3s cluster                                              │
              │                                                         │
              │   prometheus-stack-grafana              ┌──► grafana    │
              │   prometheus-stack-kube-prometheus      │              │
              │        │ (Service + annotation)         │  proxy StatefulSet
              │        ▼                                │              │
              │   ts-prometheus-stack-grafana-...       └──► grafana pod │
              │   ts-prometheus-stack-...-prometheus-...            │
              └────────────────────────────────────────────────────────┘
```

The operator pod keeps its own `tailscaled` process running. Each annotated
Service spawns a dedicated proxy Pod (`ts-<service>-<hash>-0`) that also runs
`tailscaled`, presents a stable `100.100.x.x` address, and advertises the Service's
MagicDNS name to the tailnet. Proxies are tagged with the operator's tag
(`tag:k8s`); a per-service tag can be added via the `tailscale.com/tags` annotation.

## Components

| Namespace | Component | Kind | Purpose |
|-----------|-----------|------|---------|
| `tailscale-system` | `operator` | Deployment | Owns the operator device; reconciles Services/Ingress/Connectors |
| `tailscale-system` | `ts-prometheus-stack-grafana-8nqc5` | StatefulSet | Egress proxy for the Grafana Service |
| `tailscale-system` | `ts-prometheus-stack-kube-prom-prometheus-swf45` | StatefulSet | Egress proxy for the Prometheus Service |

## How It Was Installed

Helm was not present on the workstation. It was installed to `~/.local/bin` and
the `windows-amd64` subdirectory was added to `PATH`, matching the pattern used for
kubectl on the same machine.

```powershell
# 1. Helm ships as a zip whose contents live in a windows-amd64/ subdir
$binDir = "$env:USERPROFILE\.local\bin\windows-amd64"
# (download helm-v3.16.4-windows-amd64.zip, unzip, add to PATH)

# 2. Add the official chart repo
helm repo add tailscale https://pkgs.tailscale.com/helmcharts
helm repo update tailscale

# 3. Install the operator, authenticating with the OAuth client
helm upgrade --install tailscale-operator tailscale/tailscale-operator `
  --namespace tailscale-system `
  --create-namespace `
  --set-string oauth.clientId="$env:TAILSCALE_CLIENT_ID" `
  --set-string oauth.clientSecret="$env:TAILSCALE_CLIENT_SECRET" `
  --wait
```

The credentials come from a single OAuth client (a Trust credential in the Admin
console), **not** a long-lived auth key. The operator mints short-lived auth keys
from the OAuth client on start-up, so there is no key to rotate. The client is
`kJ5beeozNZ11CNTRL`.

## Authentication & The Tag Problem

The very first install succeeded at deploying, but the operator pod immediately
entered `CrashLoopBackOff`. The failure was not a connection problem — the operator
authenticated fine and connected to the tailnet. It failed when the operator tried
to register its device with an ACL tag:

```
{"level":"fatal","ts":"2026-10-24T02:15:21Z","logger":"startup",
 "msg":"creating operator authkey: requested tags [tag:k8s-operator] are
 invalid or not permitted (400)"}
```

The default chart applies the tag `tag:k8s-operator` to the operator's device at
startup. For Tailscale to accept that tag, two things must be true:

1. The OAuth client must have the write scope for **Tags**.
2. The tailnet policy must declare `tagOwners` for the tag, owned by a principal
   the operator is allowed to use.

The fix was a `tagOwners` block added to the tailnet policy in the Admin console
(**Settings → Policy → Policy editor**):

```json
"tagOwners": {
  "tag:k8s-operator": ["autogroup:admin"],
  "tag:k8s":          ["tag:k8s-operator"]
}
```

- `tag:k8s-operator` is owned by `autogroup:admin` (console users), so the operator
  device may wear it.
- `tag:k8s` is owned by `tag:k8s-operator`, so the operator may tag the proxy
  Pods it creates.

After the policy was saved, the operator device appeared in **Devices** as
`tailscale-operator`, tagged and connected. No change to the OAuth client's
scopes was needed beyond confirming **Tags Read** is granted.

### Getting the tag wrong

Two pitfalls were hit while getting this working:

- **Attaching the tag to the wrong place.** `tagOwners` alone is not enough — the
  operator still needs the tag attached to its own device. In the Admin console the
  OAuth client carries no tag by default, so the operator's auth-key request is
  rejected with the `400` above.
- **Relying on MagicDNS-only resolution.** The operator logs the tags it applies;
  watch `kubectl logs -n tailscale-system deployment/operator` for the fatal
  line if the pod will not stay up.

## Exposing Services

Services are exposed by annotating them, not by adding an Ingress. The operator's
`service-reconciler` picks up the annotation and creates a proxy StatefulSet.

```powershell
kubectl annotate service prometheus-stack-grafana -n monitoring \
  tailscale.com/expose="true" --overwrite

kubectl annotate service prometheus-stack-kube-prom-prometheus -n monitoring \
  tailscale.com/expose="true" --overwrite
```

The Prometheus service name is the Prometheus Operator naming
(`prometheus-stack-kube-prom-prometheus`), which differs from the naive
`prometheus-stack-kube-prometheus` and the older standalone `prometheus-operated`.
Use `kubectl get svc -n monitoring` to confirm the exact name before annotating.

To also tag a proxy (for ACL scoping), add the tailnet tag as an annotation:

```powershell
kubectl annotate service prometheus-stack-grafana -n monitoring \
  tailscale.com/tags="tag:monitoring" --overwrite
```

## MagicDNS Hostnames — The Namespace Prefix

The most common source of confusion is the hostname itself. A proxy's MagicDNS
name is `<namespace>-<service>.<tailnet-domain>.ts.net` — the namespace is **part
of the name**, not a path segment. For example, the Grafana proxy resolves to its
`100.x.x.x` address as

```
monitoring-prometheus-stack-grafana.taile4e2f7.ts.net
```

Visiting `prometheus-stack-grafana.taile4e2f7.ts.net` (without the
`monitoring-` prefix) returns "DNS not found": that name does not exist. The
`monitoring-` prefix is why the two probes differ.

### How to Verify

Resolve and curl the host from a device on the tailnet with MagicDNS enabled:

```powershell
# Correct name resolves and returns Grafana's 302 redirect
curl.exe http://monitoring-prometheus-stack-grafana.taile4e2f7.ts.net/
# HTTP 302  0.16s  -> http://monitoring-prometheus-stack-grafana.taile4e2f7.ts.net/

# Missing the namespace prefix and the name is simply unknown
curl.exe http://prometheus-stack-grafana.taile4e2f7.ts.net/
# curl: (6) Could not resolve host: prometheus-stack-grafana.taile4e2f7.ts.net
```

MagicDNS resolution happens inside Tailscale, not in the cluster's CoreDNS, so a
`nslookup` from inside a pod against CoreDNS will not resolve `.ts.net` — that is
expected, not a fault.

## Verifying Operator Health

```powershell
kubectl rollout status deployment/operator -n tailscale-system --timeout=90s

# Proxy StatefulSets should each show 1/1 ready
kubectl get statefulsets -n tailscale-system

# Operator pod should be running with a few restarts from the earlier crash loops
kubectl get pods -n tailscale-system
```

| Service | Namespace | MagicDNS Host |
|---------|-----------|---------------|
| Grafana | `monitoring` | `monitoring-prometheus-stack-grafana.taile4e2f7.ts.net` |
| Prometheus | `monitoring` | `monitoring-prometheus-stack-kube-prom-prometheus.taile4e2f7.ts.net` |

## Notes and Trade-offs

- **ACL by default.** Without a per-service `tailscale.com/tags` annotation, the
  proxies carry the `tag:k8s` tag, so reachability is scoped to anyone who can use
  `tag:k8s` per the tailnet ACL. For a homelab this is fine; add a dedicated tag
  and a `tagOwners` rule if you want to restrict the endpoint further.
- **Proxy Pods are egress, not ingress.** The operator's `ingress-pg-reconciler`
  also logs "ProxyGroup does not exist" for the public Ingress objects
  (`aoa-marching-cubes-ingress`, `catchall-404`, `jdools-com-ingress`,
  `mock-trading-ingress`, `power-playlist-*`). That is expected — those routes are
  served by Traefik, and they were annotated with `tailscale.com/expose` only
  in passing. Only Services were annotated for Tailnet exposure.
- **Helm version.** Helm v3.16.4 is the version installed on the workstation, not
  Helm 4.x. See the note in [monitoring.md](./monitoring.md) about the `routePrefix`
  MSYS2 path trap if you re-run the Prometheus install.
- **Re-registration on config churn.** Changing the operator's `nameOverride`/
  `fullnameOverride` creates a new device on the tailnet and leaves the old one
  behind. The default (`operator-oauth` secret) avoids this.

## Current State

- `tailscale-operator` release: **deployed** (chart revision 3 after the tag fix).
- Operator pod: `Running`; device tagged and connected.
- Proxy StatefulSets `ts-prometheus-stack-grafana` and
  `ts-prometheus-stack-kube-prom-prometheus`: `Running`, 1/1 ready.
- Grafana and Prometheus reachable at their MagicDNS hostnames from any tailnet
  device. See [monitoring.md](./monitoring.md) for the Ingress and port-forward
  alternatives.
