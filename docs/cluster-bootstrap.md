# Bootstrapping a Hydra cluster

How `hydra-wl0` was built: a five-node Kubernetes cluster created entirely by
Cluster API and `cluster-api-provider-hydra`, with **no manual `kubeadm` command
run anywhere**. Written as the specification a future `hydra cluster create`
would automate, so each step notes what a tool should do rather than only what a
person did.

Executed 2026-09-05 against the reference lab (PET-16, PET-17).

## What you need first

| | |
|---|---|
| A management cluster | Any Kubernetes cluster with the CAPI controllers and this provider installed — see [`capi-management-install.md`](capi-management-install.md). It is temporary scaffolding; it does not become part of the cluster you build |
| A libvirt host | Storage pool, base cloud image, and a route from the management cluster to `libvirtd` — see [`host-libvirt-prep.md`](host-libvirt-prep.md) |
| **An API endpoint address** | The one thing you must decide before anything else, and the one that cannot be changed afterwards. See below |

### Choosing the endpoint address

`HydraCluster.spec.controlPlaneEndpoint` is **immutable**, and a
`KubeadmControlPlane` needs an endpoint that survives replicas coming and going.
So the address must be one no DHCP server will ever hand to a machine.

**Confirm that against the DHCP server's configuration. Do not infer it from
addresses you have seen leased.** Building this cluster, that inference was made
three times and was wrong every time: four early leases all landed in
`.200`-`.243`, which looked like evidence the pool began near `.200`; later
machines were handed `.85`, `.90`, `.142`, and finally `.15`. Many DHCP servers
allocate by hashing the client ID rather than counting upward, so the addresses
already issued say nothing about the bounds of the pool.

Getting this wrong is not recoverable in place. The cluster has to be rebuilt.

> A `hydra cluster create` cannot ask a router it knows nothing about. The
> durable answer is for Hydra to own the cluster's addressing — a network with a
> DHCP range it controls, so the endpoint comes from a block it knows is free.
> That is PET-40, and until it lands this step needs a human who can read the
> DHCP configuration.

## 1. Declare the cluster

