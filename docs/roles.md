# Roles

Per-role reference: what each role consumes, what it does, and which asserts fail loudly.
Play order is defined in [`../site.yml`](../site.yml) — three `import_playbook`
entries (`base`, `k3s`, `virtualization`), one play per function in
[`../plays/`](../plays/). Shared tunables live in
[`../inventory/group_vars/all/main.yml`](../inventory/group_vars/all/main.yml).

Order on the wire:

```
base → k3s (k3s_prereqs → kernel_modules → longhorn_prereqs → k3s server
       → k3s agents → cilium → argocd) → virtualization (cockpit)
```

Firewall model: **no single firewall role.** `base` owns only the common
firewall (daemon running, Tailscale port/interface, masquerade); every other
rule lives in the role that needs it (`k3s_prereqs`, `cockpit`). A future
docker-only node would get `base` + its own role and zero k3s packages.

---

## `base`

**Purpose:** turn a clean AlmaLinux 9/10 host into a managed node — generic
only, no k3s payload.

| | |
| --- | --- |
| **Runs on** | all managed nodes (`plays/base.yml`) |
| **Asserts** | distribution in AlmaLinux family, major in `[9, 10]` |
| **Installs** | `curl`, `tar`, `git`, `chrony` (time sync) |
| **Verifies** | cgroup v2 marker `/sys/fs/cgroup/cgroup.controllers`; records SELinux status as a fact for later roles |
| **Common firewall** | ensures the firewall daemon (`firewalld` service) is running; opens `41641/udp` (Tailscale) in `base_firewall_zone` (`public`); trusts `tailscale0` by interface; masquerade on; validates with `firewall-cmd --check-config` before reload |
| **Switch** | `base_firewall_enabled` — other roles skip their own rules when it is false |

Helm, `helm-diff`, `kernel-modules-extra` and `socat` used to live here and
were moved to `k3s_prereqs`: a docker-only or hypervisor-only node must not
pay for the k3s/Longhorn ecosystem.

## `k3s_prereqs`

**Purpose:** everything a k3s host needs before `k3s_server` runs — the k3s
payload that `base` no longer carries, plus the k3s firewall.

| | |
| --- | --- |
| **Runs on** | `k3s_cluster` (first in `plays/k3s.yml`, before `kernel_modules` + Longhorn prereqs + `k3s_server`) |
| **Installs** | `socat` (k3s/Longhorn-ecosystem troubleshooting only — grep shows no other consumer in this repo) |
| **Helm** | official `get-helm-3` script → `/usr/local/bin` |
| **helm-diff plugin** | pinned **3.15.15** — required for `kubernetes.core.helm` idempotency checks; `git` comes from `base` so the plugin clone cannot fail |
| **kernel-modules-extra** | version-pinned on Alma 10 (`k3s_prereqs_kernel_modules_extra_required_major`), unversioned fallback elsewhere |
| **k3s firewall** | `6443/tcp`, `10250/tcp`, `8472/udp`, `4240/tcp`, `80/tcp`, `443/tcp` in `k3s_prereqs_firewall_zone` (defaults to `base_firewall_zone`); pod CIDR `10.42.0.0/16` + service CIDR `10.43.0.0/16` as trusted sources; `k3s_prereqs_extra_ports: []` extension point; validated with `firewall-cmd --check-config` |
| **Gating** | skips its firewall block when `base_firewall_enabled` is false (base runs first and guarantees the daemon) |

**Why the pin matters:** unpinned `helm-diff` upgrades changed diff output semantics and
made `helm` tasks flap between `changed`/`ok`. The pin is the idempotency contract.

**Why no separate `k3s_firewall` role:** same hosts, same lifecycle, ~15 lines —
an extra role would only add indirection.

## `kernel_modules`

**Purpose:** kernel-level prerequisites with an honest reboot story.

| | |
| --- | --- |
| **Runs on** | `k3s_cluster` (`plays/k3s.yml`) |
| **Modules** | `br_netfilter`, `overlay` — persisted via template `k3s.conf.j2` → `/etc/modules-load.d/k3s.conf` |
| **Sysctls** | one canonical file `99-k3s.conf`: `bridge-nf-call-iptables/ip6tables=1`, `ip_forward=1`, inotify limits `8192 / 524288 / 16384` |
| **Detection** | `modinfo` vs `modules.builtin`: missing for this kernel → **reboot-required** report; built-in → OK; present → ensure loaded |
| **Cleanup** | obsolete drop-in `99-inotify.conf` removed (folded into `99-k3s.conf`) |

No forced reboots: the playbook reports and stops short of pretending the node is ready.

## `cockpit`

**Purpose:** host management surface — asserted, not installed.

| | |
| --- | --- |
| **Runs on** | `hypervisors` (`plays/virtualization.yml`, after `base` in `site.yml`) |
| **Asserts** | `cockpit` package **present** (role never installs it — your package policy, your call) |
| **Probes** | `cockpit.socket` / `cockpit.service` via `systemctl` (`service_facts` cannot see socket units) |
| **Firewall** | owns its rule: opens `9090` only after the probe succeeds (`service: cockpit` zone rule in `cockpit_firewall_zone`, defaulting to `base_firewall_zone`); skipped when `base_firewall_enabled` is false — base runs first and guarantees the daemon |
| **libvirt** | when `cockpit_install_libvirt: true`: installs `cockpit-machines`, `libvirt`, `qemu-kvm`, `virt-install`; probes both monolithic (`libvirtd`) and modular (`virtqemud`, `virtstoraged`, `virtnetworkd`, `virtnodedevd`, …) socket units and **asserts the required modular sockets are listening** |
| **Storage pool** | `default` pool at `/home/libvirt/images`, `autostart: true`; SELinux fcontext **`virt_image_t`** + `restorecon`; idempotent `virsh define/build/start/autostart` (raw `virsh`, not `community.libvirt`) |

