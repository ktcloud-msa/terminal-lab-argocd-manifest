# terminal-lab-argocd-manifest — repository layout

GitOps entry point for the **terminal-lab** platform, running on **OpenStack
on-premise** (bare-metal-style networking and storage — *not* AWS/EKS). ArgoCD's
`root-app` points at **`bootstrap/`** and everything else fans out from there
(app-of-apps).

```
bootstrap/        ArgoCD entry point — applied by root-app (path: bootstrap)
  projects.yaml       Application → applies projects/ first (sync-wave -10)
  platform.yaml       ApplicationSet → one Application per platform/* addon
  applications.yaml   ApplicationSet → the terminal-lab web + api workloads
projects/         ArgoCD AppProjects (RBAC / source & destination scoping)
  platform-project.yaml
  app-project.yaml
platform/         Cluster addons, operators, observability, security.
                  Folders are wave-numbered for readability; the real ordering
                  is driven by argocd.argoproj.io/sync-wave in bootstrap/platform.yaml.
applications/     The terminal-lab workload layer
  charts/web/         in-repo chart → web.external.terminal-lab.kr
  charts/api/         in-repo chart → api.external.terminal-lab.kr
docs/             This document.
```

## On-prem adaptation (vs. the AWS/EKS reference repo)

| Concern                | AWS/EKS reference        | terminal-lab (OpenStack on-prem) |
|------------------------|--------------------------|----------------------------------|
| LoadBalancer           | NLB/ALB + service annots | **MetalLB** (L2) `01-metallb` + `02-metallb-config` |
| Block/File storage     | `aws-ebs-csi` (gp3)      | **OpenEBS** `05-openebs` — `openebs-hostpath` (RWO) + `openebs-iscsi-rwx` (cStor iSCSI + NFS RWX) |
| Node autoscaling       | cluster-autoscaler (ASG) | dropped (no cloud ASG; scale via OpenStack/Cluster-API out of band) |
| Secrets                | external-secrets → AWS SM | dropped from this scope |
| Certificates           | (ACM / manual)           | **cert-manager** `40-cert-manager` + `41-cluster-issuer`, ACME **HTTP-01** via Traefik |
| IRSA (`role-arn`)      | per-SA IAM annotations   | removed — on-prem has no IRSA |

## Wave-number bands (`platform/`)

| Prefix | Concern                          | Apps |
|--------|----------------------------------|------|
| 01–02  | Bare-metal LoadBalancer          | `01-metallb`, `02-metallb-config` |
| 05     | Storage CSI                      | `05-openebs` |
| 10–13  | Service mesh (Envoy) + ingress   | `10-istio-base`, `11-istiod`, `13-traefik` |
| 30–31  | Observability                    | `30-kube-prometheus-stack`, `31-monitoring-config` |
| 40–41  | Operators / controllers / security | `40-cert-manager`, `41-cluster-issuer`, `40-gatekeeper`, `40-falco`, `40-falco-talon` |
| 91     | Policies / isolation             | `91-gatekeeper-policies`, `91-network-policies` |

### Sync-wave ordering (from `bootstrap/platform.yaml`)

- **Wave 0** — `metallb`, `openebs`, `istio-base` (foundation: LB, storage, mesh CRDs)
- **Wave 1** — `metallb-config`, `istiod`, `kube-prometheus-stack`, `cert-manager`, `gatekeeper`, `falco`
- **Wave 2** — `traefik` (needs a MetalLB VIP + ServiceMonitor CRD), `falco-talon`, `cluster-issuer`, `monitoring-config`
- **Wave 3** — `gatekeeper-policies`, `network-policies`

## Security stack

- **Gatekeeper** (`40-gatekeeper` + `91-gatekeeper-policies`) — admission policies:
  non-root, read-only root filesystem, and **dedicated ServiceAccount** (rejects
  any pod that uses or defaults to the `default` SA → enforces *1 workload — 1 SA*).
- **Falco** (`40-falco`) — eBPF runtime detection with custom rules for
  **crypto-mining** (known miner binaries / `stratum+tcp` markers, tag
  `crypto_mining`) and **persistence** (writes to cron / systemd / init /
  `authorized_keys` / `ld.so.preload`, tag `persistence`). Events are forwarded
  by Falcosidekick to Talon.
- **Falco Talon** (`40-falco-talon`) — response engine: on a `crypto_mining` or
  `persistence` critical event it **labels** the offending pod
  (`terminal-lab/quarantine=true`) and applies a **NetworkPolicy** that isolates
  it (deny all ingress/egress).
- **NetworkPolicies** (`91-network-policies`) — default-deny baseline in
  `terminal-lab`, with explicit allows for DNS, intra-namespace, Traefik ingress,
  and istiod egress.

## Ingress + TLS

Traefik (MetalLB-fronted) terminates TLS for two public hosts; cert-manager
issues the certs over ACME **HTTP-01** (solver Ingress on class `traefik`):

- `web.external.terminal-lab.kr` → `web` Service (`applications/charts/web`)
- `api.external.terminal-lab.kr` → `api` Service (`applications/charts/api`)

Each app chart owns its `Certificate`, `IngressRoute` (HTTP→HTTPS redirect +
HTTPS), and TLS secret — all in the `terminal-lab` namespace so Traefik can load
the secret from the same namespace as the route.

## Before you deploy — environment-specific values

- `platform/02-metallb-config/values.yaml` — set `addressPool.addresses` to a free
  VIP range on your OpenStack provider/tenant network (excluded from Neutron DHCP;
  L2 mode needs port-security/allowed-address-pairs to permit the announcements).
- `platform/05-openebs/values.yaml` — to use `openebs-iscsi-rwx`/`openebs-cstor-iscsi`,
  set `cstorPoolCluster.enabled: true` and list each node's block devices
  (`kubectl get bd -n openebs`).
- `platform/41-cluster-issuer/values.yaml` — switch to the LE **staging** ACME URL
  while testing to avoid prod rate limits.
- DNS — point `web.external.terminal-lab.kr` and `api.external.terminal-lab.kr`
  at the Traefik MetalLB VIP (required for HTTP-01 to validate).
- All `repoURL`s assume `https://github.com/kanei0415/terminal-lab-argocd-manifest.git`.
