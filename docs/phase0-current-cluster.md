# Phase 0 — Current Cluster Capture

**Linear:** [PET-14 / BOOT-01](https://linear.app/petatron/issue/PET-14/boot-01-capture-the-current-manual-cluster-bootstrap-as-a-reproducible)
**Captured:** 2026-08-20, from `ssh 100.116.30.60` (live cluster inspection)
**Re-checked:** 2026-11-06 for [PET-32 / NET-01](https://linear.app/petatron/issue/PET-32/net-01-harden-control-plane-network-and-api-endpoint-assumptions), read-only — see [§9](#9-network-re-check--2026-11-06-pet-32)
**Status:** both hosts inspected. Control-plane facts verified 2026-08-20/21; workstation facts
verified 2026-08-21 via a privileged Kubernetes debug pod. Original `kubeadm init` **recovered**.

> **Several earlier entries in this document were wrong and have been corrected in place.** See
> [Corrections](#corrections) for what changed and why it matters.

This document records the cluster as it actually exists, not as the repos describe it. Where the
two disagree, reality is recorded and the drift is listed in [Drift](#drift-repos-vs-reality).

---

## 1. Topology

| | Control plane | Worker |
|---|---|---|
| K8s node name | `hlcluster-ctrlr0` | `hycluster-worker-0` |
| Hardware | Dell XPS 13 | Custom workstation |
| CPU | 8 vCPU | **AMD Ryzen Threadripper 7960X**, 24C / 48T |
| Memory | 15 GiB | 125 GiB |
| Node IP | `192.168.16.10` (DHCP, USB NIC) | `192.168.15.13` (DHCP, on `br0`) |
| Tailscale | `100.116.30.60` | `100.100.51.95` (as `hycluster-worker-0-1`) |
| Ephemeral storage | 468 GiB | 915 GiB |
| Roles | `control-plane` | *(none)* |
| Taints | `node-role.kubernetes.io/control-plane:NoSchedule` | none |
| `/dev/kvm` | **absent** | **present** |
| Virtualization | none exposed | **AMD-V**, `kvm_amd` loaded |
| IOMMU | not checked | **enabled**, 51 groups |
| GPUs | none | **3 × NVIDIA `10de:2d04`** |
| libvirt | not installed | **already installed** (12 pkgs, `virsh`, `qemu-system-x86_64`) |

Both nodes: Ubuntu 24.04.4 LTS, kernel 6.17.0-20-generic, containerd 2.2.1, kubelet v1.35.3.
Cluster age at capture: 128 days. Single-node etcd, single control plane (no HA).

### Naming inconsistency

Three spellings are in use for the same machine and its pair. Recorded as-is; do not "fix"
silently, since the k8s node name is baked into certificates and kubelet registration:

- K8s control-plane node / hostname: `hlcluster-ctrlr0`  (`hl`, no dash before `0`)
- Tailscale device name: `hycluster-ctrlr-0`  (`hy`, dash before `0`)
- K8s worker node: `hycluster-worker-0`  (`hy`)

### Network path between nodes

The two machines are **not on the same L2 segment or the same L3 network**:

```
192.168.16.10 (control plane, enx9c69d34cc8a1, DHCP)
      |  1.9 ms
192.168.16.1 (local gateway)
      |  32.5 ms          <-- ~30 ms RTT is incurred on this hop
192.168.0.1 (upstream router)
      |
192.168.15.13 (worker)
```

- Measured RTT control plane → worker: **~30 ms** (min 29.2 / avg 30.6 / max 31.7 ms).
- Path MTU: **1420**.
- The control plane has no route to `192.168.15.0/24`; traffic exits via the default gateway.
  *(Misleading — corrected 2026-11-06, [Corrections](#corrections) #7: the default gateway **is** the
  route. The two sites are joined by UniFi SD-WAN, and `192.168.0.1` is its tunnel peer.)*
- Control-plane uplink is a **USB Ethernet dongle** (`enx9c69d34cc8a1`, MTU 1500). Onboard Wi-Fi
  (`wlp2s0`) is DOWN.
- The control-plane node address is **DHCP-assigned** (`proto dhcp`), and that address is the
  API server's advertise address and appears in every kubeconfig and kubelet bootstrap config.

Consequences to carry into later phases: all worker kubelet → API server traffic, all pod-to-pod
VXLAN traffic, and (see below) all cluster DNS traffic cross the ~30 ms hop.

---

## 2. kubeadm cluster configuration

Source: `kubectl -n kube-system get cm kubeadm-config -o jsonpath='{.data.ClusterConfiguration}'`

```yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
kubernetesVersion: v1.35.3
clusterName: kubernetes
certificatesDir: /etc/kubernetes/pki
caCertificateValidityPeriod: 87600h0m0s   # 10 years
certificateValidityPeriod: 8760h0m0s      # 1 year
encryptionAlgorithm: RSA-2048
etcd:
  local:
    dataDir: /var/lib/etcd
networking:
  dnsDomain: cluster.local
  podSubnet: 10.244.0.0/16                # DECLARED but NOT in use - see Drift
  serviceSubnet: 10.96.0.0/12
apiServer: {}
controllerManager: {}
scheduler: {}
dns: {}
proxy: {}
```

Effective API server flags:

```
--advertise-address=192.168.16.10
--secure-port=6443
--etcd-servers=https://127.0.0.1:2379
--service-cluster-ip-range=10.96.0.0/12
```

Admin kubeconfig server: `https://192.168.16.10:6443`

### kubelet configuration (kube-system/kubelet-config)

```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd
clusterDNS: [10.96.0.10]
clusterDomain: cluster.local
authentication:
  anonymous: {enabled: false}
  webhook: {enabled: true}
  x509: {clientCAFile: /etc/kubernetes/pki/ca.crt}
authorization: {mode: Webhook}
healthzBindAddress: 127.0.0.1
healthzPort: 10248
```

### Certificate validity — measured 2026-08-21

From `sudo kubeadm certs check-expiration`:

| Certificate | Expires | Residual | CA |
|---|---|---|---|
| `admin.conf`, `apiserver`, `apiserver-kubelet-client`, `controller-manager.conf`, `scheduler.conf`, `super-admin.conf` | 2027-04-14 03:44 UTC | 235d | `ca` |
| `apiserver-etcd-client`, `etcd-healthcheck-client`, `etcd-peer`, `etcd-server` | 2027-04-14 03:44 UTC | 235d | `etcd-ca` |
| `front-proxy-client` | 2027-04-14 03:44 UTC | 235d | `front-proxy-ca` |

Certificate authorities: `ca`, `etcd-ca`, `front-proxy-ca` all expire **2036-04-11 03:18 UTC** (9 years).

None are externally managed. All leaf certificates share one expiry date, so renewal is a single
event rather than a staggered series.

### API server certificate SANs

```
DNS:hlcluster-ctrlr0, DNS:kubernetes, DNS:kubernetes.default,
DNS:kubernetes.default.svc, DNS:kubernetes.default.svc.cluster.local,
IP Address:10.96.0.1, IP Address:192.168.16.10
```

This confirms the DHCP-assigned address is embedded in the serving certificate. There is **no
stable DNS name** for the endpoint — `hlcluster-ctrlr0` is the hostname, not a resolvable record.
Re-addressing the control plane therefore requires regenerating this certificate.

### Static pod manifests

`/etc/kubernetes/manifests/` (all written 2026-04-13 23:44, mode 0600 root:root):

| File | Size |
|---|---|
| `etcd.yaml` | 2618 |
| `kube-apiserver.yaml` | 3959 |
| `kube-controller-manager.yaml` | 3458 |
| `kube-scheduler.yaml` | 1726 |
| `.kubelet-keep` | 0 (2026-03-18) |

---

## 3. Host prerequisites (from `~/bootstrap-control-plane.sh`)

The control plane was prepared by `~/bootstrap-control-plane.sh` on the XPS 13. Steps it performs,
which any reproduction must also perform:

1. `swapoff -a` and strip swap entries from `/etc/fstab`.
2. Load kernel modules `overlay`, `br_netfilter`; persist via `/etc/modules-load.d/k8s.conf`.
3. Sysctl via `/etc/sysctl.d/k8s.conf`:
   `net.bridge.bridge-nf-call-iptables=1`, `net.bridge.bridge-nf-call-ip6tables=1`,
   `net.ipv4.ip_forward=1`.
4. Install `containerd`, write `containerd config default` to `/etc/containerd/config.toml`,
   set `SystemdCgroup = true`, enable + restart.
5. Add the `pkgs.k8s.io` apt repo, install `kubelet kubeadm kubectl`, `apt-mark hold` all three,
   enable kubelet.

The script **stops before `kubeadm init`** — it only prints suggested next steps.

### Recovered bootstrap sequence

The actual invocation is **not** in the script, but it was recovered on 2026-08-21 from two
independent sources that agree:

- `/root/.bash_history`
- the systemd journal: `Apr 13 23:18:52 hlcluster-ctrlr0 sudo[17529]: yibofu : ... COMMAND=/usr/bin/kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=192.168.16.10`

The sequence, in order:

```bash
sudo kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --apiserver-advertise-address=192.168.16.10

chmod +x /home/yibofu/install-cilium.sh
sudo bash ~/install-cilium.sh          # -> `cilium install` (CLI, not Helm)

kubeadm token create --print-join-command
```

### Worker join — recovered, and it points at a different control-plane address

The workstation's shell history contains the actual join, repeated across several retries:

```bash
sudo swapoff -a
sudo apt-get install -y containerd
sudo systemctl enable --now containerd
sudo modprobe br_netfilter
sudo kubeadm join 192.168.15.10:6443 --token <redacted> \
  --discovery-token-ca-cert-hash sha256:<redacted>
```

(The token is long expired — kubeadm tokens default to 24h — and the CA hash is a public key
fingerprint, so neither is sensitive. Both are nonetheless left out of this document.)

**The endpoint is `192.168.15.10`, not `192.168.16.10`.** That is the workstation's own `/24`.

The current cluster's API server advertises `192.168.16.10`, the recovered `kubeadm init` used
`192.168.16.10`, and the serving certificate's SANs contain `192.168.16.10` — so the *running*
cluster is consistent. But the join history, together with
`hydra-infra/terraform.tfvars.example`'s `control_plane_ip = "192.168.15.10"`, is evidence that at
some point the control plane sat at `192.168.15.10`, on the **same subnet as the workstation**.

**Resolved 2026-08-21.** The workstation's live kubelet configuration reads:

```
/etc/kubernetes/kubelet.conf ->  server: https://192.168.16.10:6443
```

`kubelet.conf` is written at join time and certificate rotation does not rewrite its server URL, so
the **current** cluster was joined against `192.168.16.10`. The `192.168.15.10` entries in shell
history therefore belong to an **earlier, discarded cluster generation** in which the control plane
sat on the workstation's own subnet.

Consequences:

- The ~30 ms inter-site hop was present from the **beginning of this cluster**. It is not a
  regression that crept in — it was baked in at build time.
- A previous cluster generation did have the control plane on `192.168.15.0/24`. So the topology
  being proposed for the rebuild (control plane on the workstation) has precedent here.
- `hydra-infra/terraform.tfvars.example` describes that **earlier** generation. It is genuinely
  stale with respect to the running cluster, though its values were real rather than invented.

**This explains the pod-subnet mismatch.** `kubeadm init` was correctly given
`--pod-network-cidr=10.244.0.0/16`, but `cilium install` does not read kubeadm's `podSubnet` — it
defaults to its own `cluster-pool` of `10.0.0.0/8`. The divergence was an **oversight, not a
deliberate choice**: nothing ever told Cilium about the intended range.

The script's printed guidance does not match the cluster that was built (see Drift): it pins
`KUBE_VERSION="v1.32"`, suggests `--pod-network-cidr=10.244.0.0/16`, and suggests Flannel.

---

## 4. CNI — Cilium

Installed with the **`cilium` CLI** (`~/install-cilium.sh` → `cilium install`), *not* Helm and
*not* GitOps. This is the handoff gap for Phase 7: nothing declaratively owns Cilium today.

| Setting | Value |
|---|---|
| Version | 1.19.1 (chart 1.19.1) |
| IPAM mode | `cluster-pool` |
| `cluster-pool-ipv4-cidr` | **`10.0.0.0/8`** |
| `cluster-pool-ipv4-mask-size` | 24 |
| Routing mode | `tunnel` |
| Tunnel protocol | `vxlan` |
| `kube-proxy-replacement` | **`false`** (kube-proxy also runs) |
| IPv4 masquerade | enabled |
| L7 proxy | enabled (`cilium-envoy` DaemonSet, 2/2) |
| Hubble relay | disabled |
| ClusterMesh | disabled |
| WireGuard / IPsec | **not enabled** |

Allocated per-node pod CIDRs: control plane `10.0.0.0/24` (`cilium_host` `10.0.0.213`),
worker `10.0.1.0/24`.

Pod-network MTU is **1230**, auto-detected. Path MTU to the worker is 1420, so pod MTU is
conservative by ~140 bytes. 1230 = 1280 − 50 (VXLAN), i.e. detection appears to have keyed off
the `tailscale0` interface (MTU 1280) rather than the 1500-MTU egress NIC.

**Pod traffic between the two hosts is unencrypted VXLAN across a routed ~30 ms path.**

*Re-checked 2026-11-06: all of this still holds. The pod CIDR divergence and the MTU are now
accepted on this cluster rather than fixed — see [§9](#9-network-re-check--2026-11-06-pet-32).*

---

## 5. Deployed platform add-ons

**None.** Namespaces present: `cilium-secrets`, `default`, `kube-node-lease`, `kube-public`,
`kube-system`.

Verified absent: Argo CD, MetalLB, metrics-server, cert-manager, any StorageClass, any
PersistentVolume, any GPU device plugin. No `helm`, `argocd`, or `clusterctl` binary on the
control plane.

All 14 running pods are `kube-system` (Cilium ×3 kinds, CoreDNS ×2, kube-proxy ×2, and the four
static control-plane pods).

### CoreDNS placement

Both CoreDNS replicas run on the **control plane** (`10.0.0.181`, `10.0.0.140`). Every DNS query
from a workload pod — and all workloads run on the worker — therefore crosses the ~30 ms link and
back. Worth fixing independently of Hydra; it is a standing latency tax on every workload.

*No longer true — corrected 2026-11-06, [Corrections](#corrections) #8: the workloads moved onto
the control plane, so the placement now matches the load.*

---

## 6. Virtualization state

- **Control plane (XPS 13): cannot host VMs.** `/dev/kvm` does not exist and `virsh` is not
  installed. `~/vms/` does not exist.
- **Workstation: already a working libvirt host.** This corrects the earlier "unverified" entry.

### Workstation virtualization — verified 2026-08-21

| Check | Result |
|---|---|
| CPU | AMD Ryzen Threadripper 7960X, 24 cores / 48 threads, 1 socket |
| Virtualization | `AMD-V`; `vmx\|svm` flag on all 48 threads |
| `/dev/kvm` | `crw-rw----+ 1 root kvm 10, 232` — present |
| Kernel modules | `kvm_amd` (241664) and `kvm` (1445888) loaded |
| IOMMU | enabled — `iommu: Default domain type: Translated`, AMD-Vi counters, **51 groups** |
| libvirt/qemu | **installed** — 12 matching packages, `/usr/bin/virsh`, `/usr/bin/qemu-system-x86_64` |
| `virbr0` | exists at `192.168.122.1/24`, currently DOWN (default libvirt network, unused) |
| `br0` | **exists and UP**, `192.168.15.13/24` — this is the node's own address |

**No firmware work is required.** Virtualization and IOMMU are already on, KVM is loaded, and the
hypervisor stack is already installed. The `br0` bridge that `hydra-infra` expects already exists.

### GPUs — three, not one

Three identical NVIDIA cards (`10de:2d04`, consistent with RTX 5060 Ti), on separate root ports:

| PCI address | Subsystem | Driver in use |
|---|---|---|
| `01:00.0` | Gigabyte `1458:418f` | `nvidia` |
| `21:00.0` | ZOTAC `19da:1772` | `nvidia` |
| `c1:00.0` | Gigabyte `1458:418f` | `nvidia` |

Each has a companion HDMI audio function (`10de:22eb`) bound to `snd_hda_intel`. All three GPUs are
currently held by the host `nvidia` driver, and Kubernetes shows no `nvidia.com/gpu` resource
because no device plugin is installed.

**This invalidates the single-GPU premise of ADR-004** — see [Corrections](#corrections).

### Network configuration

```
enp141s0   DOWN                      (spare 1/10GbE)
enp142s0   UP     (enslaved to br0)
wlp143s0   DOWN                      (spare Wi-Fi)
br0        UP     192.168.15.13/24   default via 192.168.15.1, proto dhcp
virbr0     DOWN   192.168.122.1/24
tailscale0 UNKNOWN 100.100.51.95/32
```

Netplan has **two files that conflict**:

- `/etc/netplan/01-bridge.yaml` — `enp142s0: {dhcp4: no}`, `br0: {interfaces: [enp142s0], dhcp4: yes}`
- `/etc/netplan/50-cloud-init.yaml` — `enp142s0: {dhcp4: true}`

Netplan merges by ascending filename, so `50-cloud-init.yaml` wins and re-enables DHCP directly on
the bridge's slave interface. That is a latent fault: `enp142s0` may acquire its own lease
alongside `br0`. The cloud-init file should be neutralised rather than left to race.

> **Fixed 2026-11-14 (PET-34)** — see [Corrections](#corrections) #11. Confirmed first: `netplan get`
> merged to `enp142s0: dhcp4: true`, and the generated `10-netplan-enp142s0.network` carried both
> `DHCP=ipv4` and `Bridge=br0`, so networkd sat at `enslaved (configuring)`. `enp142s0` never held a
> lease of its own. Now `50-cloud-init.yaml` is renamed `50-cloud-init.yaml.disabled-pet34`,
> cloud-init's network management is off (`/etc/cloud/cloud.cfg.d/99-disable-network-config.cfg`),
> `01-bridge.yaml` is `0600`, and `enp142s0` is `enslaved (configured)` with no address.

Two spare interfaces are available (`enp141s0`, `wlp143s0`), which matters for giving a relocated
control plane a dedicated static address without disturbing `br0`.

### Firewall — not the cause of the SSH block

`ufw` is **inactive**; `iptables -L INPUT` policy is **ACCEPT**, with only the expected
`CILIUM_INPUT`, `ts-input`, and `KUBE-*` chains. Nothing on the host blocks TCP 22.

Combined with the earlier probes — `tailscale ping` succeeds (35 ms, direct), TCP 22 over the
tailnet times out silently, TCP 22 over the LAN is refused — the SSH block is in the **tailnet
access policy**, not on the host. The host has no `sshd` at all (consistent with the control
plane), so the LAN refusal is expected; the tailnet-side silent drop is an ACL denial.

The device was previously registered as `hycluster-worker-0` and had fallen out of the tailnet; it
re-registered as `hycluster-worker-0-1`, so any ACL rule naming the old hostname no longer matches.

### Implication for ADR-002

[ADR-002](https://app.notion.com/p/3c2ee32e02e58115acbbdbe17d53ef7c) guards against split-brain
worker lifecycle between the `hydra-infra` Terraform/`sync-nodes.sh` path and CAPI. As captured,
**there are zero Terraform-managed workers** — `hydra-infra` has never been applied against this
cluster. The single worker was joined by hand.

The single-writer invariant is still the right rule, but the migration it guards is currently
*empty*: there is nothing to adopt or recreate. This materially de-risks PET-29 and supports
moving it off the critical path ahead of PET-9.

---

## 7. Drift: repos vs reality

| # | Repo / artifact says | Cluster actually is |
|---|---|---|
| 1 | `kubeadm-config` `podSubnet: 10.244.0.0/16` | Cilium `cluster-pool` **`10.0.0.0/8`**; the declared podSubnet is unused |
| 2 | `bootstrap-control-plane.sh`: `KUBE_VERSION="v1.32"` | v1.35.3 |
| 3 | `bootstrap-control-plane.sh`: suggests **Flannel** | **Cilium** 1.19.1 |
| 4 | `hydra-infra` README: Terraform + `sync-nodes.sh` manage worker VMs | No VMs; the one worker is bare metal, joined manually |
| 5 | `hydra-infra` README: "see `~/vms/setup-host.sh`" | Path does not exist on the control plane |
| 6 | `hydra-infra` `terraform.tfvars.example`: `control_plane_ip = "192.168.15.10"` | Control plane is `192.168.16.10`. The value describes an **earlier cluster generation** (confirmed: shell history joins to `.15.10`, but the live `kubelet.conf` points at `.16.10`). Stale with respect to the running cluster, but not invented |
| 7 | `hydra-infra` `versions.tf`: `provider libvirt { uri = "qemu:///system" }` | Implies Terraform runs *on* the libvirt host; no libvirt host is configured |
| 8 | `hydra-infra` expects bridge `br0`, image at `/home/yibofu/vms/images/...` | **Revised:** `br0` **does exist** on the workstation (`192.168.15.13/24`, UP). Only the control plane lacks it — the tfvars was written against the workstation, where it is correct |
| 9 | `hydra-gitops`: 10 manifests (Argo CD, Cilium, KEDA, MetalLB, local-path) | **Nothing deployed**; no `argocd` namespace |
| 10 | `hydra-gitops` app-of-apps owns Cilium | Cilium installed imperatively via `cilium install` CLI |

Items 6–8 together indicate `terraform.tfvars.example` was written against an **earlier cluster
generation** whose control plane sat in `192.168.15.0/24` alongside the workstation, and which was
intended to run **on the workstation**. That generation demonstrably existed. The file is stale with
respect to the running cluster, but `br0` and the general shape were accurate for the host it
targets.

---

## 8. Reproduction gaps

Facts a reproduction needs that are **not** currently recoverable from any source-controlled
artifact or script:

**Closed 2026-08-21:**

1. ~~The exact `kubeadm init` command and flags.~~ **Recovered** — see above, corroborated by two
   independent sources.
2. ~~Actual certificate expiry dates.~~ **Measured** — see above.
3. ~~Why `podSubnet` diverged from the Cilium pool.~~ **Answered** — `cilium install` ignores
   kubeadm's `podSubnet`; it was an oversight.

**Still open:**

4. The literal `kubeadm join` command. The *generating* command is known
   (`kubeadm token create --print-join-command`) and the form is standard, so this is closed for
   reproduction purposes. The host prep actually performed on the workstation is **not** recorded —
   `bootstrap-control-plane.sh` is only known to have run on the XPS 13.
5. Whether the control-plane DHCP lease for `192.168.16.10` is reserved/static-mapped.
6. Workstation host state: KVM/libvirt presence, IOMMU status, GPU driver binding, why sshd
   refuses connections.

PET-14's "no critical step depends on undocumented shell history" criterion is now **substantially
met** for the control plane: the bootstrap is reproducible from recorded commands rather than
reconstructed from observed state. It is **not** met for the worker's host preparation.

---

## 9. Network re-check — 2026-11-06 (PET-32)

**State verified:** 2026-11-06, read-only, `kubectl` and `openssl` on `hlcluster-ctrlr0`
(`192.168.16.10`, v1.35.3) and `hydra-wl0` through the SOCKS tunnel. Nothing was changed.

PET-74 plans to replace this cluster with a management cluster Cluster API builds, so PET-32 was
re-scoped (Yibo, 2026-11-06): **mitigate and document here, and carry the real fixes into PET-74.**

| # | Finding (2026-08) | State 2026-11-06 | Decision |
|---|---|---|---|
| 1 | API endpoint is a DHCP address on a USB NIC | **Still true.** `192.168.16.10/27 proto dhcp` on `enx9c69d34cc8a1`. Serving cert SANs are that IP plus the default names — no DNS name, no VIP; no `controlPlaneEndpoint` in `kubeadm-config` | **Mitigate:** the home UniFi already gives this client a fixed IP (confirmed by the UniFi admin, 2026-11-06), so the lease cannot move while the same adapter is used; and [`control-plane-address-change.md`](./control-plane-address-change.md) for when it changes anyway. A VIP or DNS endpoint is a PET-74 criterion. |
| 2 | Hosts ~30 ms apart, no route | Routed via SD-WAN ([Corrections](#corrections) #7). Every CAPI controller runs on this node, ~30 ms from every VM and API it manages | Where management runs is **PET-74's ADR**. |
| 3 | Pod traffic unencrypted | Still no WireGuard / IPsec in `cilium-config`. The inter-site hop is UniFi **Site Magic SD-WAN, which uses WireGuard** peer-to-peer tunnels between the sites, with no user-configurable encryption settings (confirmed by the UniFi admin, 2026-11-06) | **Accept on this cluster:** pod traffic is encrypted wherever it leaves a site, and plaintext only on each site's own LAN. The decision for the new cluster is a PET-74 criterion. |
| 4 | CoreDNS on the control plane | Placement now matches the load ([Corrections](#corrections) #8) | **No change.** |
| 5 | Pod MTU 1230 vs path 1420 | Still: `cilium_host` 1280, remote pod routes 1230. The same 1230 appears on the workstation for `hydra-wl0`, which also runs Tailscale | **Accept**: conservative, not broken (throughput only). Restarting Cilium on the management cluster to fix it is not worth it. Set explicitly in PET-74. |
| 6 | kubeadm `10.244.0.0/16` vs Cilium `10.0.0.0/8` | Unchanged here. **`hydra-wl0` agrees** (both `10.244.0.0/16`), because the KCP example installs Cilium with the pod CIDR | **Accept as deliberate on this cluster** (re-IPing every pod of a cluster PET-74 retires is not worth it). Clusters Cluster API builds do not inherit it. |

For comparison, `hydra-wl0`: endpoint is the kube-vip VIP `192.168.15.10:6443`, kubeadm and Cilium
agree on `10.244.0.0/16`, CoreDNS runs on two different nodes.

---

## Corrections

Entries in earlier revisions of this document that turned out to be wrong, and what replaced them.

| # | Earlier claim | Corrected finding | Why it matters |
|---|---|---|---|
| 1 | The workstation's virtualization state is unverified; PET-31 must verify or enable KVM and install libvirt | **Already done.** AMD-V on, IOMMU on, `kvm_amd` loaded, `/dev/kvm` present, libvirt and qemu installed, `br0` up | PET-31's scope shrinks to resource partitioning, a storage pool, base image, and the control-path decision. No firmware or install work. |
| 2 | A **single** NVIDIA 5060 Ti, so GPU passthrough is all-or-nothing and a GPU pool caps at `max: 1` | **Three** identical NVIDIA `10de:2d04` cards on separate root ports (`01:00.0`, `21:00.0`, `c1:00.0`) | Invalidates the core premise of ADR-004 and PET-33. See below. |
| 3 | The original `kubeadm init` and `join` commands are lost to shell history | Both **recovered** — `init` from the control plane's root history and the systemd journal, `join` from the workstation's history | PET-14's reproducibility criterion is met from records, not reconstruction. |
| 4 | Certificate expiry ≈ 237 days (estimated) | **2027-04-14**, 235 days; CAs 2036-04-11 (measured) | Renewal is one event, all leaf certs share a date. |
| 5 | `hydra-infra`'s `control_plane_ip` and `br0` assumptions are drift/wrong | Both were likely **correct when written**; `br0` exists on the workstation and the join history references `192.168.15.10` | The repo is stale relative to a topology *change*, not built on false assumptions. |
| 6 | The workstation "refuses SSH", implying a misconfiguration | Neither host runs `sshd`; access is Tailscale SSH. The block is a **tailnet ACL**, confirmed by `ufw` inactive and `iptables` INPUT ACCEPT | Fix is in the Tailscale admin console, not on the host. |
| 7 | The control plane has no route to `192.168.15.0/24`; `192.168.0.1` is an "upstream router" | The default route reaches the office through **UniFi SD-WAN**; `192.168.0.1` is its tunnel peer (`tracepath` 2026-11-06: path MTU 1420, RTT ≈ 31 ms). SD-WAN carries only networks UniFi defines at each site — a static route at one site is not advertised to the other (PET-75) | Cross-site reachability is set in UniFi, not on the hosts. |
| 8 | Every workload DNS query crosses the inter-site hop because both CoreDNS replicas sit on the control plane | Since the cluster became the CAPI management cluster, **34 of 41 pods run on `hlcluster-ctrlr0`** (every management controller is pinned there); the two worker VMs carry 7 system pods. CoreDNS on the control plane is next to its clients | PET-32 item 4 needs no change. |
| 9 | (implicit) the cluster's workers are the bare-metal workstation | The workers are now two **Hydra-managed VMs** (`hydra-md-0-mj9gx-4c6dv` `192.168.15.52`, `hydra-worker-0` `192.168.15.200`); the workstation is a node of `hydra-wl0` instead | Anything in §1 about "the worker" describes the August state. |
| 10 | The workstation's host prep is a subset of `bootstrap-control-plane.sh`: no persisted modules or sysctls, no `SystemdCgroup`, no package hold ([Open items](#open-items)) | Re-audited 2026-11-14 (PET-34). Module and sysctl persistence existed **since 2026-02-04**, under names the capture did not look for: `/etc/modules-load.d/k8s.conf` (`br_netfilter`), and `/etc/sysctl.d/99-kubernetes.conf` plus `99-kubernetes-cri.conf` (bridge-nf-call-iptables/ip6tables, `ip_forward`). `SystemdCgroup = true` is set in `/etc/containerd/config.toml`, matching kubelet's `cgroupDriver: systemd`; kubeadm, kubelet and kubectl are held at 1.35.3; swap is off. All of it survived the 2026-11-05 boot. The one real gap was `overlay` missing from `k8s.conf`, added 2026-11-14 | Only `overlay`'s load at boot is unproven, until the host's next reboot. |
| 11 | The netplan conflict is latent | Confirmed and fixed 2026-11-14 (PET-34); see [Network configuration](#network-configuration). Applied with `networkctl reload` and `networkctl reconfigure enp142s0`: `netplan try` refuses whenever a bridge has custom parameters, and reconfiguring only the port left `br0` and its address alone. A 1 s reachability monitor of the host, both VIPs and a hydra-wl0 control-plane node lost **0** packets, and all 8 VMs and both clusters were unchanged | Backups of the previous files are in `/root/pet34-20261114/` on the host. |

### ADR-004 must be rewritten

The three-GPU finding changes the available options qualitatively, not just quantitatively:

- **Host and VMs can coexist.** One or two cards can be bound to `vfio-pci` for guests while the
  remaining card stays on the host `nvidia` driver. The earlier "the host loses the GPU entirely"
  trade-off does not apply.
- **A GPU node pool can genuinely scale.** `min: 0, max: 2` (keeping one card on the host) or
  `max: 3` is a real autoscaling demonstration, not a one-node token.
- **Still out of reach:** "multiple GPU classes" (all three cards are the same model), vGPU
  (consumer silicon), and MIG (datacenter-only). Those roadmap items still need different hardware.
- **Prerequisite satisfied.** Per-device IOMMU isolation verified 2026-08-21 — each GPU sits alone
  in its own group with only its companion HDMI audio function:

  | IOMMU group | Devices |
  |---|---|
  | 15 | `c1:00.0` GPU + `c1:00.1` audio |
  | 26 | `01:00.0` GPU + `01:00.1` audio |
  | 38 | `21:00.0` GPU + `21:00.1` audio |

  No bridges or unrelated devices share these groups. This is textbook-clean for VFIO: each
  GPU/audio pair can be passed to a guest independently, without pulling other hardware with it.
  **All three cards are independently passable.**

## Open items

Blocked on access to the workstation (`192.168.15.13`), which refuses TCP 22:

- [x] ~~libvirt / KVM present? IOMMU enabled?~~ — **all present and enabled**
- [x] ~~NVIDIA driver binding~~ — all three cards on the host `nvidia` driver
- [x] ~~Host prep applied on the workstation~~ — **recovered** from its shell history: `swapoff -a`,
      `apt-get install containerd`, `systemctl enable --now containerd`, `modprobe br_netfilter`,
      then `kubeadm join`. Note this is a **subset** of `bootstrap-control-plane.sh` — no
      persisted `/etc/modules-load.d/k8s.conf`, no `/etc/sysctl.d/k8s.conf`, no
      `SystemdCgroup = true` edit, no `apt-mark hold`. The worker's prep is less complete than the
      control plane's and should not be assumed identical. **Corrected 2026-11-14**: the persistence
      existed under other file names; see [Corrections](#corrections) #10.
- [x] ~~Per-device IOMMU group isolation~~ — **verified clean**: groups 15 / 26 / 38, one GPU each
- [x] ~~Which API endpoint the workstation's kubelet uses~~ — `https://192.168.16.10:6443`; the
      `.15.10` history belongs to an earlier cluster generation
- [x] ~~Neutralise the `50-cloud-init.yaml` / `01-bridge.yaml` netplan conflict~~ — **done 2026-11-14**
      (PET-34); see [Corrections](#corrections) #11
- [ ] Choose a static address (or DNS name) for the relocated control-plane API endpoint
- [ ] Decide the pod/VM resource split on the workstation
- [x] ~~Reason sshd is closed, and the intended admin path~~ — **answered**: no machine runs sshd;
      access is via Tailscale SSH. The workstation simply is not on the tailnet.
- [ ] Get the workstation onto the tailnet (`tailscale up --ssh`) so it is reachable at all to this host

~~Blocked on sudo on the control plane~~ — **resolved 2026-08-21**, passwordless sudo granted via
`/etc/sudoers.d/yibofu-nopasswd`. Certificate expiry, static pod manifest inventory, certificate
SANs, and the original `kubeadm init` command have all been captured above.