## `cilium`

**Purpose:** the cluster datapath — installed in a strict order.

1. **Preflight:** fail loudly if `helm` or `/etc/rancher/k3s/k3s.yaml` is missing
   ("run the k3s play first") — no half-installed Cilium.
2. **Gateway API CRDs v1.6.1** (standard channel): `kubectl apply --server-side
   --force-conflicts` *before* the chart — Cilium does not ship them, and server-side
   apply makes re-runs converge despite field-manager conflicts.
3. **Helm release:** chart **1.20.2**, namespace `kube-system`, `wait` 600 s,
   `history_max: 10`, values loaded from `values/cilium/values.yaml`.
4. **Tailscale × L7 workaround:** drop-in `90-accept_local.conf`
   (`all.accept_local=0`, `lxc*.accept_local=1`) for
   [cilium#47591](https://github.com/cilium/cilium/issues/47591).

Key values (justification in [`architecture.md`](architecture.md)):

| Value | Why |
| --- | --- |
| `ipam.mode: cluster-pool`, `10.42.0.0/16` | matches `cluster-cidr` |
| `k8sServiceHost: 192.168.2.201:6443` | LAN IP, not `127.0.0.1` — L7/Envoy fix |
| `kubeProxyReplacement: true` | k3s runs with `disable-kube-proxy: true` |
| `socketLB.hostNamespaceOnly: true` | correct service resolution from host-namespace pods |
| `gatewayAPI.enabled` + `enableAlpn`/`AppProtocol` | Gateway API datapath |
| `operator.replicas: 1` | single-node reality; HA comes from the GitOps profile |
| Hubble + relay + UI, Prometheus metrics/dashboards, Envoy | observability from day zero |

## `longhorn_prereqs`

**Purpose:** make nodes storage-capable; the Longhorn release itself is GitOps-owned.

| | |
| --- | --- |
| **Runs on** | `k3s_cluster` (`plays/k3s.yml`, before `k3s_server`) |
| **Asserts** | `data_engine == v1`; data path (`longhorn_prereqs_data_path`, default `/mnt/data`) is `ext4` or `xfs` via `findmnt` (enforce flag) |
| **Packages** | `iscsi-initiator-utils`, `nfs-utils`, `cryptsetup`, `device-mapper`, `xfsprogs` |
| **Modules** | `iscsi_tcp`, `dm_crypt` — template `longhorn.conf.j2` → `/etc/modules-load.d/longhorn.conf` |
| **Ordering** | packages installed **before** `modinfo` probing — avoids a false "reboot required" on a fresh node |
| **Verification** | two-stage: `modinfo` vs `modules.builtin` (missing-for-kernel → reboot report) then `lsmod` (available but not loaded → assert distinct error and load); `iscsid` enabled; package versions reported |

## `argocd`

**Purpose:** GitOps control plane, network-locked from the start.

| | |
| --- | --- |
| **Runs on** | first server |
| **Preflight** | same fail-loud helm/kubeconfig check as `cilium` |
| **Helm release** | chart **10.9.2** (`argoproj.github.io/argo-helm`), namespace `argocd`, `wait` 900 s, values from `values/argocd/values.yaml` |
| **Prod shape (HA)** | controller ×2; server + repoServer HPA `minReplicas: 2`; applicationSet ×2 |
| **Dev overlay** | `values/argocd/values-dev.yaml`: 1+1+1 with resource limits for an 8 GiB node |
| **Health** | custom **Lua** checks: `Application` wave-policy labels (`healthy`/`sync-only`), `ClusterSecretStore`/`ExternalSecret`, `Prometheus` |
| **Ingress** | `server.insecure: true` — TLS at the Cilium Gateway, strip mode, path `/argocd` |
| **NetworkPolicy** | **kept enabled** (Cilium strict mode ⇒ explicit allows, `defaultDenyIngress: false`) |
| **CiliumNetworkPolicies** | `argocd-allow-ingress-portforward` (kube-apiserver/host/remote-node → 8080/8083/8082; the port-forward pattern — never 80/443 on pod ports) and `argocd-allow-ingress-tailscale` (tailnet gateway pod → `argocd-server:8080`) |

---

## Conventions every role follows

- **Defaults-driven:** tunables in `defaults/main.yml` + `inventory/group_vars/all/main.yml`,
  never hard-coded in tasks.
- **Firewall ownership:** `base` owns the common firewall only; each role owns
  its own rules and gates them on `base_firewall_enabled`. Never centralize
  per-service ports in one role again.
- **Stable paths:** resolve repo files relative to `inventory_dir`
  (as `cilium`/`argocd` do for `values/`), never `playbook_dir` — the latter
  breaks when plays move under `plays/`. `group_vars` lives next to the
  inventory (`inventory/group_vars/`), not at the repo root.
- **Fail-loud:** missing prerequisites `assert` with an actionable message
  (what's missing, what to run, whether a reboot is needed) instead of continuing.
- **Honest `changed_when`:** re-runs report `ok`, not `changed` — enforced with the
  pinned `helm-diff`, `systemctl is-active` probes, and `creates`/`removes` guards.
- **No secrets in git:** no tokens, no kubeconfigs — k3s generates credentials and the
  role only reads the root kubeconfig in place (`0600`).
