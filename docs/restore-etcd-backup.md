# Runbook — restore an etcd backup

**Linear:** [PET-70 / REL-08](https://linear.app/petatron/issue/PET-70/rel-08-back-up-the-management-clusters-own-etcd-off-host)
(management cluster), [PET-23 / REL-02](https://linear.app/petatron/issue/PET-23/rel-02-implement-etcd-backup-restore-and-cluster-recovery-runbook)
(workload cluster) · **ADR:** ADR-007 (management on the office hypervisor, the XPS as off-host observer)
**Backups:** hourly, taken by CronJobs on `hlcluster` (`hlcluster-ctrlr0`, the XPS) — `etcd-backup-hydra-mgmt`
at :37 and `etcd-backup-hydra-wl0` at :07 — written age-encrypted to `/var/lib/hydra-backup/<cluster>/`
on `hlcluster-ctrlr0`. Manifests: hydra-gitops `clusters/hydra-wl0/management-cluster/etcd-backup*.yaml`.
**Clusters:** `hydra-mgmt` (3 KCP members, VIP `192.168.15.11`) and `hydra-wl0` (3 KCP members, VIP
`192.168.15.10`), both Kubernetes v1.35.3, etcd 3.6, on the office hypervisor.
**State verified:** 2026-11-10.

> **Status.** [§2, the verification drill](#2-verification-drill--prove-a-backup-restores) was **executed on
> 2026-11-10** against both clusters' backups; the results are in [§3](#3-results-2026-11-10).
> [§4, full recovery](#4-full-recovery-after-losing-the-hypervisor) is **WRITTEN, NEVER EXECUTED**: it is
> assembled from kubeadm's and Cluster API's documented behaviour and has not been rehearsed.

---

## 0. Read this first — a restored copy must never run workloads

A snapshot holds the whole cluster: every Secret in plaintext (neither cluster encrypts Secrets at rest),
every Deployment, and — for the management cluster — the Hydra provider **with the real hypervisor
credentials** (`libvirt-pki`, `hydra-wl0-libvirt-pki`). Each VM records its owner as the HydraMachine's
UID, and a restored copy has **the same UIDs**. So a copy that schedules pods starts a provider that is,
by the hypervisor's own record, the owner of every live VM: it will scale, remediate and delete them
alongside the real management cluster.

The drill in §2 is safe because it runs **only etcd and kube-apiserver**, on loopback, with no kubelet,
scheduler or controller-manager. Nothing it serves can run. Keep it that way: never point a kubelet,
`kube-controller-manager` or `kube-scheduler` at a restored copy while the original lives, and never
expose its ports.

## 1. What you need

| | Where | Notes |
|---|---|---|
| The age private key | Offline, with the operator only | Public recipient: `age1mmeznvnqvwxvsf4tcynj8j6mjt22s5henll8hlx8ca9e6djegdzq7gv9e9`. Check a key file with `age-keygen -y <file>`, which prints only the public half. |
| `age` | The operator's machine | Decryption happens there; the key never leaves it. |
| sudo on `hlcluster-ctrlr0` | `cp` | The backups are `root` 0600; the drill mounts a tmpfs and reads container images. |
| `etcd`, `etcdutl`, `kube-apiserver` | `hlcluster-ctrlr0`'s image store | `registry.k8s.io/etcd:3.6.6-0` and `registry.k8s.io/kube-apiserver:v1.35.3` are already pulled there; copy the binaries out (step 2.1). |

Each backup job logs what it wrote, for example:

```
wrote /backup/hydra-mgmt-20261110T063700Z.db.age from https://192.168.15.252:2379: revision 1488987, 582 keys, 17264672 bytes before encryption
```

Keep that line: step 2.3 checks the decrypted file against it.

## 2. Verification drill — prove a backup restores

Proves, without touching any live cluster, that a backup decrypts, restores into etcd, and serves the
same objects as the cluster it came from. Run it per cluster; ~10 minutes each.

### 2.1 Workspace (on `cp`, needs sudo)

**Use a dedicated tmpfs, not `/dev/shm`.** On `hlcluster-ctrlr0`, systemd-logind runs with the default
`RemoveIPC=yes` and the user does not linger, so **everything a user owns in `/dev/shm` is deleted when
their last login session ends** — between two `ssh` commands. (Found the hard way on 2026-11-10: the
first decrypt had nowhere to land.) A separate tmpfs mount is not touched.

```sh
D=/run/pet70-drill
sudo mkdir -p $D && sudo mount -t tmpfs -o size=1g,mode=0700,uid=$(id -u),gid=$(id -g) pet70-drill $D
umask 077; mkdir $D/bin $D/pki
for spec in "registry.k8s.io/etcd:3.6.6-0 usr/local/bin/etcd usr/local/bin/etcdutl usr/local/bin/etcdctl" \
            "registry.k8s.io/kube-apiserver:v1.35.3 usr/local/bin/kube-apiserver"; do
  set -- $spec; img=$1; shift
  M=$(mktemp -d); sudo ctr -n k8s.io images mount "$img" $M >/dev/null
  for f in "$@"; do sudo cp $M/$f $D/bin/; done
  sudo ctr -n k8s.io images unmount $M >/dev/null; rmdir $M
done
sudo chown -R $(id -u):$(id -g) $D/bin
```

Verify: `findmnt -no FSTYPE,OPTIONS $D` shows `tmpfs … mode=700`, and
`$D/bin/etcd --version` / `$D/bin/kube-apiserver --version` print 3.6.6 / v1.35.3.

A throwaway PKI for the drill's API server (valid one day; nothing to do with either cluster's real CA):

```sh
cd $D/pki
openssl req -x509 -newkey ec -pkeyopt ec_paramgen_curve:prime256v1 -nodes -keyout ca.key -out ca.crt -days 1 -subj /CN=drill-ca
openssl req -newkey ec -pkeyopt ec_paramgen_curve:prime256v1 -nodes -keyout serving.key -out serving.csr -subj /CN=kube-apiserver
printf 'subjectAltName=IP:127.0.0.1\nextendedKeyUsage=serverAuth\n' > serving.ext
openssl x509 -req -in serving.csr -CA ca.crt -CAkey ca.key -CAcreateserial -days 1 -extfile serving.ext -out serving.crt
openssl req -newkey ec -pkeyopt ec_paramgen_curve:prime256v1 -nodes -keyout admin.key -out admin.csr -subj "/O=system:masters/CN=drill-admin"
printf 'extendedKeyUsage=clientAuth\n' > admin.ext
openssl x509 -req -in admin.csr -CA ca.crt -CAkey ca.key -CAcreateserial -days 1 -extfile admin.ext -out admin.crt
openssl genrsa -out sa.key 2048 && openssl rsa -in sa.key -pubout -out sa.pub
cat > $D/kubeconfig <<EOF
apiVersion: v1
kind: Config
clusters: [{name: drill, cluster: {server: "https://127.0.0.1:16443", certificate-authority: $D/pki/ca.crt}}]
users: [{name: drill, user: {client-certificate: $D/pki/admin.crt, client-key: $D/pki/admin.key}}]
contexts: [{name: drill, context: {cluster: drill, user: drill}}]
current-context: drill
EOF
```

### 2.2 Decrypt (operator, on the machine holding the key)

Streams the encrypted file from `cp`, decrypts it locally, and writes the plaintext over ssh straight
into the tmpfs — it never lands on a disk on either machine:

```sh
ssh cp 'sudo cat /var/lib/hydra-backup/hydra-mgmt/hydra-mgmt-<STAMP>.db.age' \
  | age -d -i <offline key file> | ssh cp 'umask 077; cat > /run/pet70-drill/mgmt.db'
```

Prints nothing on success. `No such file or directory` from the far side means the workspace is gone
(see the `/dev/shm` note in 2.1).

### 2.3 Restore and serve (on `cp`, no sudo)

```sh
D=/run/pet70-drill; B=$D/bin; N=mgmt
$B/etcdutl snapshot status $D/$N.db -w table          # revision and keys MUST equal the job's log line
$B/etcdutl snapshot restore $D/$N.db --name drill --data-dir $D/etcd-$N \
  --initial-cluster drill=http://127.0.0.1:23800 --initial-advertise-peer-urls http://127.0.0.1:23800
rm -f $D/$N.db                                        # the restored data dir is all that's needed now
setsid nohup $B/etcd --name drill --data-dir $D/etcd-$N \
  --listen-client-urls http://127.0.0.1:23790 --advertise-client-urls http://127.0.0.1:23790 \
  --listen-peer-urls http://127.0.0.1:23800 --initial-advertise-peer-urls http://127.0.0.1:23800 \
  --initial-cluster drill=http://127.0.0.1:23800 --listen-metrics-urls http://127.0.0.1:23810 \
  > $D/etcd-$N.log 2>&1 < /dev/null &
setsid nohup $B/kube-apiserver --etcd-servers=http://127.0.0.1:23790 \
  --secure-port=16443 --bind-address=127.0.0.1 --advertise-address=127.0.0.1 \
  --tls-cert-file=$D/pki/serving.crt --tls-private-key-file=$D/pki/serving.key --client-ca-file=$D/pki/ca.crt \
  --service-account-key-file=$D/pki/sa.pub --service-account-signing-key-file=$D/pki/sa.key \
  --service-account-issuer=https://kubernetes.default.svc.cluster.local --service-cluster-ip-range=10.96.0.0/12 \
  --authorization-mode=Node,RBAC --disable-admission-plugins=ValidatingAdmissionWebhook,MutatingAdmissionWebhook \
  --endpoint-reconciler-type=none --profiling=false > $D/apiserver-$N.log 2>&1 < /dev/null &
```

Verify:

- `$B/etcdctl --endpoints http://127.0.0.1:23790 endpoint status -w table` shows the snapshot's revision.
- `kubectl --kubeconfig $D/kubeconfig get --raw /readyz` prints `ok`.
- `ss -ltn | grep -E ':(23790|23800|23810|16443) '` lists **only `127.0.0.1`** addresses.

Webhook admission is off because the restored webhook configurations point at Services that do not
exist here; the drill only reads. The API server still writes its own bootstrap objects (default RBAC,
priority levels) into the restored copy on start — harmless, the copy is discarded.

### 2.4 Compare with the live cluster

Counts per kind, restored against live — they should match, give or take churn since the snapshot's
hour. For each kind:

```sh
R="kubectl --kubeconfig $D/kubeconfig"; L="kubectl --kubeconfig <live kubeconfig>"
for k in namespaces crd nodes "deployments -A" "configmaps -A" "secrets -A" "serviceaccounts -A" clusterroles \
         validatingwebhookconfigurations "clusters.cluster.x-k8s.io -A" "machines.cluster.x-k8s.io -A"; do
  echo "$k restored=$($R get $k --no-headers 2>/dev/null | wc -l) live=$($L get $k --no-headers 2>/dev/null | wc -l)"
done
```

For the management cluster, also compare the provider's backend config by hash, never by printing it:

```sh
for o in cm/libvirt-config secret/libvirt-pki; do
  echo "$o restored=$($R -n cluster-api-provider-hydra-system get $o -o json | jq -S .data | sha256sum | cut -c1-12)" \
       "live=$($L -n cluster-api-provider-hydra-system get $o -o json | jq -S .data | sha256sum | cut -c1-12)"
done
```

### 2.5 Tear down (on `cp`, needs sudo for the unmount)

```sh
D=/run/pet70-drill
pkill -f "^$D/bin/kube-apiserver"; pkill -f "^$D/bin/etcd "
sudo umount $D && sudo rmdir $D
```

Verify: `pgrep -af "^$D/bin/"` prints nothing, no drill port is listening, and `findmnt $D` finds nothing.
To drill a second cluster first, stop the two processes and `rm -rf $D/etcd-<name>`, then repeat 2.2–2.4
with the other backup; the same ports are reused.

## 3. Results, 2026-11-10

Both backups decrypted with the offline key, restored, and served. Nothing of either was left behind
(processes stopped, ports closed, tmpfs unmounted, `/dev/shm` empty).

| | hydra-mgmt `…T063700Z` | hydra-wl0 `…T060700Z` |
|---|---|---|
| Job log | rev 1488987, 582 keys, 17264672 B | rev 21309952, 1125 keys, 42168352 B |
| `etcdutl snapshot status` | rev 1488987, 582 keys, hash `b9885760` | rev 21309952, 1125 keys, hash `25392074` |
| Restored etcd serves | rev 1488987 | rev 21309952 |
| namespaces / CRDs / nodes | 10 / 37 / 3 — **same as live** | 15 / 27 / 6 — **same** |
| Deployments / ConfigMaps / Secrets | 9 / 20 / 8 — **same** | 25 / 125 / 19 — **same** |
| ServiceAccounts / ClusterRoles / webhooks | 58 / 100 / 4 — **same** | 84 / 96 / 3 — **same** |
| Cluster API objects | 0 Clusters / 0 Machines — same as live (see below) | n/a |
| `libvirt-config`, `libvirt-pki` | **hash-equal** to live | n/a |
| Workload detail | Hydra provider `sha-847066f08e67`; the 3 Hydra CRDs carry the move label | radar Deployment+StatefulSet, 7 Argo CD workloads, 11 Applications — same; PV `radar-postgres-data` `Retain`, path intact |

**What this does not yet prove for the management cluster.** On 2026-11-10 hydra-mgmt's etcd held no
Cluster API objects — those still lived on `hlcluster` until the PET-74 pivot. Once it holds them
(after the pivot and the next hourly backup), re-run §2 for hydra-mgmt and add a row: the restored copy
must list both Clusters, all Machines and HydraMachines, and the `hydra-*-ca` / `-etcd` / `-sa` /
`-proxy` / `-kubeconfig` Secrets.

## 4. Full recovery after losing the hypervisor

> **WRITTEN, NEVER EXECUTED.** An outline from documented kubeadm and Cluster API behaviour. Rehearse on
> throwaway clusters before relying on it.

Both clusters run as VMs on the one office hypervisor. If it is lost, the VMs and their disks are gone;
what survives is the backups on `hlcluster-ctrlr0`. The management cluster comes back first, because it
is what rebuilds everything else.

1. **A replacement hypervisor**, prepared as in [host-libvirt-prep.md](./host-libvirt-prep.md), with a
   libvirt TLS listener and a client certificate for the provider.
2. **A single-node control plane for the management cluster**, at the same VIP (`192.168.15.11`) so the
   restored kubeconfigs stay valid, using **hydra-mgmt's own PKI** taken from the restored copy — after
   the pivot, the `hydra-mgmt-ca`, `-etcd`, `-sa` and `-proxy` Secrets are in its own snapshot. Read them
   from a §2 drill copy into `/etc/kubernetes/pki` on the new node; never print them.
3. **Restore the snapshot** as that node's etcd (`etcdutl snapshot restore` with the new member's name
   and URLs), then start only etcd and kube-apiserver — exactly the §2 shape.
4. **Before anything can run a pod**, against that API server:
   - pause every Cluster (`spec.paused: true`);
   - scale the Hydra provider and the Cluster API controllers to 0;
   - point `libvirt-config` (and each cluster's credentials Secret) at the replacement hypervisor.
5. Start the kubelet, controller-manager and scheduler; scale the controllers back up; delete the
   stale Node objects of the lost VMs.
6. Unpause one Cluster at a time. Its Machines' VMs no longer exist, so the provider reports them gone
   and Cluster API replaces them — workload control planes from their own backups per PET-23's
   scenario, workers as fresh machines.

Open questions to settle in the rehearsal: whether KCP accepts a single surviving member after a
3-member restore without manual etcd membership surgery; how the provider treats a providerID whose
domain is absent on a different hypervisor; and the order for hydra-wl0's control plane when its etcd
must come from its own backup rather than from a surviving member.
