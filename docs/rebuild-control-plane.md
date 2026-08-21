# Rebuild Specification — Control Plane on the Workstation

**Status:** DRAFT FOR REVIEW. Nothing here has been executed.
**Linear:** PET-14 (Phase 0 exit), PET-31, PET-32, PET-33
**Decision:** ADR-003 (workstation = worker + hypervisor), plus the "Option 2" relocation agreed
2026-08-21.
**Inputs:** [`phase0-current-cluster.md`](./phase0-current-cluster.md) — all values below are drawn
from measured state, not assumption.

This document is the reproducible specification the current cluster never had. It is written to be
executed top to bottom on a machine matching the recorded hardware.

---

## 1. Target state

```
                     WORKSTATION (Threadripper 7960X, 24C/48T, 125GiB)
                     ┌──────────────────────────────────────────────┐
                     │  kubeadm control plane + etcd                │
                     │  kubelet (untainted — runs workloads)        │
                     │  libvirt/KVM hypervisor                      │
                     │  GPU 01:00.0  → host nvidia driver           │
                     │  GPU 21:00.0  → vfio-pci (guest passthrough) │
                     │  GPU c1:00.0  → vfio-pci (guest passthrough) │
                     └──────────────────────────────────────────────┘
                                        │ br0 192.168.15.0/24
                                        │
     XPS 13  ── leaves the cluster; retained as a future second hypervisor
```

Single-node cluster to start. Every VM Hydra later creates is local to the API server, so the ~30 ms
inter-site hop is eliminated rather than inherited.

### Why the XPS 13 leaves rather than becoming a worker

It cannot host VMs (no `/dev/kvm`), it is 8 vCPU / 15 GiB against the workstation's 24C/125GiB, and
keeping it as a worker reintroduces the 30 ms hop for any pod scheduled there. Its value is as a
**second hypervisor** later, which is what makes failure-domain and multi-host placement testable
(PET-28). Rejoining it is a deliberate later step, not part of this rebuild.

---

## 2. Decisions this rebuild settles

| # | Decision | Value | Rationale / issue |
|---|---|---|---|
| D1 | API endpoint address | **static `192.168.15.10`** on `br0` | Currently DHCP, which is baked into certs. `.10` is the address the previous cluster generation used and what `hydra-infra`'s tfvars expects — reusing it makes that file correct again. **Verify it is outside the router's DHCP pool first.** PET-32 |
| D2 | `--control-plane-endpoint` | `192.168.15.10:6443` | A stable endpoint from day one so HA (PET-22) does not need a cert regeneration later |
| D3 | Extra cert SANs | `100.100.51.95`, `hycluster-worker-0-1.tail912472.ts.net`, `hycluster-worker-0` | Lets `kubectl` work over the tailnet from any device without re-issuing certs. Cheap now, painful later |
| D4 | Pod CIDR | `10.244.0.0/16`, told to **both** kubeadm and Cilium | Fixes drift #1 — the current cluster's declared subnet is silently unused |
| D5 | Service CIDR | `10.96.0.0/12` | Unchanged; no reason to move |
| D6 | Cilium install method | **Helm**, pinned 1.19.1 | The current cluster used `cilium install` (CLI), which is not reproducible and not GitOps-able. Helm values are the handoff point to `hydra-gitops` |
| D7 | kube-proxy | **replaced by Cilium** | `kubeProxyReplacement=true` + `kubeadm init --skip-phases=addon/kube-proxy`. Currently both run, which is redundant. Cheapest to decide at build time |
| D8 | Pod MTU | **set explicitly**, not auto-detected | Current cluster auto-detected 1230 off the Tailscale interface. Single node on `br0` (1500) with VXLAN → **1450** |
| D9 | Inter-node encryption | **not enabled initially** | Single node; nothing crosses a network. **Must be enabled before the XPS 13 rejoins** — WireGuard via Helm |
| D10 | GPU split | **1 host / 2 passthrough** | Three cards with clean per-device IOMMU groups makes this possible. Resolves ADR-004 — see §7 |
| D11 | Control-plane taint | **removed** | Single-node cluster must schedule workloads |

---

## 3. Resource partition (ADR-003 precondition)

kubelet does not account for memory or CPU held by libvirt guests. Without reservations the
scheduler will hand pods capacity that VMs are already using. Starting allocation of 48 threads /
125 GiB:

| Consumer | CPU | Memory | Notes |
|---|---|---|---|
| Host OS + libvirt daemon | 2 | 4 GiB | |
| **VM budget** | **24** | **64 GiB** | Fenced off via `systemReserved` |
| kubelet / containerd / control plane | 4 | 8 GiB | `kubeReserved` |
| Eviction threshold | — | 2 GiB | `evictionHard` |
| **Left allocatable to pods** | **~18** | **~47 GiB** | |

`KubeletConfiguration` fragment:

```yaml
systemReserved:
  cpu: "26"          # host (2) + VM budget (24)
  memory: "68Gi"     # host (4Gi) + VM budget (64Gi)
kubeReserved:
  cpu: "4"
  memory: "8Gi"
evictionHard:
  memory.available: "2Gi"
```

