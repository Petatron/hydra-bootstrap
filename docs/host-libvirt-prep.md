# Workstation libvirt host preparation — runbook

Steps to finish PET-31 on `hycluster-worker-0`. Written to be executed in order
by someone at a terminal with `sudo`, because most of what remains needs root
and `sudo` there is password-gated.

- **Linear:** [PET-31](https://linear.app/petatron/issue/PET-31/host-01-prepare-workstation-as-libvirt-hypervisor-alongside-bare-metal) (HOST-01)
- **ADR:** ADR-003 (Option C — combined worker and hypervisor)
- **Host:** `hycluster-worker-0`, 192.168.15.13, tailnet 100.111.255.29
- **State verified:** 2026-08-31

Every step says whether it needs root. Run the verification in each step before
moving on — several later steps are wasted work if an earlier one did not take.

---

## What was actually found on the host

Verified directly on 2026-08-31, and **three of PET-31's premises no longer
hold**. Read this before following the issue description.

| Premise in PET-31 | Reality on 2026-08-31 |
|---|---|
| "No VMs exist anywhere in the cluster" | **Three domains already exist** — `wk1`, `wk2`, `wk3`, all `shut off`, each with a qcow2 root disk and a `seed.iso` |
| Storage pool and base image both owed | **Base image already present** at `/home/yibofu/vms/images/ubuntu-24.04-server-cloudimg-amd64.img` (629 MB). Pool still owed |
| `sudo`-only access to libvirt | **`yibofu` is in the `libvirt` and `kvm` groups**, so `virsh -c qemu:///system` works with no root at all |

Everything else checked out as the issue describes:

| Item | State |
|---|---|
| libvirt / qemu | 10.0.0, `libvirtd.socket` active (socket-activated; `libvirtd.service` itself shows `inactive`, which is normal) |
| Socket for a local control path | `/run/libvirt/libvirt-sock`, `root:libvirt`, `srw-rw----` |
| `br0` | UP, `192.168.15.13/24` — guests can attach |
| CPU / memory | 48 vCPU, 125 GiB total |
| **kubelet reservations** | **None.** `capacity` 48 cpu / `allocatable` 48 cpu; memory differs by ~100 Mi (eviction threshold only). The scheduler can hand every core and nearly all memory to pods |
| GPUs | All three (`01:00.0`, `21:00.0`, `c1:00.0`, all `10de:2d04`) on the host `nvidia` driver. `vfio` not loaded. IOMMU active, 51 groups |
| `k8s-workers` pool | Does not exist. `hydra-infra` was never applied |
| Existing pools | `wk1`, `wk2`, `wk3` — one per VM, at `/home/yibofu/vms/<name>`, **not** the single-pool model the provider expects |
| sshd | Not running, nothing listening on 22. Access is Tailscale SSH |

### Two findings that validate PET-9's design for free

`file` on the existing seed image reports **`ISO 9660 CD-ROM filesystem data
'cidata'`**, attached to the domain as `sda`. That is exactly the NoCloud
delivery mechanism PET-9 implemented — a `cidata`-labelled ISO on the SATA bus.
The mechanism is already known to work on this host by hand.

Its contents are **`user-data` and `meta-data` only — no `network-config`**, and
no static addressing anywhere in it (`instance-id: iid-wk1`,
`local-hostname: wk1`). So the manual VMs relied on DHCP on `br0`, which is also
what PET-9's ISO assumes. **Step 3 is where that assumption gets confirmed**, and
it is the one thing in this runbook that could force a code change.

Its `user-data` also installs `qemu-guest-agent` and enables it. That is not
incidental — on this host the agent is the *only* way the provider can learn a
guest's address, and the base image does not ship it. See the callout in Step 3;
it has a direct consequence for PET-37's `KubeadmConfigTemplate`.

---

## Smoke test — already run, 2026-08-31. Both unknowns are answered

A throwaway VM was built and destroyed on this host to settle the two questions
that gated everything else. **Both came back positive**, and the host was left
exactly as found (`wk1`–`wk3` shut off, three pools, nothing added).

| Question | Answer |
|---|---|
| Does a guest on `br0` get an address? | **Yes — `192.168.15.203/24`**, DHCP, on the expected subnet. No `network-config` needed, so PET-9's ISO is correct as written |
| Does `virsh domuuid` match the guest's `/sys/class/dmi/id/product_uuid`? | **Yes, exactly** — both `28c81977-a89c-4a83-b608-420d76ec9bf1`, same byte order, already lowercase |

That second one is the load-bearing answer: **PET-37's providerID design works**,
the guest can derive its own providerID from SMBIOS, and no provider code
changes. The `tr 'A-Z' 'a-z'` in the `preKubeadmCommands` snippet turns out to be
unnecessary — keep it anyway, it costs nothing and guards other hardware.

Proven along the way, all without root:

- A `cidata` ISO built with `genisoimage` is found and consumed by cloud-init
- `local-hostname` in `meta-data` sets the guest hostname (`hydra-smoke`) —
  which is exactly what PET-9's `Hostname` field feeds
- A copy-on-write clone from the base image boots
- **`--source agent` is the only address source that works.** `--source arp`
  returned nothing for the whole run, `--source lease` cannot work for a bridge
  libvirt does not manage. The agent answered within ~15 s of boot

### Three operational traps, all hit during the test

> **`virsh console` needs a controlling TTY and a stdin that stays open.**
> Over a non-interactive SSH it fails with *"Cannot run interactive console
> without a controlling TTY"*; with `ssh -tt` it connects and then exits
> immediately when stdin reaches EOF. Neither failure looks like a VM problem,
> and both waste time. Use the guest agent instead — it needs no terminal:
> ```bash
> virsh -c qemu:///system qemu-agent-command <dom> \
>   '{"execute":"guest-file-open","arguments":{"path":"/sys/class/dmi/id/product_uuid","mode":"r"}}'
> # then guest-file-read with the returned handle; the buffer is base64
> ```

> **A file-backed serial is written as `root:root` mode 0600**, and ownership is
> *not* restored when the domain stops. Configure one and you cannot read your
> own console log without `sudo`. The agent route above sidesteps it entirely.

> **`virt-install` silently defines a storage pool named `dirpool` targeting
> `/`.** It appears in `pool-list` alongside the real pools and enumerates the
> whole root filesystem as volumes. Remove it with `pool-destroy` +
> `pool-undefine` — **never `pool-delete`**, which operates on the files.

### What this leaves

Steps 1 and 2 are the only real work remaining, and neither is large. Step 3 is
now a re-run of a known-good procedure against the production pool rather than
an experiment.

---

## Step 0 — Decide the libvirt control path *(no root; decide before Step 1)*

ADR-003 left two candidates open. The state above changes the balance, and there
is a third the provider already supports.

| Path | Verdict |
|---|---|
| `qemu+ssh://` | **Not now.** Nothing listens on 22 and access is Tailscale SSH, which authenticates interactive tailnet identities rather than service accounts. This means standing up a conventional `sshd` with a dedicated keypair purely for the provider — new attack surface, and it still crosses the ~30 ms inter-site hop |
| **hostPath `/run/libvirt/libvirt-sock` + `nodeSelector`** | **Recommended for now.** No SSH, no PKI, no WAN hop for libvirt RPC. `config/manager/manager.yaml` already carries a commented `[LOCAL-LIBVIRT]` block for exactly this. Also sidesteps PET-36 entirely, which only affects the TLS dialer |
| `qemu+tls://` | **The eventual answer, not this step.** The provider supports it (`--libvirt-remote-addr`, `--libvirt-pki-path`), and it is what principle 9 and the multi-hypervisor model point at — but it needs a CA and certificates, and PET-36's unbounded verification read is on this path |

**Recommendation: take the hostPath socket, and record it in ADR-003 as a
deliberate stepping stone rather than the destination.** It pins the controller
to this node, which does conflict with the future multi-hypervisor model — the
point is to get PET-37 provable with the fewest new moving parts, then move to
TLS once something is known to work end to end.

---

## Step 1 — Fence off a VM budget from the scheduler *(ROOT)*

The non-negotiable item in ADR-003, and currently not done at all: kubelet does
not account for memory or CPU held by libvirt guests, so as things stand the
scheduler will happily allocate the entire host to pods and then the VMs and the
pods will fight.

Proposed split of 48 vCPU / 125 GiB:

| Consumer | CPU | Memory |
|---|---|---|
| libvirt guests + OS (`systemReserved`) | 18 | 52 Gi |
| kubelet/containerd (`kubeReserved`) | 1 | 2 Gi |
| Left allocatable to pods | ~29 | ~71 Gi |

The guest share of that is **16 vCPU / 48 GiB**, which fits four 4-vCPU/8-GiB
workers with room, and leaves the larger share to pods since that is what this
node is currently doing.

Set it in `/etc/default/kubelet`, **not** in `/var/lib/kubelet/config.yaml`:

```bash
sudo cp /etc/default/kubelet /etc/default/kubelet.bak-$(date +%Y%m%d)
printf 'KUBELET_EXTRA_ARGS=--system-reserved=cpu=18,memory=52Gi --kube-reserved=cpu=1,memory=2Gi\n' \
  | sudo tee /etc/default/kubelet
sudo systemctl restart kubelet
```

> ### Why not `config.yaml`, which is the obvious place
>
> Because **`kubeadm upgrade node` regenerates it** from the cluster-wide
> `kubelet-config` ConfigMap. A reservation written there is node-local
> configuration living in a file kubeadm owns, and it disappears at the next
> upgrade — silently, months later, with the symptom being pods and VMs
> fighting over a host that looks correctly configured.
>
> `/etc/default/kubelet` is node-local and kubeadm does not rewrite it. This
> unit sources it (`EnvironmentFiles=/etc/default/kubelet`) and expands
> `$KUBELET_EXTRA_ARGS` **last** in `ExecStart`, so these flags also win over
> anything in the config file.
>
> The cost is that `--system-reserved` and `--kube-reserved` are flags, and
> kubelet flags are deprecated in favour of config. They still work in 1.35 and
> this cluster is explicitly temporary, so the trade is worth it. The tidy
> long-term form is a `--config-dir` drop-in
> (`/etc/kubernetes/kubelet.conf.d/`), which is upgrade-safe *and* uses the
> non-deprecated mechanism — two changes instead of one, worth doing if this
> node ever stops being disposable.

> **Deliberately no `systemReservedCgroup`.** Without it these numbers only
> change the `Allocatable` arithmetic — they reserve headroom from the
> *scheduler* rather than hard-limiting anything. That is what is wanted here:
> a cgroup cap would throttle libvirt itself. If someone later "fixes" this by
> adding the cgroup, they will have changed what it does.

> **Editing over VS Code Remote SSH will not work** for this or any root-owned
> file — the remote server runs as `yibofu` and gets `EACCES`. Use a terminal
> with `sudo`.

**Verify** — `allocatable` must now be visibly below `capacity`:

```bash
kubectl get node hycluster-worker-0 \
  -o jsonpath='{.status.capacity.cpu}/{.status.allocatable.cpu} cpu  {.status.capacity.memory}/{.status.allocatable.memory} mem{"\n"}'
```

Expect roughly `48/29 cpu` and allocatable memory near `74600000Ki` (~71 GiB).
If it still reports `48/48`, the restart did not pick the flags up — check
`journalctl -u kubelet -n 30`.

---

## Step 2 — Storage pool and base image *(ROOT for the directory, then no root)*

The provider takes **one** pool that holds both the base image and every machine
clone. The existing per-VM `wk1`/`wk2`/`wk3` pools do not fit that model and are
left alone.

Using `hydra-infra`'s never-applied names deliberately, so the two stop
diverging: pool `k8s-workers` at `/var/lib/libvirt/k8s-workers`.

```bash
sudo mkdir -p /var/lib/libvirt/k8s-workers
sudo chown root:libvirt /var/lib/libvirt/k8s-workers
sudo chmod 0771 /var/lib/libvirt/k8s-workers

# Copy rather than move: the original is what the existing wk1-3 disks back onto.
sudo cp /home/yibofu/vms/images/ubuntu-24.04-server-cloudimg-amd64.img \
        /var/lib/libvirt/k8s-workers/
sudo chown libvirt-qemu:kvm /var/lib/libvirt/k8s-workers/ubuntu-24.04-server-cloudimg-amd64.img
```

Then, as your own user (no root — you are in the `libvirt` group):

```bash
virsh -c qemu:///system pool-define-as k8s-workers dir \
  --target /var/lib/libvirt/k8s-workers
virsh -c qemu:///system pool-build k8s-workers
virsh -c qemu:///system pool-start k8s-workers
virsh -c qemu:///system pool-autostart k8s-workers
virsh -c qemu:///system pool-refresh k8s-workers
```

**Verify** — the base image must appear as a **volume**, not just a file:

```bash
virsh -c qemu:///system vol-list k8s-workers
```

> **The provider resolves the base image by volume name, not path.**
> `--libvirt-base-image` takes `ubuntu-24.04-server-cloudimg-amd64.img` and looks
> it up with `StorageVolLookupByName` inside `--libvirt-storage-pool`. A file
> sitting in the directory that the pool has not been refreshed to see does not
> count, and produces a terminal `image not found in pool` on first provision.

---

## Step 3 — Prove a VM can be created and destroyed *(no root)*

Acceptance criterion 1, and it also answers **PET-37's blocking question** for
free, so do not skip the DMI check at the end.

```bash
cd /tmp && mkdir -p pet31 && cd pet31

cat > meta-data <<'EOF'
instance-id: iid-pet31-smoke
local-hostname: pet31-smoke
EOF

cat > user-data <<'EOF'
#cloud-config
password: pet31
chpasswd: { expire: false }
ssh_pwauth: true
# Not optional -- see the callout below. Copied from the existing wk1 seed.iso,
# which needed the same thing.
package_update: true
packages:
  - qemu-guest-agent
runcmd:
  - systemctl enable --now qemu-guest-agent
EOF

# Same shape as the existing wk1 seed.iso, and as what the provider builds.
genisoimage -output seed.iso -volid cidata -joliet -rock user-data meta-data

virsh -c qemu:///system vol-create-as k8s-workers pet31-smoke.qcow2 20G \
  --format qcow2 \
  --backing-vol ubuntu-24.04-server-cloudimg-amd64.img \
  --backing-vol-format qcow2

virt-install --connect qemu:///system \
  --name pet31-smoke --memory 2048 --vcpus 2 --import \
  --disk vol=k8s-workers/pet31-smoke.qcow2,bus=virtio \
  --disk path=/tmp/pet31/seed.iso,device=cdrom,bus=sata,readonly=on \
  --network bridge=br0,model=virtio \
  --os-variant ubuntu24.04 --graphics none --noautoconsole
```

`genisoimage`, `virt-install`, `xorriso` and `qemu-img` are already installed on
this host — checked 2026-08-31, nothing to install for this step.

> ### The guest agent is the only way to learn a bridged guest's address
>
> `virsh domifaddr --source lease` reads libvirt's own DHCP leases, and libvirt
> only has those for networks **it** manages. These guests attach to `br0`,
> which it does not manage — `net-dhcp-leases default` is empty and `virbr0` is
> down and unused. So on this host the lease source will *never* return
> anything, and the **QEMU guest agent is the only address source that works**.
>
> The Ubuntu cloud image does not ship the agent. The existing `wk1` user-data
> installs it explicitly (`packages: [qemu-guest-agent]` plus a `runcmd` to
> enable it), which is how we know it is missing from the image rather than
> merely disabled.
>
> **Consequence for PET-37:** its `KubeadmConfigTemplate` must install
> `qemu-guest-agent` too, or the provider's `addressesOf` will come back empty
> forever, `status.addresses` will stay unset, and the machine will sit on the
> 15-second provisioning requeue instead of settling to the slow health
> interval. The alternative is baking the agent into the base image, which is
> tidier for a fleet and worth considering once there is more than one.

**Verify.** The two questions below were already answered by the 2026-08-31
smoke test; re-running them here confirms the *production pool* works, not the
technique:

```bash
# 1. it runs
virsh -c qemu:///system list

# 2. THE DHCP QUESTION. Does a guest on br0 get an address on 192.168.15.0/24?
#    The existing seed.iso carries no network-config, so both the manual VMs and
#    PET-9's generated ISO depend on this. If it comes up empty, PET-9 needs a
#    network-config file and that is a code change.
#    Give cloud-init a minute first -- the agent is installed on first boot.
virsh -c qemu:///system domifaddr pet31-smoke --source agent
#    Expected to be EMPTY on this host, and that is not a failure -- libvirt has
#    no leases for a bridge it does not manage. Run it only to confirm that.
virsh -c qemu:///system domifaddr pet31-smoke --source lease

# 3. THE PROVIDERID QUESTION (PET-37 step 1). These two must match exactly,
#    including case, or the whole self-derived providerID design fails.
virsh -c qemu:///system domuuid pet31-smoke
#    then, on the guest console (virsh console pet31-smoke, user ubuntu / pet31):
#      cat /sys/class/dmi/id/product_uuid
```

Record both answers — they decide whether PET-37 can proceed as designed.

**Tear down**, which is the other half of criterion 1:

```bash
virsh -c qemu:///system destroy pet31-smoke
virsh -c qemu:///system undefine pet31-smoke
virsh -c qemu:///system vol-delete pet31-smoke.qcow2 --pool k8s-workers
```

---

## Step 4 — GPU binding per ADR-004 *(ROOT, and needs a reboot — read the warning)*

All three GPUs are currently on the host `nvidia` driver and `vfio` is not
loaded. ADR-004 wants `01:00.0` left on `nvidia` and `21:00.0` + `c1:00.0` bound
to `vfio-pci`.

> **Pin by PCI address, never by device ID.** All three cards report
> `10de:2d04`, so a `vfio-pci.ids=10de:2d04` rule captures **all three** and
> leaves the host with no GPU at all.

> ### Do PET-34's persistence gaps before rebooting this host
>
> This step needs a reboot, and PET-34 has three unresolved reboot-survival
> gaps on this machine: `overlay` missing from `/etc/modules-load.d/k8s.conf`,
> `/etc/sysctl.d/k8s.conf` absent (the sysctls are set at runtime only), and the
> `50-cloud-init.yaml` netplan conflict. Rebooting now risks a node that does
> not come back as a working Kubernetes worker, and debugging that on top of a
> GPU change is worse than doing them separately.
>
> **Suggestion: skip Step 4 for now.** Nothing in PET-37 needs GPU passthrough —
> the first VM-backed pool is a plain test pool. Close PET-31's other criteria,
> clear PET-34, then come back to this with one reboot that exercises both.

If proceeding anyway, use `driverctl` so the binding is by address and
persistent, rather than hand-editing initramfs:

`driverctl` is **not** installed on this host (checked 2026-08-31), unlike the
Step 3 tooling.

```bash
sudo apt install driverctl
sudo driverctl set-override 0000:21:00.0 vfio-pci
sudo driverctl set-override 0000:c1:00.0 vfio-pci
sudo driverctl list-overrides
```

**Verify** after reboot — `01:00.0` on `nvidia`, the other two on `vfio-pci`:

```bash
lspci -nnk -d 10de: | grep -E '^[0-9a-f]{2}:|Kernel driver in use'
```

---

## Step 5 — Decide what happens to `wk1`, `wk2`, `wk3` *(no root)*

Not in PET-31's description, because it did not know they existed. They are
`shut off`, so they cost nothing today — but they are
[PET-29](https://linear.app/petatron/issue/PET-29/mig-01-define-and-execute-terraform-to-capi-worker-ownership-cutover)'s
split-brain concern in concrete form: three VMs on the host Hydra is about to
start managing, created by something that is not Hydra.

They were not made by `hydra-infra` either — its pool is
`/var/lib/libvirt/k8s-workers` and these sit in `/home/yibofu/vms/<name>`, so
they are from an earlier by-hand attempt.

**Recommendation: leave them shut off and untouched, and record them.** They are
useful reference — `wk1`'s `seed.iso` is the only known-good cloud-init image on
this host. What matters is that Hydra never points at pools `wk1`/`wk2`/`wk3`,
and it will not: it is configured for `k8s-workers`.

```bash
virsh -c qemu:///system list --all
virsh -c qemu:///system pool-list --all
```

---

## Step 6 — Record the results

PET-31's remaining acceptance criteria are documentation:

- Fill in the **Results** section below
- Record the chosen control path and the resource split in **ADR-003**, closing
  its open questions
- Note in the PET-31 worklog that the "no VMs exist" premise was wrong

### Results — completed 2026-09-01

| Step | Outcome |
|---|---|
| 0 — control path | hostPath `/run/libvirt/libvirt-sock` + `nodeSelector`, **proven from a pod** (see below) |
| 1 — kubelet reservations | **Done.** `48/29 cpu`, `131371300Ki/74645796Ki mem` (~71 GiB allocatable). Set in `/etc/default/kubelet`, kubelet healthy |
| 2 — pool and base image | **Done.** Pool `k8s-workers` active + autostart, 914.78 GiB. Base image registered as a volume under the name the provider expects |
| 3 — VM create/destroy | **Done, against the production pool.** Both disks as pool volumes, exactly the shape the provider produces |
| 3 — DHCP on `br0`? | **Yes**, twice — `192.168.15.203/24` and `192.168.15.82/24` |
| 3 — guest agent reachable? | **Yes**, ~10 s after boot. Only working address source |
| 3 — `domuuid` == `product_uuid`? | **Yes, exact, twice** — `28c81977-…` and `d39d0480-41bb-40c1-9b0c-4ef7de075480` |
| 3 — teardown reclaims both volumes | **Yes.** Pool back to just the base image, 601 M |
| 4 — GPU binding | *Deferred — see the reboot warning. Nothing in PET-37 needs it* |
| 5 — `wk1`–`wk3` disposition | Left shut off and untouched, as recommended |

### The management cluster's path to libvirtd, proven

Criterion 2 asks for a *working* path, so it was tested rather than assumed. A
pod carrying the manager's exact security context was scheduled onto
`hycluster-worker-0` with the socket mounted, twice:

| Pod `securityContext` | Result |
|---|---|
| `supplementalGroups: [126]` | `groups=65532 126` → socket **writable**, connect would succeed |
| omitted | `groups=65532` → socket **not writable**, provider would fail |

> **The mount alone is not enough, and the failure does not look like a
> permission problem.** The socket is `srw-rw---- root:libvirt` while the
> manager runs as nonroot uid 65532, so without the group it is mounted,
> visible and unopenable — and libvirt reports that as a connection failure.
> The gid is per-host (126 on Ubuntu 24.04 here); read it with
> `getent group libvirt`, do not copy the number. Now documented in the
> provider's `config/manager/manager.yaml`.

### One trap avoided this time

Putting the cloud-init ISO into the pool as a **volume**, and referencing it as
`vol=k8s-workers/…` rather than a loose path, stops `virt-install` defining its
`dirpool` over `/`. It also matches what the provider actually does, so the test
exercised the real shape rather than an approximation.

---

## What this does *not* do

Deploying the provider itself. Once Step 2 is done, the manager needs the
`[LOCAL-LIBVIRT]` configuration from `config/manager/manager.yaml` — hostPath
mount of `/run/libvirt/libvirt-sock`, a `nodeSelector` pinning it to
`hycluster-worker-0`, and the `libvirt-config` ConfigMap set to
`storagePool: k8s-workers` and
`baseImage: ubuntu-24.04-server-cloudimg-amd64.img`. That belongs to PET-37,
which is where it gets exercised.
