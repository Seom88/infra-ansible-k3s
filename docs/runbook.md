# Runbook

Day-2 operations: verify, re-run, upgrade, and debug. Architecture rationale is in
[`architecture.md`](architecture.md); per-role inputs are in [`roles.md`](roles.md).

## First run

```bash
ansible-galaxy collection install -r requirements.yml   # k3s.orchestration tag 1.2.2
$EDITOR inventory/hosts.yml                             # ansible_host + ansible_user per node
ansible-playbook site.yml --syntax-check
ansible-playbook site.yml
```

What to expect, in order:

1. Base play (`base` on managed nodes, common firewall included) — mostly `ok`
   on a second run.
2. k3s play — `k3s_prereqs` + `kernel_modules` + Longhorn prereqs (all on
   `k3s_cluster`, before the server), then k3s server + agents — the upstream
   collection reports the cluster coming up.
3. Cilium — preflight fails early if helm/kubeconfig are missing.
4. Longhorn prereqs — may stop with **reboot required** if kernel modules are missing
   for the running kernel.
5. Argo CD — installs last, `wait` up to 900 s.
6. Virtualization play (`cockpit` on hypervisors, own `9090` rule).

> **Reboot contract:** nothing reboots itself. If a role reports `reboot required`,
> reboot the node (or use `ansible.builtin.reboot`) and re-run the playbook — the
> second run must report `ok`, not `changed`.

## Re-runs and idempotency

```bash
ansible-playbook site.yml --check --diff    # dry run
ansible-playbook site.yml                   # must converge to ~all ok
ansible-lint                                # production profile (.ansible-lint)
```

An idempotency violation (a task flipping `changed` on every run) is a **bug**: check
`changed_when` on Helm tasks and the pinned `helm-diff` plugin
(`helm plugin list` → `helm-diff 3.15.15`).

## Verification checklist

```bash
# Cluster + dataplane
kubectl --kubeconfig /etc/rancher/k3s/k3s.yaml get nodes
kubectl --kubeconfig /etc/rancher/k3s/k3s.yaml -n kube-system get pods
cilium status --wait                              # from a node with cilium CLI, or:
kubectl -n kube-system rollout status ds/cilium

# No kube-proxy, no flannel, no traefik
kubectl -n kube-system get ds | grep -E 'kube-proxy|flannel'   # expect: none
kubectl get pods -n kube-system -l app.kubernetes.io/name=traefik  # expect: none

# Gateway API
kubectl get gatewayclass,gateway

# GitOps
kubectl -n argocd get pods
kubectl get ciliumnetworkpolicy -n argocd

# Host firewall
firewall-cmd --list-all
firewall-cmd --check-config

# Kernel prerequisites
lsmod | grep -E 'br_netfilter|overlay|iscsi_tcp|dm_crypt'
sysctl net.bridge.bridge-nf-call-iptables net.ipv4.ip_forward
```

## Common failures

### `assert` failed: cockpit package not installed

By design — the role never installs Cockpit. Install it with your package manager
(`dnf install cockpit cockpit-machines`), start `cockpit.socket`, re-run.

### `reboot required` from `kernel_modules` or `longhorn_prereqs`

The module is missing **for this kernel** (`modinfo` finds nothing and it is not in
`modules.builtin`). Reboot into the kernel that ships it (e.g. after `kernel-modules-extra`
install) and re-run. If it appears in `modules.builtin`, it is built-in — no reboot
needed; the roles distinguish these cases.

### Cilium preflight: `helm` or kubeconfig missing

Run the k3s play first (`site.yml` covers it) — Cilium intentionally refuses to
bootstrap out of order.

### Cilium installs but L7/Envoy + Tailscale breaks DNS/API calls

The `accept_local` drop-in (`/etc/sysctl.d/90-accept_local.conf`) is missing or was
overwritten. Re-run the `cilium` role, verify:

```bash
sysctl net.ipv4.conf.all.accept_local          # expect: 0
sysctl net.ipv4.conf.lxc0.accept_local         # expect: 1 (per-endpoint)
```

Upstream tracker: [cilium/cilium#47591](https://github.com/cilium/cilium/issues/47591).

### Gateway API CRD apply conflicts

`--server-side --force-conflicts` is there on purpose (field-manager conflicts from
earlier installs). If `kubectl apply` still fails, check CRD versions:
`kubectl get crd gateways.gateway.networking.k8s.io -o jsonpath='{.spec.versions[*].name}'`
should include `v1`.

### Argo CD UI unreachable from the tailnet

Strict mode: everything is denied unless allowed. Check both policy layers:

```bash
kubectl get ciliumnetworkpolicy -n argocd      # both policies must exist
kubectl -n argocd describe netpol              # default-deny is expected; CNPs are the allows
kubectl -n tailscale get pods                  # cluster-gateway pod must be Running
```

Remember the port-forward pattern: port-forwards target `8080/8083/8082`, **not** 80/443
(those belong to the Gateway).

### Helm task reports `changed` every run

1. `helm plugin list` → confirm `helm-diff 3.15.15`.
2. Inspect the task's `changed_when` — it must key off the diff, not task success.

## Upgrades

Pinned versions move deliberately — bump one pin at a time, re-run, verify:

| Pin | Where |
| --- | --- |
| k3s | `inventory/group_vars/all/main.yml` → `k3s_version` |
| Cilium chart | `roles/cilium/defaults/main.yml` → `cilium_chart_version` |
| Gateway API CRDs | `roles/cilium/defaults/main.yml` → `cilium_gateway_api_version` |
| Argo CD chart | `roles/argocd/defaults/main.yml` → `argocd_chart_version` |
| helm-diff | `roles/k3s_prereqs/defaults/main.yml` |
| k3s-orchestration | `requirements.yml` → tag `1.2.2` |
| Helm values | `values/cilium/values.yaml`, `values/argocd/values.yaml` |

Suggested order: **k3s → Cilium (chart, then CRDs) → Argo CD**. After each step run the
verification checklist; the embedded-etcd `cluster-init` setup takes server upgrades one
node at a time.

## Adding a node

1. Add an `agent` host under `k3s_cluster` in `inventory/hosts.yml`.
2. Run `ansible-playbook site.yml` — the base play baselines it and the k3s
   play covers prereqs, agent join, and Longhorn prerequisites automatically.
3. Verify: `kubectl get nodes` shows the new node `Ready`.

## Nightly hygiene

- `git status` — clean tree; `main` is the deliverable.
- Re-run `site.yml` after any hand-edit on a node: **the playbook is the source of
  truth**, hand-edits are drift.
- `ansible-lint` + `ansible-playbook site.yml --syntax-check` before every commit.