This is a **starting point to tune**, not a derived optimum. The VM budget is a policy choice: it
should match the largest node pool you intend to run concurrently. Revisit once PET-28 defines the
machine classes.

---

## 4. Procedure

### Phase A — Preconditions (non-disruptive only)

> **Ordering correction 2026-08-21.** An earlier revision of this spec put the `br0` static-address
> change and the containerd restart in Phase A. Both are wrong there. The workstation is still a
> joined worker at `192.168.15.13`; moving it to `.10` changes the node's registered address
> mid-cluster and breaks kubelet, Cilium, and node identity. Restarting containerd restarts every
> pod on the node — including the debug pod that is currently the **only** access path to this host.
> Both have moved to Phase B2, after teardown, where they are free.

1. Confirm `192.168.15.10` is free and outside the router's DHCP pool. **Blocking** — D1 depends on it.
2. Delete the leftover debug pod:
   ```bash
   kubectl get pods -A | grep node-debugger
   kubectl delete pod <name>
   ```
3. Stop cloud-init from managing the network. This only writes files; it changes nothing live:
   ```bash
   sudo mv /etc/netplan/50-cloud-init.yaml /etc/netplan/50-cloud-init.yaml.disabled
   echo 'network: {config: disabled}' | sudo tee /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg
   ```
   Do **not** run `netplan apply` yet — the bridge is still DHCP at this point, deliberately.
4. Complete the non-disruptive part of the host prep the workstation never received (PET-34). None
   of this touches networking or restarts containerd:
   ```bash
   printf 'overlay\nbr_netfilter\n' | sudo tee /etc/modules-load.d/k8s.conf
   sudo modprobe overlay br_netfilter
   cat <<'EOF' | sudo tee /etc/sysctl.d/k8s.conf
   net.bridge.bridge-nf-call-iptables  = 1
   net.bridge.bridge-nf-call-ip6tables = 1
   net.ipv4.ip_forward                 = 1
   EOF
   sudo sysctl --system
   sudo apt-mark hold kubelet kubeadm kubectl
   ```
5. Verify `swapoff` is persistent: `swapon --show` must be empty and `/etc/fstab` free of swap.
6. Confirm the tailnet ACL now permits SSH to the workstation. **Blocking** — see §6.

### Phase B — Tear down (destructive; nothing runs on this cluster)

```bash
# On the XPS 13
sudo kubeadm reset -f && sudo rm -rf /etc/cni/net.d ~/.kube

# On the workstation
sudo kubeadm reset -f && sudo rm -rf /etc/cni/net.d ~/.kube
sudo iptables-save | grep -v -E "KUBE|CILIUM" | sudo iptables-restore
```

### Phase B2 — Disruptive host changes (only safe once the cluster is gone)

Both of these were previously listed in Phase A, incorrectly.

1. Fix the containerd cgroup driver and restart it. Safe now that no pods are running:
   ```bash
   sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
   sudo systemctl restart containerd
   ```
2. Move `br0` to the static address. This changes the host's address, so **do it only with a
   confirmed independent way back in** (Tailscale SSH, per §6):
   ```yaml
   # /etc/netplan/01-bridge.yaml
   network:
     version: 2
     renderer: networkd
     ethernets:
       enp142s0: {dhcp4: no}
     bridges:
       br0:
         interfaces: [enp142s0]
         addresses: [192.168.15.10/24]
         routes: [{to: default, via: 192.168.15.1}]
         nameservers: {addresses: [192.168.15.1, 1.1.1.1]}
         parameters: {stp: false, forward-delay: 0}
   ```
   `sudo netplan try` (auto-reverts after 120 s) before `sudo netplan apply`.
3. Confirm the new address and that `enp142s0` holds none of its own:
   ```bash
   ip -brief addr show br0 enp142s0
   ```

### Phase C — Initialise the control plane on the workstation

```bash
sudo kubeadm init \
  --control-plane-endpoint=192.168.15.10:6443 \
  --apiserver-advertise-address=192.168.15.10 \
  --pod-network-cidr=10.244.0.0/16 \
  --service-cidr=10.96.0.0/12 \
  --apiserver-cert-extra-sans=100.100.51.95,hycluster-worker-0-1.tail912472.ts.net,hycluster-worker-0 \
  --skip-phases=addon/kube-proxy \
  --upload-certs

mkdir -p ~/.kube && sudo cp /etc/kubernetes/admin.conf ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config
```

Record the printed join command in `hydra-bootstrap` immediately — that is the step that was lost
last time.

### Phase D — Cilium via Helm

```bash
helm repo add cilium https://helm.cilium.io && helm repo update
helm install cilium cilium/cilium --version 1.19.1 --namespace kube-system \
  --set ipam.mode=cluster-pool \
  --set ipam.operator.clusterPoolIPv4PodCIDRList='{10.244.0.0/16}' \
  --set ipam.operator.clusterPoolIPv4MaskSize=24 \
  --set kubeProxyReplacement=true \
  --set k8sServiceHost=192.168.15.10 \
  --set k8sServicePort=6443 \
  --set MTU=1450 \
  --set routingMode=tunnel \
  --set tunnelProtocol=vxlan \
  --set operator.replicas=1 \
  --set hubble.relay.enabled=true \
  --set hubble.ui.enabled=true
```

