# Identity & RBAC — CapDev GitOps Integration (sandbox: vlo-ocp-capdev demo)

## Environment
- Hub (RHACM): https://api.openshift-install-aws.sandbox1479.opentlc.com:6443
- Spoke: https://api.spoke-cluster.sandbox3978.opentlc.com:6443
- ArgoCD (existing, on hub): https://openshift-gitops-server-openshift-gitops.apps.openshift-install-aws.sandbox1479.opentlc.com

## Current architecture: pull model (spoke-local ArgoCD), not hub-push
This repo originally used a hub-push model — a single ArgoCD on the hub,
pushing to the spoke via `GitOpsCluster`/cluster-proxy. That was fully
validated end-to-end, but **retired** in favor of the pull model in
`argocd-per-spoke-prototype/`, because Globe's actual production hub runs
ACM 2.11, and the push model's registration mechanism turned out to have
version-specific quirks (a manifest bug, a missing ArgoCD bind RBAC grant,
and a `destination.server` vs `destination.name` mismatch — all fixed
during testing, but all specific to this ACM build's behavior). The pull
model uses only `Policy`/`Placement`/`PlacementBinding` + a plain OLM
`Subscription` — primitives available well before 2.11 — to install a real
OpenShift GitOps Operator directly on the spoke and have it sync its own
Applications locally, independent of the hub. See
`argocd-per-spoke-prototype/README.md` for the full mechanism and how it
works.

The hub-push Applications, `GitOpsCluster`/`Placement`/
`ManagedClusterSetBinding` registration, and the ArgoCD-bind RBAC have been
deleted from both git and the live clusters. This repo's git history still
has them if needed for reference.

## What's in this folder
- `hub/rbac/` — Groups, ACM-governance-scoped ClusterRole, bindings (view
  for all, ACM edit for platform-eng). **Still live on the hub** — real
  team RBAC, not push-model-specific — but no longer synced by anything
  automatically; it was previously kept in sync by the now-retired
  `hub-bootstrap` Application. A pull-model equivalent (Policy-managed,
  matching the pattern in `argocd-per-spoke-prototype/`) would need to be
  built to restore GitOps management of this content.
- `spoke-capdev/rbac/` — Groups, NFV network-edit ClusterRole, cluster-wide
  view bindings, namespace-scoped edit/admin bindings. **Still live on the
  spoke**, same caveat as above.
- `spoke-capdev/namespaces/` — `capdev-workloads` and `capdev-lab`
  namespaces + quotas. **Still live on the spoke**, same caveat.
- `argocd-per-spoke-prototype/` — the active, validated pull-model
  mechanism. Currently proves itself against a throwaway nginx demo, not
  yet extended to manage the RBAC/namespace content above.

## What is intentionally NOT in this folder
**The HTPasswd secret itself.** Even hashed, it should not be committed to git. Provision it out-of-band:

```
htpasswd -c -B -b users.htpasswd shienneth.concepcion '<password>'
htpasswd -B -b users.htpasswd jrtorres '<password>'
# ...repeat for all 8 users...

oc create secret generic htpass-secret \
  --from-file=htpasswd=users.htpasswd \
  -n openshift-config
```
(Already provisioned on both hub and spoke as of this handoff — see
project history for validation steps. Real password values were never
committed to git.)

## Known interim state (flag, don't hide)
1. **Groups are duplicated** across hub and spoke-capdev because identity is still per-cluster HTPasswd. This resolves once the hub-federated identity broker (RHSSO/Keycloak, Option B) is live and sourcing group membership from LDAP.
2. **Namespace names in `spoke-capdev/rbac/04-rolebindings.yaml`** (openshift-gitops, openshift-logging, openshift-compliance) assume Tier 1/2/3 operators land in their default install namespaces — confirm and adjust once those operators are actually installed (Sprint 2+). Two of these bindings are currently commented out for exactly this reason (see the file).
3. **capdev-nfv-network-edit** only covers `NetworkPolicy` and Multus `network-attachment-definitions` today. Extend it, don't grant cluster-admin, if SR-IOV or other CNI CRDs are added later.
4. **RBAC/namespace content is orphaned from automation** (see above) — real on both clusters, but not currently reconciled by any Policy or Application. Next step is extending the pull-model pattern to manage it, the same way `nginx-demo-local` proves the mechanism works for a workload.
