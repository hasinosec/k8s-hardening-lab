# Kubernetes Hardening Lab — real cluster, real enforcement

A real local Kubernetes cluster (`kind`), with an insecure workload tested
against two independent admission-control layers — everything below is
actual `kubectl` output, saved in [`docs/evidence/`](docs/evidence/).

## What's real here

- A real `kind` cluster (`kubectl get nodes` → `v1.34.0`, `Ready`)
- A real insecure Deployment/Pod that **Kubernetes' own Pod Security
  Admission genuinely rejected** — its ReplicaSet retried and failed
  repeatedly, exactly as it should
- A real Kyverno install, with 5 custom policies, tested against the same
  insecure manifest **in a namespace without Pod Security Admission**, to
  isolate Kyverno's own enforcement
- A real hardened Deployment that runs, passes both layers, serves traffic,
  and was verified non-root and read-only from inside the container

## Before / after

| | `manifests/insecure/deployment.yaml` | `manifests/hardened/deployment.yaml` |
| --- | --- | --- |
| Runs as | root (`runAsUser: 0`) | uid 101 (verified: `kubectl exec ... id`) |
| Privileged | `privileged: true` | `false`, all capabilities dropped |
| Host access | `hostPath: /` mounted into the pod | none |
| Root filesystem | writable | read-only (verified: `touch` fails) |
| Image tag | `nginx:latest` | `nginx-unprivileged:1.27-alpine`, pinned |
| Resource limits | none | CPU + memory requests and limits set |
| Service account token | auto-mounted (default) | not mounted at all |
| **Result against Pod Security "restricted"** | **Rejected** — 0 pods ever created | Not applicable — passes cleanly |
| **Result against Kyverno (5 custom policies)** | **Blocked by all 5** | **Passes all 5** |

Full evidence: [`psa-rejection.txt`](docs/evidence/psa-rejection.txt),
[`kyverno-blocks-insecure.txt`](docs/evidence/kyverno-blocks-insecure.txt),
[`kyverno-allows-hardened.txt`](docs/evidence/kyverno-allows-hardened.txt),
[`hardened-pod-checks.txt`](docs/evidence/hardened-pod-checks.txt).

## Two independent enforcement layers

1. **Pod Security Admission** — built into Kubernetes, zero extra install.
   A single namespace label (`pod-security.kubernetes.io/enforce: restricted`)
   blocks privileged/root/hostPath pods at the API server.
2. **Kyverno** — a separate admission controller with 5 custom
   `ClusterPolicy` rules ([`policies/`](policies/)) covering what PSA
   doesn't: no `:latest` tags, mandatory resource limits, no `hostPath`
   anywhere in the cluster.

Both were tested against the *same* insecure manifest, independently, to
show each layer actually does its job rather than assuming one covers the
other.

## A real operational finding

Applying the policies cluster-wide initially **blocked the kube-bench job
itself** — kube-bench legitimately needs `hostPath` mounts and root to read
the node's Kubernetes config files. Real finding, not a bug: admin/diagnostic
tooling needs an explicit exception. Fixed by excluding `kube-system` from
all 5 policies (see [`docs/decisions.md`](docs/decisions.md)) rather than
weakening the policy for application workloads.

## CIS Benchmark (kube-bench)

Real run against this cluster's control plane, node, etcd, and policies:
**63 PASS / 12 FAIL / 56 WARN** (131 checks). Full output:
[`docs/evidence/kube-bench-results.txt`](docs/evidence/kube-bench-results.txt).

`kind` optimises for a fast local dev cluster, not CIS hardening, so the 12
failures are expected and real — mostly missing audit logging
(`--audit-log-path` and related args unset), `--profiling` left enabled on
the API server/controller-manager/scheduler, and file-permission checks. None
of the 12 are things this project's own manifests or policies control — they're
control-plane configuration, out of scope for the Kyverno/PSA layer above,
and would be fixed at the `kind` cluster-config or kubelet level, not in the
workload manifests.

## What's not covered here (documented honestly)

- **NetworkPolicy is not enforced** on this cluster — `kind`'s default CNI
  (`kindnet`) accepts the policy objects but doesn't enforce them. This is a
  known `kind` limitation, not a flaw in the policies. See
  [`THREAT_MODEL.md`](THREAT_MODEL.md) and [`docs/decisions.md`](docs/decisions.md).

## Reproduce it

```bash
kind create cluster --name hardening-lab
kubectl apply -f manifests/hardened/namespace.yaml
kubectl apply -f manifests/insecure/deployment.yaml   # watch it get rejected
kubectl apply -f manifests/hardened/serviceaccount.yaml
kubectl apply -f manifests/hardened/deployment.yaml   # this one runs

kubectl create -f https://github.com/kyverno/kyverno/releases/latest/download/install.yaml
kubectl apply -f policies/
kubectl apply -f manifests/kube-bench-job.yaml
```

## CI

`.github/workflows/ci.yml` runs the Kyverno CLI against both manifest sets
on every push — no cluster needed — and fails the build if the insecure
manifest is ever accepted or the hardened one is ever rejected.
