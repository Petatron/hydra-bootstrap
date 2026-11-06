# Runbook — the management control plane's address changes

**Linear:** [PET-32 / NET-01](https://linear.app/petatron/issue/PET-32/net-01-harden-control-plane-network-and-api-endpoint-assumptions)
**Target:** `hlcluster-ctrlr0`, `192.168.16.10` on USB NIC `enx9c69d34cc8a1`, Kubernetes v1.35.3,
single control plane, stacked etcd
**State verified:** 2026-11-06, read-only. The inventory in [§3](#3-what-carries-the-address) was
taken from the live cluster on that date.

> **Status: WRITTEN, NEVER EXECUTED.** No step here has been run. [§2](#2-recovery-put-the-old-address-back)
> is the expected path and touches nothing in Kubernetes; [§4](#4-last-resort-re-address-the-cluster)
> is a last resort assembled from kubeadm's behaviour and the inventory below, and should be
> rehearsed on a throwaway kubeadm cluster before it is ever needed for real.

The management cluster's API address is a DHCP lease, and that address is written into its
certificates, its etcd membership, every kubeconfig, and two Cluster API objects
([phase0 §9](./phase0-current-cluster.md#9-network-re-check--2026-11-06-pet-32), item 1). If the lease
changes, every client — the workers' kubelets, every Cluster API controller's view of the `hydra`
cluster, your own `kubectl` — keeps dialling the old address. Nothing heals by itself.

The durable fix is an endpoint that is not a host address (a VIP or a DNS name). That is a
[PET-74](https://linear.app/petatron/issue/PET-74/mig-03-move-management-onto-a-cluster-cluster-api-builds-so-the)
acceptance criterion for the management cluster that replaces this one. This runbook covers the
cluster until then.

---

## 1. Prevent it — fixed IP on the home router

**Who:** the home UniFi console's owner. No root on any host.

1. UniFi Network → **Client Devices** → `hlcluster-ctrlr0` (MAC of `enx9c69d34cc8a1`) →
   **Settings** → **Fixed IP Address** → `192.168.16.10`.
2. Verify, on `hlcluster-ctrlr0`:
   ```bash
   ip -4 -br addr show enx9c69d34cc8a1    # still 192.168.16.10/27
   ```

The reservation is bound to the **dongle's MAC**. Replacing the USB adapter, or plugging a
different one in, gives a new MAC and therefore a new lease — move the reservation to the new MAC
before, or immediately after, swapping hardware.

## 2. Recovery: put the old address back

**When:** `kubectl` against `https://192.168.16.10:6443` times out and the node has a different
address. This is the expected recovery: it changes nothing in Kubernetes, so there is nothing to
get wrong there.

1. Confirm the cause, on `hlcluster-ctrlr0` (console or Tailscale SSH, which does not depend on the
   LAN address):
   ```bash
   ip -4 -br addr show enx9c69d34cc8a1      # shows the new address
   ```
2. **Preferred:** set or correct the fixed IP in UniFi as in [§1](#1-prevent-it--fixed-ip-on-the-home-router),
   then renew the lease. The interface is managed by **NetworkManager** (connection
   `netplan-enx9c69d34cc8a1`; `systemd-networkd` is inactive and reports it `unmanaged`, so
   `networkctl` does nothing here). Needs root, and drops the link for a few seconds:
   ```bash
   sudo nmcli connection up netplan-enx9c69d34cc8a1
   ip -4 -br addr show enx9c69d34cc8a1      # 192.168.16.10/27 again
   ```
3. **Stopgap** if the router cannot be changed right now and `192.168.16.10` is not in use by
   anything else on the LAN — add it back as a secondary address. Needs root; lost on reboot or
   when NetworkManager reapplies the connection:
   ```bash
   ping -c 2 -W 1 192.168.16.10             # must NOT answer before you add it
   sudo ip addr add 192.168.16.10/27 dev enx9c69d34cc8a1
   ```
4. Verify, in this order:
   ```bash
   kubectl get --raw /livez                                    # ok
   kubectl get nodes                                           # all Ready within ~1 min
   kubectl get cluster hydra-wl0 -o jsonpath='{.status.conditions[?(@.type=="RemoteConnectionProbe")].status}'   # True
   ```

## 3. What carries the address

Measured 2026-11-06. Secret **names** were found by searching decoded data without printing it;
never print their contents.

| Where | What holds `192.168.16.10` |
|---|---|
| `/etc/kubernetes/pki/apiserver.crt` | SAN (with `10.96.0.1` and the default DNS names) |
| `/etc/kubernetes/pki/etcd/server.crt`, `peer.crt` | SAN |
| etcd static pod | `--advertise-client-urls`, `--initial-advertise-peer-urls`, `--initial-cluster`, `--listen-client-urls`, `--listen-peer-urls`; the member's stored peer URL in etcd itself |
| kube-apiserver static pod | `--advertise-address`; the `kube-apiserver.advertise-address.endpoint` annotation |
| `/etc/kubernetes/{admin,super-admin,controller-manager,scheduler,kubelet}.conf` | `server:` |
| ConfigMaps | `kube-public/cluster-info`, `kube-system/kube-proxy` (its kubeconfig `server:`), `monitoring/prometheus-radar-prometheus-rulefiles-0` (generated from PrometheusRule `silkroad-radar`) |
| Cluster API objects | `Cluster hydra` and `HydraCluster hydra` `spec.controlPlaneEndpoint` (both immutable) |
| Secrets | `default/hydra-kubeconfig`, `default/hydra-worker-0-bootstrap` (the join data both `hydra-md-0` and `hydra-worker-0` boot from, PET-61) |
| Worker VMs (`hydra-md-0-mj9gx-4c6dv`, `hydra-worker-0`) | `/etc/kubernetes/kubelet.conf` `server:` |
| Operators' machines | local kubeconfigs, e.g. `~/.kube/hlcluster.yaml`, `~/.kube/hlcluster-socks.yaml` |

Not affected: Cilium (no `k8sServiceHost` override — it reaches the API through the `kubernetes`
Service), and `hydra-wl0` itself (its own VIP; the `hydra-backup` etcd jobs run *on* this cluster
but target `hydra-wl0`).

## 4. Last resort: re-address the cluster

**Only if the old address can never come back.** Every step needs root on `hlcluster-ctrlr0`
(passwordless sudo exists there) or on the worker VMs. `NEW` below is the new address.

1. **Back up first.**
   ```bash
   sudo cp -a /etc/kubernetes /root/etc-kubernetes.before-readdress
   sudo ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
     --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/server.crt \
     --key=/etc/kubernetes/pki/etcd/server.key snapshot save /root/etcd-before-readdress.db
   ```
   (Run while the API is still up on `127.0.0.1` — etcd listens there regardless of the LAN address.)
2. **Certificates** — re-issue the three that carry the address, from the same CA:
   ```bash
   sudo mkdir /root/old-certs
   sudo mv /etc/kubernetes/pki/apiserver.{crt,key} /etc/kubernetes/pki/etcd/{server,peer}.{crt,key} /root/old-certs/
   kubectl -n kube-system get cm kubeadm-config -o jsonpath='{.data.ClusterConfiguration}' > /tmp/cc.yaml
   # write /tmp/readdress.yaml: an InitConfiguration with localAPIEndpoint.advertiseAddress: NEW
   # and nodeRegistration.name: hlcluster-ctrlr0, then '---', then /tmp/cc.yaml
   sudo kubeadm init phase certs apiserver   --config /tmp/readdress.yaml
   sudo kubeadm init phase certs etcd-server --config /tmp/readdress.yaml
   sudo kubeadm init phase certs etcd-peer   --config /tmp/readdress.yaml
   openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -ext subjectAltName   # NEW present
   ```
3. **Static pods** — replace the address in both manifests; the kubelet restarts them:
   ```bash
   sudo sed -i "s/192\.168\.16\.10/NEW/g" /etc/kubernetes/manifests/etcd.yaml /etc/kubernetes/manifests/kube-apiserver.yaml
   ```
   etcd keeps its **stored** peer URL from the data directory; a single member runs without peers,
   so it starts, but correct the record once it is up:
   `etcdctl member list` then `etcdctl member update <id> --peer-urls=https://NEW:2380`
   (same TLS flags as step 1).
4. **Control-plane kubeconfigs** — point every `server:` at the new address:
   ```bash
   sudo sed -i "s#https://192\.168\.16\.10:6443#https://NEW:6443#" /etc/kubernetes/{admin,super-admin,controller-manager,scheduler,kubelet}.conf
   sudo systemctl restart kubelet
   kubectl --kubeconfig /etc/kubernetes/admin.conf get --raw /livez    # ok (copy admin.conf to ~/.kube/config if that is what you use)
   ```
5. **In-cluster references:**
   ```bash
   kubectl -n kube-public edit cm cluster-info          # server:
   kubectl -n kube-system edit cm kube-proxy            # kubeconfig.conf server:
   kubectl -n kube-system rollout restart ds kube-proxy
   kubectl -n monitoring edit prometheusrule silkroad-radar   # wherever it names the address
   ```
6. **Workers** — on each worker VM, as root (Hydra VMs: console or guest agent):
   ```bash
   sed -i "s#https://192\.168\.16\.10:6443#https://NEW:6443#" /etc/kubernetes/kubelet.conf
   systemctl restart kubelet
   ```
7. **Cluster API.** Update the two Secrets that carry the address — `hydra-kubeconfig` (the
   kubeconfig Cluster API uses to reach this cluster) and `hydra-worker-0-bootstrap` (the join
   data any replacement worker boots from) — by writing new data, never by printing the old.
   `Cluster hydra` / `HydraCluster hydra` `spec.controlPlaneEndpoint` are **immutable**: they will
   keep showing the old address. Recreating them would cascade to the management cluster's own
   Machines, so leave them; PET-74 retires the adopted `hydra` Cluster.
8. **Verify:**
   ```bash
   kubectl get nodes -o wide                         # all Ready; control plane INTERNAL-IP = NEW
   kubectl -n kube-system get pods                   # etcd, apiserver, kube-proxy Running
   kubectl get clusters -A                           # hydra-wl0 Available
   ```
   then update local kubeconfigs on operator machines, and record the change in
   [phase0-current-cluster.md](./phase0-current-cluster.md) and on PET-32's Notion page.

The control plane's certificates all expire **2027-04-14**
([phase0](./phase0-current-cluster.md#certificate-validity--measured-2026-08-21)). `kubeadm certs renew`
keeps the SANs it finds, so renewal neither fixes nor breaks anything here.
