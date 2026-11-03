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
7. Power play (`scheduled_shutdown` on the server) — arms the nightly 22:00
   `America/Bogota` power-off; see [Nightly power-off](#nightly-power-off).

> **Reboot contract:** nothing reboots itself. If a role reports `reboot required`,
> reboot the node (or use `ansible.builtin.reboot`) and re-run the playbook — the
> second run must report `ok`, not `changed`. The one scheduled stop is the nightly
> power-off, documented under [Nightly power-off](#nightly-power-off).

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

## Nightly power-off

`roles/scheduled_shutdown` owns one systemd timer on the cluster server
(`plays/power.yml`, `hosts: server`). Power-on is out of band — BIOS RTC wake — so
nothing in the repo brings the node back.

```bash
# what is armed, and when it fires next
systemctl list-timers scheduled-shutdown.timer
systemd-analyze calendar "*-*-* 22:00:00 America/Bogota"

# cancel tonight without touching the repo
sudo systemctl stop scheduled-shutdown.timer

# re-arm with the configured calendar
ansible-playbook plays/power.yml

# inspect the units and the wall message
systemctl cat scheduled-shutdown.timer scheduled-shutdown.service
```

`Persistent=false` on purpose: a window missed while the node was off is not
replayed at boot. Setting `scheduled_shutdown_enabled: false` stops, disables and
**removes** both units — a flipped switch must not leave a live timer behind.

The stop is graceful (`shutdown -h now`), so k3s gets its stop timeout instead of
a hard reset; Longhorn volumes reattach and pods restart after the next boot.

## ZFS on the host (NAS group)

`plays/nas.yml` runs the `zfs` role on hosts in the `nas` group. Add the host,
then configure existing pools to import, or new pools to create, using
`/dev/disk/by-id/` devices. These are distinct operations: importing preserves
the pool; `zfs_pools` invokes `zpool create` and must only be used for blank/new
disks.

```bash
ansible-playbook plays/nas.yml --check --diff   # see the plan first
ansible-playbook plays/nas.yml
```

Aleph's existing `Data` mirror is listed under `zfs_import_pools`,
identified by an attribute of the drive itself: `ata-<MODEL>_<SERIAL>` or
`wwn-0x...`. Do not use the `usb-*` names: both USB-SATA bridges report the same
generic serial, so those depend on the model string and the port.

ZFS members carry raw ZFS labels, not `blkid` UUIDs; a vdev is identified by the
GUIDs in its label (`zdb -l /dev/sdb1` prints them). Import scans
`/dev/disk/by-id` and matches those GUIDs, so a renamed device does not break
it: the configured paths are an existence guard and the path recorded in
`zpool status` is only a hint. The role imports only the named pool (with `-f`,
because it was last accessed by the TrueNAS VM) and never creates or formats
disks. Inspect the pool and its datasets after import before declaring any
dataset or mountpoint.

For Aleph, the pool uses native ZFS encryption (`aes-256-gcm`, `keyformat=hex`).
Keep the exported `Data` key as the `zfs_dataset_key` variable in the locally
ignored `inventory/group_vars/nas/vault.yml` file, encrypted with Vault. The role
writes it to `/root/.config/zfs/Data.key` (`root:root`, mode `0600`) without
logging its contents, then installs a systemd unit that loads the key after pool
import and before ZFS mounts datasets. Run with the Vault password prompt:

```bash
ansible-playbook plays/nas.yml --ask-vault-pass
```

The Vault file is ignored by Git in this repository; keep a secure backup of
both it and the original TrueNAS key export. The pool key is necessarily present
in plaintext on Aleph for unattended unlock, so root access to Aleph grants
access to the data.

### RustFS with Podman

RustFS is configured as a host service using Podman Quadlet (no Docker daemon or
Compose provider). Add these values to the encrypted
`inventory/group_vars/nas/vault.yml` file without displaying them:

```yaml
rustfs_ts_authkey: "<Tailscale auth key>"
rustfs_access_key: "<RustFS access key>"
rustfs_secret_key: "<RustFS secret key>"
```

Then set `rustfs_enabled: true` in `inventory/group_vars/nas/main.yml` and run:

```bash
ansible-playbook plays/nas.yml --ask-vault-pass
```

Quadlet shares the pod network between `rustfs-ts` and `rustfs`; only UDP
41642 is published on the host (41641 belongs to Aleph's host Tailscale daemon),
while the existing Serve configuration proxies to ports 9000 and 9001. Existing data at `/Data/Docker/rustfs/data` is reused,
not erased. Podman healthchecks with systemd restart unhealthy containers,
replacing the Compose `autoheal` sidecar.

Boot chain: `zfs-import-cache` → `zfs-unlock-pool` → `zfs-mount` → `rustfs-pod`
→ `rustfs-ts` → `rustfs`. The pod unit `Requires=zfs-unlock-pool.service`; each
container unit orders itself after the mount of its volumes. Nothing runs
`systemctl enable`: Quadlet regenerates the transient units from
`/etc/containers/systemd` and applies their `[Install] WantedBy=` on every boot.

#### `rustfs.service` fails at boot with `result 'timeout'`

Quadlet starts the container with `podman run`. Mounting the data volume with
`:Z` made Podman relabel the whole dataset (>100k files on USB HDDs) on every
start, exceeding systemd's default `TimeoutStartSec=90 s`; `ExecStopPost` then
removed the container while `Restart=always` retried, so `podman ps` was empty
and the Tailscale sidecar logged `connection refused` on 127.0.0.1:9000.

The volume is mounted without `:Z`, the container runs with
`SecurityLabelDisable=true` (it is the only writer), and both units set
`TimeoutStartSec` to `rustfs_start_timeout` (300 s). Restart `rustfs-ts.service`
and `rustfs.service` after applying, or reboot to exercise the real boot path.
Delete stale `*.bak` files in `/etc/containers/systemd`.

### Enroll the signing key (manual, once)

Secure Boot is enabled on the target hosts and AlmaLinux does not sign
third-party modules, so the role generates a local MOK and signs the modules,
but the firmware must be told to trust that key. This is deliberately manual:

```bash
sudo mokutil --import /var/lib/zfs-mok/MOK.der   # choose a one-time password
sudo reboot
# In the blue MokManager screen: Enroll MOK -> Continue -> Yes -> password -> Reboot
```

Then confirm both the enrollment and the module:

```bash
mokutil --list-enrolled | grep -i "ZFS module signing"
modinfo -F signer zfs
sudo modprobe zfs && zpool status
```

Re-run `plays/nas.yml` afterwards: it reports `mok_enrolled: true` and loads the
module. Kernel and `kmod-zfs` updates re-sign automatically through the dnf
`post-transaction-actions` hook; no manual signing after the first enrollment.

## Nightly hygiene

- `git status` — clean tree; `main` is the deliverable.
- Re-run `site.yml` after any hand-edit on a node: **the playbook is the source of
  truth**, hand-edits are drift.
- `ansible-lint` + `ansible-playbook site.yml --syntax-check` before every commit.

### USB passthrough into a TrueNAS VM dropped (`device undefined bus undefined`)

Symptoms from the host (e.g. `Aleph`) and VM:

- the Realtek RTL9201R bridge (`0bda:9201`) is under `usbfs` and QEMU
  pins `/dev/bus/usb/<bus>/<addr>`; after a reset the kernel number changes;
- TrueNAS ZFS pool turns `DEGRADED`/`FAULTED`, drives come back as
  `device undefined bus undefined`.

Apply:

1. Use managed hostdev in the VM XML (vendor/product instead of a raw
   `/dev/bus/usb/...` path), so libvirt redetects the bridge after reset.
2. Disable the aggressive autosuspend with `roles/usb_autosuspend`:
   ```bash
   ansible-playbook plays/virtualization.yml --tags usb_autosuspend
   ```
   or rerun `site.yml`; the role is part of the virtualization play.
   Verify the live setting, not just the file:
   ```bash
   for d in /sys/bus/usb/devices/*; do
     [ -r "$d/idVendor" ] || continue
     [ "$(cat "$d/idVendor"):$(cat "$d/idProduct")" = 0bda:9201 ] || continue
     echo "$d control=$(cat "$d/power/control")"
   done
   ```
   The role now fails the play if an attached bridge still reports
   `power/control=auto` after reloading and triggering udev.
3. Move the enclosures to a real USB 3 port (5000M+) and give them dedicated
   power — a shared low-current bus is the most common root cause.
4. If UAS still resets, force Bulk-Only Transport inside the VM and on the host:
   `options usb-storage quirks=0bda:9201:u`.

The existing `usbcore.quirks=0bda:9201:k` cmdline entry only disables Link
Power Management (`k`); it does not address UAS or autosuspend.

#### `usb usbN-portN: Cannot enable. Maybe the USB cable is bad?`

This loop is a **USB link-training/power failure**, not autosuspend and not a
ZFS naming problem. The bridge never enumerates, so no `/dev/disk/by-id` path
exists and `zpool` cannot open or online the member. Typical causes, in order:

1. Cable or connector (the kernel says it explicitly; swap the cable first).
2. Port or hub: try a different root port, avoid unpowered hubs, and check
   whether the device negotiated 5000M or fell back to 480M with `lsusb -t`.
3. Enclosure power: two 2.5" HDDs on bus power can brown out at spin-up; use a
   powered hub or an enclosure with its own supply.
4. Only after the disk appears in `lsblk`/`/dev/disk/by-id`, bring the vdev
   back with `zpool online <pool> <vdev-or-guid>`. If the label is invalid,
   `zpool replace` is required; never recreate the pool.

On Aleph (2026-11-03) the HGST member failed to enable for ~9 minutes after
boot, then enumerated at 480M, came back ONLINE and resilvered 129M
automatically. The pool was never misnamed: `wwn-*`, `ata-*` and `usb-*`
aliases all point at the same two disks. The role now also tries
`zpool online` for UNAVAIL members and fails the play if the pool is still not
ONLINE, so a lost mirror cannot pass silently.
