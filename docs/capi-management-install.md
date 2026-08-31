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
ASSET=clusterctl-linux-amd64
mkdir -p ~/bin
cd "$(mktemp -d)"

curl -fsSLO "https://github.com/kubernetes-sigs/cluster-api/releases/download/${V}/${ASSET}"

# Verify before installing. The digest lives in the release asset's API
# metadata, NOT in a separate .sha256 file -- see the note below.
WANT=$(curl -fsSL "https://api.github.com/repos/kubernetes-sigs/cluster-api/releases/tags/${V}" \
  | python3 -c "import json,sys;print(next(a['digest'] for a in json.load(sys.stdin)['assets'] if a['name']=='${ASSET}'))")
GOT="sha256:$(sha256sum "${ASSET}" | cut -d' ' -f1)"
[ "${WANT}" = "${GOT}" ] || { echo "DIGEST MISMATCH: want ${WANT}, got ${GOT}" >&2; exit 1; }

install -m 0755 "${ASSET}" ~/bin/clusterctl
~/bin/clusterctl version
```

> **The release uploads no `.sha256` file, but it does publish a digest.** It is
> the `digest` field on the asset in the releases API, and looking for a
> checksum *file*, not finding one, and concluding that no digest exists is the
> mistake to avoid — it throws away the only cryptographic check available. A
> byte-size match is not a substitute: it cannot detect same-length corruption
> or substitution.
>
> For v1.14.0 the published value is
> `sha256:919ec7acb93ebdec9cde46727a1ddb8810ce55dd9ceef2d416ab65b6783bb58a`,
> and the installed binary matches it.

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
# every API checked, with its stored contract version -- all eight in one loop,
# so the list here and the claimed result cannot drift apart
for k in \
  clusters.cluster.x-k8s.io \
  machines.cluster.x-k8s.io \
  machinesets.cluster.x-k8s.io \
  machinedeployments.cluster.x-k8s.io \
  machinehealthchecks.cluster.x-k8s.io \
  kubeadmconfigs.bootstrap.cluster.x-k8s.io \
  kubeadmconfigtemplates.bootstrap.cluster.x-k8s.io \
  kubeadmcontrolplanes.controlplane.cluster.x-k8s.io
do
  printf '%-52s ' "$k"
  kubectl get crd "$k" -o jsonpath='{.status.storedVersions}' 2>/dev/null || printf 'MISSING'
  echo
done

# controller deployments: READY and AVAILABLE are the fields to read
kubectl get deploy -A | grep -E 'NAME|capi-|cert-manager'

# pod phase is a separate question from deployment readiness
kubectl get pods -A | grep -E 'capi-|cert-manager'

# what clusterctl believes it installed
kubectl get providers -A
```

Result on 2026-08-31:

- all eight APIs above present, each with `storedVersions` `["v1beta2"]`
- all six deployments (3 CAPI + 3 cert-manager) `READY 1/1` with `AVAILABLE 1`,
  and all six pods in phase `Running`
- both nodes still `Ready` at v1.35.3
- `kubectl get providers -A` reports `cluster-api`, `bootstrap-kubeadm` and
  `control-plane-kubeadm`, all `v1.14.0`

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
