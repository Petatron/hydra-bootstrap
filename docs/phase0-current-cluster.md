# Phase 0 — Current Cluster Capture

**Linear:** [PET-14 / BOOT-01](https://linear.app/petatron/issue/PET-14/boot-01-capture-the-current-manual-cluster-bootstrap-as-a-reproducible)
**Captured:** 2026-08-20, from `ssh 100.116.30.60` (live cluster inspection)
**Status:** control-plane/cluster facts verified. Hypervisor-host facts **pending host access** (see [Open items](#open-items)).

This document records the cluster as it actually exists, not as the repos describe it. Where the
two disagree, reality is recorded and the drift is listed in [Drift](#drift-repos-vs-reality).

---

## 1. Topology

| | Control plane | Worker |
|---|---|---|
| K8s node name | `hlcluster-ctrlr0` | `hycluster-worker-0` |
| Hardware | Dell XPS 13 | Workstation |
| Node IP | `192.168.16.10` | `192.168.15.13` |
| CPU / Memory | 8 vCPU / 15 GiB | 48 vCPU / 125 GiB |
| Ephemeral storage | 468 GiB | 915 GiB |
| Roles | `control-plane` | *(none)* |
| Taints | `node-role.kubernetes.io/control-plane:NoSchedule` | none |
| `/dev/kvm` | **absent** | unverified |

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

### Certificate validity

`certificateValidityPeriod: 8760h` (1 year) with the cluster 128 days old means leaf certificates
expire roughly **237 days from capture**. Exact expiry was **not** read — `kubeadm certs
check-expiration` requires sudo and the session is password-gated. Record the real dates before
closing PET-14; this is a recovery-critical fact and it is currently undocumented.

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

The script **stops before `kubeadm init`** — it only prints suggested next steps. The actual
`kubeadm init` invocation used for this cluster is **not recorded anywhere** and is not
recoverable from the script. See [Open items](#open-items).

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

---

## 6. Virtualization state

- **Control plane (XPS 13): cannot host VMs.** `/dev/kvm` does not exist and `virsh` is not
  installed. `~/vms/` (referenced by the `hydra-infra` README) does not exist.
- **Worker (workstation): unverified.** TCP 22 is **refused** from the control plane
  (`ssh: connect to host 192.168.15.13 port 22: Connection refused`), and the machine is not a
  Tailscale peer. There is no path from this session to inspect it.
- **No VMs exist in this cluster.** The worker is a **bare-metal** node, joined directly. No
  libvirt domain backs any Kubernetes node.

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
| 6 | `hydra-infra` `terraform.tfvars.example`: `control_plane_ip = "192.168.15.10"` | Control plane is `192.168.16.10`, in a **different** `/27` |
| 7 | `hydra-infra` `versions.tf`: `provider libvirt { uri = "qemu:///system" }` | Implies Terraform runs *on* the libvirt host; no libvirt host is configured |
| 8 | `hydra-infra` expects bridge `br0`, image at `/home/yibofu/vms/images/...` | Neither present on the control plane |
| 9 | `hydra-gitops`: 10 manifests (Argo CD, Cilium, KEDA, MetalLB, local-path) | **Nothing deployed**; no `argocd` namespace |
| 10 | `hydra-gitops` app-of-apps owns Cilium | Cilium installed imperatively via `cilium install` CLI |

Items 6–8 together indicate `terraform.tfvars.example` was written for a topology in which the
control plane sits in `192.168.15.0/24` alongside the workstation. That is not the current layout.

---

## 8. Reproduction gaps

Facts a reproduction needs that are **not** currently recoverable from any source-controlled
artifact or script:

1. The exact `kubeadm init` command and flags used.
2. The exact `kubeadm join` command used for `hycluster-worker-0`, and the host prep performed on
   the workstation (the bootstrap script is only known to have been run on the XPS 13).
3. Actual certificate expiry dates (`kubeadm certs check-expiration`).
4. Why `podSubnet` was declared as `10.244.0.0/16` while Cilium was left on its default pool —
   deliberate or overlooked.
5. Whether the control-plane DHCP lease for `192.168.16.10` is reserved/static-mapped. If not,
   a lease change invalidates the API endpoint in every kubeconfig and kubelet config.
6. Workstation host state: KVM/libvirt presence, IOMMU status, GPU driver binding, why sshd
   refuses connections.

Items 1–2 are the core of PET-14's "no critical step depends on undocumented shell history"
criterion and cannot be closed by inspection alone.

---

## Open items

Blocked on access to the workstation (`192.168.15.13`), which refuses TCP 22:

- [ ] libvirt / KVM present? IOMMU enabled? (`/dev/kvm`, `virsh version`, `dmesg | grep -i iommu`)
- [ ] NVIDIA 5060 Ti driver binding — host `nvidia` driver vs `vfio-pci`
- [ ] Host prep actually applied on the workstation vs the XPS 13
- [ ] Reason sshd is closed, and the intended admin path to this host

Blocked on sudo on the control plane:

- [ ] `kubeadm certs check-expiration` output
- [ ] `/etc/kubernetes/manifests/*` static pod manifests verbatim
- [ ] Whether `kubeadm init` flags survive in root shell history or `/var/log`
