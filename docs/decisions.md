# Decisions

## Why kind, and what it can't show

`kind` runs a real Kubernetes control plane and node inside Docker — free,
local, no cloud account. But its default CNI (`kindnet`) **does not enforce
NetworkPolicy**. The policies in `manifests/hardened/networkpolicy.yaml` are
valid and accepted by the API server (`kubectl get networkpolicy` shows them),
but traffic isn't actually being blocked by them on this cluster. Installing
Calico to get real enforcement was the alternative; skipped here to keep the
lab fast to reproduce. A managed cluster (EKS/GKE/AKS) or a Calico/Cilium CNI
enforces these for real — noted honestly rather than staged to look like it
works.

## Why Pod Security Admission before Kyverno

Kubernetes ships **Pod Security Admission** (PSA) as a built-in control —
label a namespace `pod-security.kubernetes.io/enforce: restricted` and the
API server itself rejects privileged, root, or otherwise unsafe pods. No
extra install. It's the first line of defense here, and it's what actually
stopped `manifests/insecure/deployment.yaml`'s ReplicaSet from ever creating
a pod (see `docs/evidence/psa-rejection.txt` — it retried and failed
repeatedly, exactly as it should).

Kyverno adds what PSA can't express: no `:latest` tags, mandatory resource
requests/limits, no `hostPath` volumes. Two layers, each doing what it's
better at.

## Why `nginxinc/nginx-unprivileged`

Stock `nginx` binds port 80 and expects to run as root to do it. Rather than
fight that with extra securityContext tricks, the hardened deployment uses an
image built to run unprivileged on port 8080 from the start — the simpler,
more honest fix.

## Why `automountServiceAccountToken: false`

By default every pod gets a token that can talk to the Kubernetes API. This
demo app never calls the API, so there's no reason it should be able to.
Removing the mount removes a credential an attacker could steal if the
container were ever compromised.

## Why the policies exclude `kube-system`

Applying them cluster-wide at first **blocked the kube-bench job itself** —
kube-bench legitimately needs `hostPath` mounts and root to read the node's
config files. Real operational finding: admin/diagnostic tooling needs an
explicit exception, made narrowly (only `kube-system`, not a blanket
bypass) rather than weakening the policy for application workloads.

## `ClusterPolicy` is a legacy API in this Kyverno version

`kubectl apply -f policies/` prints a deprecation warning — Kyverno is
migrating to a CEL-based `ValidatingPolicy` API. Left on `ClusterPolicy` here
since it's still fully supported and the pattern-based syntax is more
readable for a first Kyverno project; noted as a real thing to revisit.

## The kube-bench numbers are real, and the 12 failures are expected

`kind` is built for a fast local dev loop, not CIS compliance — it doesn't
enable API server audit logging or turn off `--profiling` by default. Those
12 failures are control-plane configuration, not something this project's
manifests or Kyverno policies can fix; a managed cluster (EKS/GKE/AKS)
handles most of section 1–4 of the CIS benchmark for you.
