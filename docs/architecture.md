# Architecture

How the cluster is put together, which planes exist, and — most importantly — *why* each
non-obvious decision was made. Companion docs: [`roles.md`](roles.md) for per-role
reference, [`runbook.md`](runbook.md) for day-2 operations.

## Topology

```
                      ┌────────────────────────────────────────────┐
   Ansible control    │  site.yml (3 imports, ordered)             │
   node (your laptop/ │   1. base play: assert + base (common      │
   bastion)           │      firewall) on managed_nodes            │
        │ SSH         │   2. k3s play on k3s_cluster: k3s_prereqs  │
        │ become      │      → kernel_modules → longhorn_prereqs   │
        │ pipelining  │      → k3s server (k3s.orchestration 1.2.2)│
        └────────────►│      → k3s agents → Cilium → Argo CD       │
                      │   3. virtualization play: cockpit on       │
                      │      hypervisors                           │
                      └───────────────────┬────────────────────────┘
                                          │
                 ┌────────────────────────▼───────────────────────┐
                 │  k3s-01  (AlmaLinux, ansible_host = Tailscale  │
                 │          IP 100.96.189.24)                     │
                 │                                                │
                 │  k3s server ──── Cilium 1.20.2 ──── Argo CD    │
                 │  (cluster-init,    (kube-proxy replacement,    │
                 │   secrets-enc,      Gateway API, Hubble,       │
                 │   no traefik)        L7 policies)              │
                 │                                                │
                 │  base common firewall + k3s_prereqs k3s rules  │
                 │  (public zone)   cockpit + libvirt            │
                 │  kernel: br_netfilter, overlay, iscsi_tcp,     │
                 │          dm_crypt + sysctls                    │
                 └────────────────────────────────────────────────┘
```

`inventory/hosts.yml` defines `k3s_cluster` with `server` and `agent` groups; the
single server today is `k3s-01`. `api_endpoint` is derived with Jinja from the first
server host, so adding a second server does not require editing the value by hand.
`managed_nodes` wraps `k3s_cluster` and drives the baseline plays.

## Network planes

Four planes coexist on one NIC. Keeping them conceptually separate is what makes the
firewall rules explicable:

| Plane | CIDR / identity | Carried by | Firewall treatment |
| --- | --- | --- | --- |
| Pod network | `10.42.0.0/16` | Cilium (cluster-pool IPAM) | trusted **source** on public zone |
| Service network | `10.43.0.0/16` | Cilium (kube-proxy replacement) | trusted **source** on public zone |
| Node LAN | e.g. `192.168.2.0/24` | Ethernet | k3s/Cockpit ports opened explicitly; advertised into the tailnet as a route |
| Tailnet | `100.64.0.0/10` | `tailscale0` interface | trusted **by interface**, not by CIDR |

Why interface-based trust for Tailscale: tailnet IPs are assigned dynamically and change
across re-auths; the `tailscale0` interface name is stable for as long as Tailscale runs.
Trusting the whole `/10` as a source would also outlive the interface it arrived on.
Pod interfaces (`lxc*`) are ephemeral per endpoint, so pods keep CIDR-based source trust —
the hybrid is deliberate: *trust identity where identity is stable, CIDR where it is not.*

`base` makes the LAN reachable **from the tailnet** with two mechanisms that belong to
different layers:

- `tailscale set --advertise-routes=192.168.2.0/24` — the node becomes a subnet router, so
  the tailnet can reach the LAN. Publishing a route is not the same as using it: it must be
  approved in the Tailscale admin console (or allowed by an `autoApprovers` tailnet policy
  rule), and this repo never touches the tailnet policy.
- `net.ipv4.ip_forward=1` — what actually moves packets between `tailscale0` and the LAN.
  `base` keeps it in `/etc/sysctl.d/99-tailscale.conf` so a hypervisor-only node is a
  working router without running the k3s play; `kernel_modules` writes the same key into
  the canonical `99-k3s.conf` for cluster nodes.

The other direction is *not* symmetrical: the LAN is not a second entrance to the tailnet.
`tailscale0` stays a trusted interface, so a tailnet identity reaching the LAN is still
authenticated by Tailscale, not by being on the switch.

