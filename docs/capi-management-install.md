# Cluster API management components

How the existing Hydra cluster was made the Cluster API management cluster, and
how to reproduce or undo it.

- **Linear:** [PET-5](https://linear.app/petatron/issue/PET-5/capi-01-install-cluster-api-management-components-on-the-existing) (CAPI-01)
- **Performed:** 2026-08-31
- **Target:** `hlcluster-ctrlr0` (192.168.16.10, tailnet 100.116.30.60), Kubernetes v1.35.3

## What is installed

| Component | Version | Namespace |
|---|---|---|
| Cluster API core | v1.14.0 | `capi-system` |
| Kubeadm bootstrap provider (CABPK) | v1.14.0 | `capi-kubeadm-bootstrap-system` |
| Kubeadm control-plane provider (KCP) | v1.14.0 | `capi-kubeadm-control-plane-system` |
| cert-manager | v1.21.1 | `cert-manager` |

18 CRDs under `*.cluster.x-k8s.io`. No infrastructure provider — see
[No infrastructure provider](#no-infrastructure-provider) below.

## Why v1.14.0 specifically

Not "the latest" by coincidence. `cluster-api-provider-hydra` depends on
`sigs.k8s.io/cluster-api/api v1.14.0` and implements the **v1beta2** contract,
so the management cluster has to speak the same one. Every CRD installed here
reports `v1beta2` as its stored version, which is the check that matters:

```bash
kubectl get crd machines.cluster.x-k8s.io -o jsonpath='{.status.storedVersions}'
```

Two version constraints were verified before installing rather than after:

- **CAPI v1.14.0 supports management clusters v1.33.x–v1.36.x.** This cluster is
  v1.35.3, comfortably inside that range. Outside it, the controllers install
  but are unsupported.
- **v1beta1 contract support is on track to be dropped in CAPI v1.16.** The
  Hydra provider already implements v1beta2, so nothing to do — but it means
  v1beta1 is not a fallback if something here needs revisiting.

## Reproducing it

`clusterctl` is installed per-user rather than system-wide, because `sudo` on
this host is password-gated and a release binary in `~/bin` needs no
privileges. It is **not on `PATH`** — invoke it by full path, the same
convention `kubebuilder` follows in this project.

```bash
V=v1.14.0
mkdir -p ~/bin
curl -fsSLO "https://github.com/kubernetes-sigs/cluster-api/releases/download/${V}/clusterctl-linux-amd64"
install -m 0755 clusterctl-linux-amd64 ~/bin/clusterctl
~/bin/clusterctl version
```

> The release publishes **no checksum asset** for `clusterctl` — only the
> platform binaries. Integrity here rests on HTTPS to the canonical
> `kubernetes-sigs/cluster-api` repository plus a byte-size match against the
> GitHub release API (34,861,218 bytes for `clusterctl-linux-amd64` at v1.14.0).
> That is weaker than a published digest; do not describe it as checksum
> verification.

```bash
~/bin/clusterctl init \
  --core cluster-api:v1.14.0 \
  --bootstrap kubeadm:v1.14.0 \
  --control-plane kubeadm:v1.14.0
```

Versions are pinned explicitly on every flag. Bare `clusterctl init` resolves
"latest", which would silently drift away from the contract version the provider
was built against on any future re-run.

## No infrastructure provider

`--infrastructure` is deliberately omitted. `cluster-api-provider-hydra` is not
published to a provider registry, so `clusterctl` cannot fetch it; it is
deployed from its own repository with `make deploy`. Adding it to `clusterctl`'s
configuration is worth doing once there are releases to point at, and is not a
prerequisite for anything today.

## cert-manager is a side effect worth knowing about

`clusterctl init` installs cert-manager if it is absent, and it was absent here.
It is a cluster-wide component this cluster did not previously have, and it is
now a dependency of the CAPI webhooks. Anything else that later wants
cert-manager must reconcile with this installation rather than install a second
one.

## Verifying

```bash
# every API the acceptance criteria name, with its stored contract version
for k in clusters machines machinedeployments machinesets machinehealthchecks; do
  printf '%-40s ' "$k"
  kubectl get crd "$k.cluster.x-k8s.io" -o jsonpath='{.status.storedVersions}'; echo
done
kubectl get crd kubeadmconfigs.bootstrap.cluster.x-k8s.io -o jsonpath='{.status.storedVersions}'; echo
kubectl get crd kubeadmcontrolplanes.controlplane.cluster.x-k8s.io -o jsonpath='{.status.storedVersions}'; echo

# controllers, and what clusterctl thinks it installed
kubectl get deploy -A | grep -E 'capi-|cert-manager'
kubectl get providers -A
```

Result on 2026-08-31: all eight named APIs present at `v1beta2`; all six
deployments (3 CAPI + 3 cert-manager) `1/1` and `Running`; both nodes still
`Ready` at v1.35.3.

## Undoing it

```bash
~/bin/clusterctl delete --all --include-crd --include-namespace
```

`--include-crd` deletes the CRDs and therefore **every `Cluster`, `Machine` and
`MachineDeployment` object with them**. On a cluster where CAPI owns real
workers that destroys the records of them, not just the controllers — the VMs
would be orphaned with nothing left describing them. Safe right now only because
no CAPI objects exist. cert-manager is left behind by `clusterctl delete` and
must be removed separately if that is wanted.

## Notes

- There is no workload on this cluster, so this needed no maintenance window.
  The only non-system pod was a 10-day-old failed `node-debugger` in `default`,
  left untouched.
- This cluster is the *initial* management cluster and the roadmap treats it as
  temporary — Hydra is meant to build the real one later. Do not invest in
  polishing it beyond what the next issue needs.
