# Kubernetes Hardening Lab — Threat Model

Scope: a single-tenant `demo` namespace in a local `kind` cluster, and the
admission controls that gate what can run in it.

## Assets

- The Kubernetes API server and its credentials
- Whatever a compromised pod's container could reach (host, other pods, the
  API, other namespaces)
- The node itself, if a container escapes it

## Threats and mitigations

| Threat | Insecure baseline | Mitigation | Evidence |
|---|---|---|---|
| Container escape to the host | `privileged: true`, `hostPath: /` mount | Pod Security Admission `restricted` on the namespace rejects both outright | `docs/evidence/psa-rejection.txt` |
| Compromised container runs as root on the node | `runAsUser: 0` | `runAsNonRoot: true`, dedicated uid 101 | `docs/evidence/hardened-pod-checks.txt` — `id` returns uid 101 |
| Attacker writes a backdoor/webshell into the running container | writable root filesystem | `readOnlyRootFilesystem: true` | same file — `touch` fails with "Read-only file system" |
| Privilege escalation inside the container | `allowPrivilegeEscalation: true`, all Linux capabilities available | `allowPrivilegeEscalation: false`, `capabilities.drop: [ALL]` | enforced by Kyverno + PSA |
| Stolen pod credentials used against the API | default service account token auto-mounted | `automountServiceAccountToken: false`, dedicated minimal service account | `manifests/hardened/serviceaccount.yaml` |
| Unpinned image swapped for a malicious one at pull time | `:latest` tag | `disallow-latest-tag` Kyverno policy | `policies/disallow-latest-tag.yaml` |
| Resource exhaustion / noisy-neighbour DoS | no requests/limits | `require-resource-limits` Kyverno policy | `policies/require-resource-limits.yaml` |
| Lateral movement between pods/namespaces | no network segmentation | Default-deny NetworkPolicy + explicit allows | `manifests/hardened/networkpolicy.yaml` — **see accepted risk below** |

## Two enforcement layers, deliberately

1. **Pod Security Admission** (built into Kubernetes, zero extra install) —
   blocks privileged/root/hostPath pods outright.
2. **Kyverno** (installed separately) — adds checks PSA doesn't express:
   pinned image tags, mandatory resource limits.

## Accepted risk / known limitation

- **NetworkPolicy is not enforced on this cluster.** `kind`'s default CNI
  (`kindnet`) accepts NetworkPolicy objects via the API but does not enforce
  them — this is a documented kind limitation, not a bug in the policies.
  Real enforcement needs a NetworkPolicy-capable CNI (Calico, Cilium) or a
  managed cluster (EKS/GKE/AKS). See `docs/decisions.md`.
- Single-node cluster — no node-to-node lateral-movement scenario to test.

## Next

- Re-run this lab on a Calico-enabled cluster (or EKS) to get real
  NetworkPolicy enforcement evidence.
- Add image signature verification (`verifyImages` in Kyverno) once the
  `container-security` and CI/CD pipeline projects produce signed images.