Ports opened in the public zone, each owned by the role that needs it
(no central firewall role): `base` opens `41641/udp` (Tailscale), trusts
`tailscale0` by interface and enables masquerade; `k3s_prereqs` opens
`6443/tcp` (apiserver), `10250/tcp` (kubelet), `8472/udp` (VXLAN fallback),
`4240/tcp` (Cilium health), `80/tcp` + `443/tcp` (Gateway) and trusts the pod /
service CIDRs as sources; `cockpit` opens `9090/tcp` (service rule, only after
the service is probed active). Every change is validated with
`firewall-cmd --check-config` before reload.

## k3s configuration

Pinned in `inventory/group_vars/all/main.yml`: **`k3s_version: v1.36.5+k3s1`**.

`server_config_yaml` turns k3s into "just the orchestration core":

| Setting | Value | Reason |
| --- | --- | --- |
| `flannel-backend` | `none` | Cilium owns pod networking; two overlays fight over packets |
| `disable-network-policy` | `true` | Cilium implements policy, not k3s's kube-router fork |
| `disable-kube-proxy` | `true` | Cilium `kubeProxyReplacement: true` |
| `disable` | `traefik` | Gateway API via Cilium replaces the bundled ingress |
| `cluster-init` | `true` | Embedded etcd bootstrap, no external datastore |
| `secrets-encryption` | `true` | Encrypt k8s Secrets at rest in the embedded store |
| `supervisor-metrics` | `true` | Scrapeable supervisor metrics |
| `write-kubeconfig-mode` | `"0600"` | kubeconfig readable by root only |

Cilium's `k8sServiceHost` points at the node's **LAN IP** (`192.168.2.201:6443`), not
`127.0.0.1` — the loopback shortcut broke L7/Envoy traffic when combined with Tailscale
(see the workaround below).

## Cilium + Tailscale: the one real fight

Two kernel-level features collided:

1. **Tailscale sets `net.ipv4.conf.*.src_valid_mark = 1`**, marking locally-originated
   packets for policy routing.
2. **Cilium's L7 proxy (Envoy)** returns traffic whose source is a local address, which
   the kernel drops/reorders under `accept_local` semantics.