Commit these values to `hydra-gitops` so Argo CD adopts them rather than re-deriving them.

### Phase E — Single-node scheduling and storage

```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml
kubectl patch storageclass local-path -p \
  '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

`hydra-gitops` already carries a local-path manifest — prefer that over the upstream URL once Argo CD
is running.

### Phase F — Apply the kubelet reservations from §3

Write the `systemReserved` / `kubeReserved` block into
`/var/lib/kubelet/config.yaml`, restart kubelet, then confirm:

```bash
kubectl get node -o jsonpath='{.items[0].status.allocatable}'
```

Expect roughly 18 CPU / 47 GiB. If it still reports 48 / 125, the reservations did not take and the
scheduler will overcommit against VMs.

### Phase G — GitOps handoff

Install Argo CD and point the root app at `hydra-gitops`. This is the Phase 7 boundary: after this
point, platform add-ons are changed by commit, not by hand. Covered by PET-17 / PET-18.

---

## 5. Verification

| Check | Command | Expected |
|---|---|---|
| Node ready | `kubectl get nodes -o wide` | 1 node, `Ready`, `192.168.15.10` |
| No kube-proxy | `kubectl -n kube-system get ds` | no `kube-proxy` DaemonSet |
| Pod CIDR honoured | `kubectl get node -o jsonpath='{.items[0].spec.podCIDR}'` | inside `10.244.0.0/16` |
| Cilium healthy | `cilium status` | all OK, kube-proxy replacement enabled |
| Pod MTU | `ip route \| grep 10.244` | `mtu 1450` |
| Reservations live | `kubectl get node -o jsonpath='{.items[0].status.allocatable}'` | ~18 CPU / ~47 GiB |
| Cert SANs | `openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -text \| grep -A2 'Alternative Name'` | contains `.15.10`, `100.100.51.95`, tailnet name |
| kubectl over tailnet | from another tailnet device | API reachable |
| DNS local | `kubectl -n kube-system get pods -o wide -l k8s-app=kube-dns` | on the workstation |

---

## 6. Rollback

There is nothing to preserve — no workloads, no persistent volumes, no StorageClass. Rollback is
re-running the procedure. The only genuine loss risk is **remote access**: a netplan error in
Phase A.4 strands the machine. Mitigations:

- `sudo netplan try` (auto-reverts after 120 s) before `netplan apply`
- Tailscale is independent of `br0`, so `100.100.51.95` should survive a `br0` misconfiguration —
  **provided the tailnet ACL is fixed first**, which it currently is not
- Otherwise the Kubernetes debug-pod back door is the fallback — but it dies with the cluster in
  Phase B

**Fix the tailnet ACL before starting.** Between Phase B and a working Phase D there is a window
with no cluster and therefore no back door.

---

## 7. GPU allocation — supersedes ADR-004's premise

ADR-004 was written believing there was one GPU. There are three, each in its own IOMMU group
(15, 26, 38) with only its companion HDMI audio function. Nothing else shares those groups.

That removes the either/or the ADR was built around:

```
GPU 01:00.0  →  host nvidia driver   →  NVIDIA GPU Operator, bare-metal GPU workloads
GPU 21:00.0  →  vfio-pci             →  guest passthrough
GPU c1:00.0  →  vfio-pci             →  guest passthrough
```

- A GPU node pool of `min: 0, max: 2` is a **real** autoscaling demonstration.
- The host keeps a GPU, so the GPU Operator path and the passthrough path can both be exercised —
  and compared, which is more instructive than either alone.
- Pass each GPU **with** its audio function; they share an IOMMU group and cannot be separated.
- Still unreachable on this hardware: multiple GPU *classes* (all three cards are the same model),
  vGPU (consumer silicon), MIG (datacenter-only). Those roadmap claims still need correcting.

Host config for the two passthrough cards:

```bash
# /etc/modprobe.d/vfio.conf
options vfio-pci ids=10de:2d04,10de:22eb
# plus a driver_override or bind script so 01:00.0 stays on nvidia
```

Binding by device ID alone would capture **all three** cards. Pin by PCI address instead, or use
`driver_override`. This is the one genuinely fiddly step in the whole GPU plan.

---

## 8. Open decisions before execution

1. **Is `192.168.15.10` free** and outside the DHCP pool? Blocks D1/D2.
2. **VM budget** — is 24 vCPU / 64 GiB the right fence? Depends on intended node-pool sizes.
3. **Tailnet ACL** — must be fixed first, per §6.
4. **`hydra-infra` reconciliation** — its storage pool (`/var/lib/libvirt/k8s-workers`) and base
   image (`/home/yibofu/vms/images/...`) paths. Adopt or supersede; do not leave both.
5. **Argo CD scope** in Phase G — the full `hydra-gitops` app-of-apps, or Cilium and storage only
   at first?
