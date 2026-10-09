# amphibian — plan

Run native Kubernetes pods on Slurm worker nodes, side by side with Slurm jobs,
without either scheduler double-booking the node.

## 1. Goal

- Slurm worker nodes are **also** k3s agent nodes ("hybrid nodes"). Pods run
  natively under kubelet/containerd, not wrapped in a Slurm job or Apptainer.
- Slurm and Kubernetes share each hybrid node's CPU and memory. Slurm counts
  everything a pod uses as allocated. Kubernetes never places a pod on
  resources Slurm has already given out.
- **Scope v0:** CPU + memory only. Ephemeral workloads only (Jobs, bare pods).
- **Out of scope for now:** GPUs/MIG, long-running services, preemption,
  multiple clusters.

## 2. Core idea: Slurm is the single source of truth

For each pod meant for a hybrid node, amphibian submits a **placeholder Slurm
job** (a "shadow job") that requests the pod's CPU and memory. Slurm decides
*whether* and *where* it runs.

```
pod created ──► shadow job submitted ──► Slurm allocates node N
                                              │
pod bound to node N ◄─────────────────────────┘
pod finishes/deleted ──► shadow job cancelled
shadow job killed (time limit, scancel, drain) ──► pod evicted/failed
```

Because one scheduler (Slurm) decides for all resources on these nodes, both
sides stay in sync: Kubernetes is only told the result. Slurm partitions,
accounts, QOS and fair-share apply to Kubernetes workloads too.

## 3. Components (sketch)

- **Admission webhook.** For pods that opt in (by label, namespace or
  `schedulerName: amphibian`), it sets the scheduler name and the hybrid-node
  toleration. It checks that requests == limits (Guaranteed QoS), so the
  shadow job's size is unambiguous. It also maps namespace and annotations to a
  Slurm partition, account, QOS and time limit.
- **Bridge controller** (Go, controller-runtime). It:
  - watches pending pods with `schedulerName: amphibian`;
  - submits shadow jobs through `slurmrestd` and tracks their state;
  - creates the pod `Binding` once the job is RUNNING;
  - cleans up both sides when either one ends;
  - matches pods to jobs again after a restart. Each pod↔job link is stored
    on both sides (a pod annotation and the Slurm job name or comment).
- **Hybrid node setup.** Hybrid nodes join k3s as agents with a taint
  (`amphibian.io/slurm=true:NoSchedule`), so the default scheduler never uses
  them. Kubelet's `kube-reserved`/`system-reserved` are set to match Slurm's
  `CoreSpecCount`/`MemSpecLimit`, so both agree on what the node can offer.
- **Node status exporter (later).** It writes Slurm's view of each node onto
  the k8s Node object (allocated CPU/memory, state, drain reason) for
  visibility and debugging.

## 4. Hard problems and open questions

- **Accounting vs. enforcement.** Pod processes run in kubelet's cgroups, not
  in the shadow job's cgroup. In v0 the only enforcement is Kubernetes' own
  Guaranteed QoS limits. Later: pin pods to the CPUs Slurm gave the shadow job
  (kubelet's static CPU manager, or an NRI plugin that reads the Slurm
  allocation).
- **Shadow job shape.** A batch script that sleeps forever, or an allocation
  with no shell? How do jobs survive restarts of slurmctld and of the
  controller?
- **Startup latency.** Images are pulled only after Slurm has allocated the
  job, so that allocated time is wasted. Pre-pulling images may help.
- **Pod → Slurm request mapping.** How to turn multi-container pods, init
  containers and pod overhead into a single Slurm request.
- **Prior art to evaluate before building:** SchedMD Slinky `slurm-bridge`
  (very close to this design), Slinky `slurm-operator`, HPK, Sylabs
  `wlm-operator`, virtual-kubelet providers. Then decide whether to reuse, fork
  or build.

## 5. Test environment (agent-friendly)

**Vagrant + libvirt** on a Linux host. Plain shell scripts provision it, and
all versions are pinned.

| VM      | Role                                                                                                     | Size (initial)  |
|---------|----------------------------------------------------------------------------------------------------------|-----------------|
| `ctrl`  | k3s server, amphibian controller + webhook, slurmctld, slurmrestd (JWT), munge, optional slurmdbd. Runs no jobs. | 2 vCPU / 4 GiB  |
| `cpu-1` | Hybrid node: slurmd + k3s agent, tainted.                                                                | 4 vCPU / 8 GiB  |

Changing a VM-count variable adds more `cpu-N` nodes.

**One set of entry points (Makefile)**, all idempotent and non-interactive with
clear exit codes, so an agent can run the full loop unattended:

`make up` · `make down` · `make reset` · `make deploy` (build the controller image and load it into k3s) · `make e2e` · `make logs`

**Test layers:**

1. Unit tests (Go, fake Slurm client).
2. Integration tests: envtest + a mock `slurmrestd`.
3. End-to-end scenarios on the Vagrant cluster:
   - A pod fills the node → a Slurm job stays pending, and the reverse.
   - Under mixed load, the node is never overcommitted (compare
     `scontrol show node` with kubelet's allocated resources).
   - Deleting a pod frees its Slurm resources.
   - `scancel` on a shadow job terminates its pod.
   - After a controller restart, pods and jobs are matched up again correctly.

## 6. Major dependencies

- **k3s** (Kubernetes ≥ 1.30), with its bundled containerd
- **Slurm** ≥ 24.05: `slurmrestd` with `auth/jwt`, munge, cgroup v2
  (`task/cgroup`, `select/cons_tres`)
- **Go**, controller-runtime / kubebuilder, a slurmrestd OpenAPI client
  (generated or hand-written)
- **Vagrant** + vagrant-libvirt (QEMU/KVM), with a pinned Linux box
- **Test tooling:** `go test`, envtest, kubectl; optionally kind/k3d for quick
  controller-only iteration

## 7. Rough milestones

1. Vagrant cluster up: k3s and Slurm both healthy, `cpu-1` joined to both.
2. Manual proof: shadow job via `sbatch` plus a hand-made `Binding` works
   end to end.
3. Controller MVP: pending pod → shadow job → bind → cleanup.
4. Webhook, matching pods to jobs after a restart, e2e suite green.
5. Enforcement (CPU pinning) and the status exporter. Then GPUs/MIG.