Upstream issue [cilium/cilium#47591](https://github.com/cilium/cilium/issues/47591).
The fix shipped in this repo (`roles/cilium`):

- Drop-in `/etc/sysctl.d/90-accept_local.conf`:
  - `net.ipv4.conf.all.accept_local = 0` (host-facing, restores Tailscale's expectations)
  - `net.ipv4.conf.lxc*.accept_local = 1` (Cilium endpoint interfaces still accept
    local-sourced returns)
- `k8sServiceHost` moved from `127.0.0.1` to the node LAN IP so API traffic rides the
  normal datapath instead of loopback.

This is recorded as a **scoped workaround for an upstream bug**, not a design preference —
if the upstream fix lands and is backported to the pinned chart, the drop-in can go.

### Gateway API

Cilium does not ship Gateway API CRDs. `roles/cilium` applies the **standard channel
CRDs v1.6.1** with `kubectl apply --server-side --force-conflicts` *before* the Helm
release, so `gatewayAPI.enabled` never races a missing CRD, and re-runs converge despite
field-manager conflicts from previous installs.

## GitOps model

Argo CD is installed by Ansible (wave 0: cluster bootstrap) and then expects to be
managed from Git:

- **Wave policy** is encoded as labels/annotations (`healthy` / `sync-only`) and
  evaluated by custom **Lua health checks** for `Application`,
  `ClusterSecretStore`/`ExternalSecret`, and `Prometheus` resources — resources Argo CD
  does not know out of the box.
- **`server.insecure: true`**: TLS terminates at the Cilium Gateway (strip mode),
  matching Tailscale Serve at `https://my-cluster.lonk-mirfak.ts.net/argocd/` →
  `argocd-server:80`. No in-cluster cert rotation.
- **NetworkPolicies stay enabled** (Cilium strict mode) with two explicit allows:
  - `argocd-allow-ingress-portforward`: from kube-apiserver/host/remote-node to
    `8080/8083/8082` (Argo CD's port-forward pattern — deliberately *not* 80/443,
    which are Gateway-only).
  - `argocd-allow-ingress-tailscale`: from the `tailscale` namespace's
    `cluster-gateway` pod (plus namespace-only fallback) to `argocd-server:8080`.

Ansible's job stops at "Argo CD exists and can reach Git"; application syncs are the
GitOps layer's job.

## Storage

Longhorn itself is *not* installed by this playbook — only its **node prerequisites**
(`roles/longhorn_prereqs`), because the release belongs in GitOps:

- packages: `iscsi-initiator-utils`, `nfs-utils`, `cryptsetup`, `device-mapper`, `xfsprogs`
- kernel modules: `iscsi_tcp`, `dm_crypt` (persisted via `/etc/modules-load.d/longhorn.conf`)
- data path (`longhorn_prereqs_data_path`, default `/mnt/data`) must be **ext4 or xfs**,
  asserted with `findmnt` — Longhorn's `data_engine: v1` requires a real filesystem,
  not a bind mount over something exotic
- verification is two-stage: `modinfo` vs `modules.builtin` (missing for this kernel →
  **reboot required** report) then `lsmod` (available but not loaded → load it)
- packages are installed **before** the `modinfo` probe so a fresh node doesn't produce a
  false "reboot required"

## Host management surface

`roles/cockpit` deliberately **asserts rather than installs**: the host must already
provide Cockpit (your package policy, your choice). What the role owns:

- probe `cockpit.socket`/`cockpit.service` with `systemctl` — `service_facts` does not
  see socket units — then open `9090` in the host firewall (firewalld daemon,
  `cockpit` service rule)
- libvirt stack (`libvirt`, `qemu-kvm`, `virt-install`) when `cockpit_install_libvirt`
  is true; probe both monolithic (`libvirtd`) and modular (`virtqemud`, `virtstoraged`,
  `virtnetworkd`, `virtnodedevd`) socket units and assert the required ones listen
- storage pool `default` at `/home/libvirt/images`, autostart, with the correct SELinux
  fcontext **`virt_image_t`** (verified: `qemu_image_t` is *wrong* for `virt_image_t`
  paths — `semanage fcontext` rejects the mistaken pattern, and `restorecon` is applied
  idempotently)
- **publishing the console in the tailnet** with `tailscale serve --bg --yes
  https+insecure://localhost:9090`, behind the same package assertion (see
  [`roles.md`](roles.md#cockpit) for why `https+insecure` and how the change signal is
  computed). This is not in `plays/virtualization.yml`: the play only lists roles, and the
  tunnel is a property of "this node has Cockpit", not of the play that happens to drive it.

## Decision log

Selected entries from the commit history (full history: `git log --oneline`):

| Commit | Decision |
| --- | --- |
| `a8adc64` | Helm releases driven from `values/`; token ceremony dropped — k3s handles tokens |
| `8660f13` | Helm + namespaces bootstrapped on first run (no chicken-and-egg) |
| `a63b5f8` | `helm-diff` pinned to `3.15.15` so idempotency checks can't drift upstream |
| `411b495` | libvirt storage pool managed from the Cockpit role (idempotent `virsh`) |
| `63fb0ca` + `b09adec` | Gateway API CRDs installed pre-chart with `--force-conflicts` |
| `6a4bd9f` | Tailscale trusted by interface; pod/service CIDRs as sources |
| `0f50c91` | L7 Envoy × Tailscale fix: `accept_local` drop-in + LAN `k8sServiceHost` |
| `8750b60` | Obsolete sysctl drop-ins removed; one canonical `99-k3s.conf` |
| encapsulated-roles | central firewall role deleted; `base` keeps only the common firewall, `k3s_prereqs` (new) and `cockpit` own their rules — per-role firewall, no central writer |

## What I'd do differently

Honest post-mortem notes, because a living doc should not pretend the path was straight:

- **Pin the upstream collection earlier.** Early runs pulled moving `k3s.orchestration`
  defaults; the tag pin (`1.2.2`) now protects us, but it arrived after two breakages.
- **`changed_when` discipline from commit one.** Several fix commits exist only because
  tasks reported `changed` on every run (Helm install without `changed_when`, deprecated
  `wait_timeout`). Every task now has an honest change signal — new tasks must too.
- **Probe sockets, not `service_facts`.** systemd *socket* units are invisible to
  `service_facts`; two separate fixes (Cockpit, libvirt) rediscovered this.
- **The dev overlays (`values-*.yaml`) are reference profiles**, not yet wired into the
  roles — selecting an overlay per inventory group is a natural next step.