Four objects, applied together —
[`cluster-with-kubeadm-control-plane.yaml`](https://github.com/Petatron/cluster-api-provider-hydra/blob/main/docs/examples/cluster-with-kubeadm-control-plane.yaml)
in the provider repo is the verified copy:

- **`HydraCluster`** — the endpoint, storage pool, base image, networks
- **`Cluster`** — pod and service CIDRs, pointing at the HydraCluster and the KCP
- **`HydraMachineTemplate`** — the control-plane machine shape
- **`KubeadmControlPlane`** — replicas, version, and the kubeadm config

State the pod CIDR **once**, on `Cluster.spec.clusterNetwork`. KCP propagates it
into kubeadm's configuration, and the CNI must later be given the same value.
Letting those two drift is a real defect in the older hand-built cluster.

Two things in the KCP config are not obvious and both were found the hard way:

- **kube-vip must read `super-admin.conf` for the duration of `kubeadm init`.**
  kubeadm 1.29+ writes an `admin.conf` that is not cluster-admin, so kube-vip
  pointed at it cannot reach the API while init runs — and init then blocks
  waiting for a control plane that can never come up. A pre/post command pair
  swaps the path and swaps it back.
- **`--node-ip` is required.** kube-vip adds the endpoint address to whichever
  node holds leadership, and the kubelet will register *that* as the node's own
  address — giving the node an identity that migrates to another machine on
  failover. Capture the interface's real address before kubeadm starts the
  kubelet.

The base cloud image carries no container runtime or kubeadm, so
`preKubeadmCommands` installs them. **Write `/etc/sysctl.d/k8s.conf` as well** —
`sysctl --system` only applies what is already on disk, and without it kubeadm's
preflight fails on `net.ipv4.ip_forward`.

> `hydra cluster create` should render all four objects from a much smaller
> input: a name, a size, a Kubernetes version, an endpoint. Everything above is
> derivable.

## 2. Wait for the first control-plane node

Hydra creates the VM, cloud-init installs the runtime, kubeadm initialises, and
kube-vip claims the endpoint. Roughly three minutes.

**If the first machine fails to initialise, KCP deadlocks.** It will not roll out
a corrected configuration, because it has no healthy control plane to roll from:
`Initialized=False`, `Available=False`, and the pending machine keeps its stale
bootstrap secret indefinitely. Fix the configuration, then **delete the failed
`Machine` by hand** — KCP builds a fresh one from the updated spec. Its volumes
are reclaimed on the way out.

> A tool should detect this state and say so, rather than leaving an operator
> watching a machine that is never going to progress.

## 3. Install a CNI

The node stays `NotReady` until one exists, and Argo CD cannot run before the
node is Ready — so this one step cannot itself be GitOps-driven.

```
cilium install --set ipam.operator.clusterPoolIPv4PodCIDRList[0]=10.244.0.0/16
```

The CIDR **must** match `Cluster.spec.clusterNetwork.pods`.

> This is the chicken-and-egg in the whole flow, and the reason a tool needs an
> opinion about the CNI rather than delegating it to GitOps.

## 4. Scale the control plane and add workers

Set `KubeadmControlPlane.spec.replicas` to 3. KCP adds them one at a time, and
the endpoint keeps serving throughout — verified here by watching `/healthz`
answer across a leader failover during a rolling replacement.

Workers come from a `MachineDeployment` over a `HydraMachineTemplate` and a
**`KubeadmConfigTemplate`** —
[`machinedeployment-adopted-cluster.yaml`](https://github.com/Petatron/cluster-api-provider-hydra/blob/main/docs/examples/machinedeployment-adopted-cluster.yaml)
shows the shape, though on a cluster built this way you use `configRef` rather
than the `dataSecretName` that file needs.

Workers do **not** take `--node-ip`. That flag exists only because of kube-vip's
floating address on the control plane; a worker has one address and the
kubelet's own choice is correct.

## 5. Hand off to Argo CD

```
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml --server-side
kubectl apply -f clusters/<cluster>/root-app.yaml
```

From here the cluster is driven by commits.

**Check the add-on manifests against what is running before pointing Argo at
them.** Reusing another cluster's tree wholesale nearly renumbered every pod
here: its Cilium application left the pod CIDR unset, which falls back to the
chart default, and set `kubeProxyReplacement` true alongside a running
kube-proxy. Adopt an existing CNI with values transcribed from its live
configuration and the chart pinned to the installed version, so adoption is not
also an upgrade — and leave `prune` off for it.

Also check any add-on that claims addresses. A MetalLB pool of
`192.168.15.200-250` was inherited from the other cluster and is **inside this
site's DHCP range**, which would have MetalLB and the DHCP server handing out the
same addresses.

## What this produced

```
hydra-wl0-cp-2j6vp          Ready  control-plane  v1.35.3  192.168.15.85
hydra-wl0-cp-jqjn4          Ready  control-plane  v1.35.3  192.168.15.142
hydra-wl0-cp-stz2g          Ready  control-plane  v1.35.3  192.168.15.90
hydra-wl0-md-0-6tlfh-56n9z  Ready  <none>         v1.35.3  192.168.15.15
hydra-wl0-md-0-6tlfh-kkrqs  Ready  <none>         v1.35.3  192.168.15.91
```

Every Node's `providerID` matches its Machine. Pods schedule across the workers,
service DNS resolves, and pod-to-service traffic works. Argo CD owns Cilium and
the storage class.

## Known gaps

- **The endpoint address is chosen by a human reading a router.** PET-40.
- **The CNI install is imperative**, for the ordering reason in step 3.
- **Add-on manifests are per-cluster copies** where they differ by only a value
  or two. PET-18.
