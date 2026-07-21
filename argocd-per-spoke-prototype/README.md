# ArgoCD-per-spoke prototype (Policy-based, ACM 2.11-compatible)

## Why this exists, and why it's separate from everything else in this repo
Globe's production hub runs ACM 2.11. This sandbox runs ACM 2.17, which
ships a native `gitops-addon` ClusterManagementAddOn that can install and
manage OpenShift GitOps on a spoke with almost no custom work — but per
Red Hat's 2.13 release notes, that addon was introduced in ACM 2.13 as
Technology Preview. It almost certainly does not exist on Globe's 2.11 hub.
Building on it here would be exactly the "prove it in sandbox, port over
unexamined" mistake this project has been explicitly trying to avoid.

So this prototype deliberately does NOT use the native addon. It uses only
ACM Policy + Placement + PlacementBinding — primitives that have existed
since well before 2.11 — to (1) install the real OpenShift GitOps Operator
on the spoke via a Subscription, and (2) bootstrap that spoke's own local
ArgoCD with an Application, entirely independent of the hub's ArgoCD.
Whatever works here should also work on Globe's actual hub.

## Why it's not wired into the existing bootstrap Applications
`hub-bootstrap` and `spoke-capdev-bootstrap` (the two existing, proven,
manually-applied Applications) already own the live RBAC/namespace/
GitOpsCluster resources on the hub and spoke. If this prototype's spoke-
local ArgoCD were pointed at that same content, two independent ArgoCD
instances would both try to own and prune the same live objects — a real
risk of ownership flapping or unexpected deletion, not just a style
concern. Instead, `03-policy-bootstrap-local-argocd.yaml` points the
spoke's new local ArgoCD at `test-payload/` in *this* repo path — a
throwaway ConfigMap that nothing else touches — purely to prove the
mechanism (operator installs → local ArgoCD comes up → Policy creates a
local Application → that Application syncs from git, entirely on its own,
independent of the hub) without touching anything already relied upon.

## What's here
- `00-namespace-and-binding.yaml` — dedicated `capdev-policies` namespace
  + ManagedClusterSetBinding (Policy/Placement/PlacementBinding must live
  in the same namespace, and that namespace needs its own binding into the
  `default` ManagedClusterSet — same requirement as
  `hub/gitops-integration/01-managedclustersetbinding.yaml`, just scoped to
  a different namespace so this prototype doesn't share objects with the
  real GitOps integration).
- `01-placement.yaml` — matches `spoke-cluster`, mirroring
  `hub/gitops-integration/02-placement.yaml` (kept as a separate object
  rather than reusing that one, for the same isolation reason above).
- `02-policy-install-gitops-operator.yaml` — Policy + PlacementBinding
  that installs the OpenShift GitOps Operator via Subscription in
  `openshift-operators` (the standard, documented install path — no addon
  framework involved). Once OLM reconciles this, the operator auto-creates
  a default ArgoCD instance in a new `openshift-gitops` namespace *on the
  spoke itself*.
- `03-policy-bootstrap-local-argocd.yaml` — Policy + PlacementBinding that
  creates an `Application` object directly in the spoke's own (new, local)
  `openshift-gitops` namespace, `destination.server:
  https://kubernetes.default.svc` — local to the spoke, not the hub. This
  is what proves "Applications synced directly on the spoke, independent
  of the hub" actually works.
- `test-payload/` — a single throwaway ConfigMap; the thing the local
  Application actually syncs, chosen specifically so it can't collide with
  anything the existing push-model setup owns.

## Manual apply (same "ArgoCD is the sole manual install" principle)
Apply in order, on the **hub**:
```
oc apply -f argocd-per-spoke-prototype/00-namespace-and-binding.yaml
oc apply -f argocd-per-spoke-prototype/01-placement.yaml
oc apply -f argocd-per-spoke-prototype/02-policy-install-gitops-operator.yaml
```
Wait for the GitOps Operator to actually install on the spoke and its
default ArgoCD to come up (check `oc get csv -n openshift-operators` and
`oc get argocd -n openshift-gitops`, both **on the spoke**), then:
```
oc apply -f argocd-per-spoke-prototype/03-policy-bootstrap-local-argocd.yaml
```

## What this does NOT decide yet
This proves the *mechanism*. It does not migrate `capdev-cluster-config`
or `capdev-business-apps` off the current hub-push model — that's a
separate decision once this pattern is validated, given it changes where
every Application actually lives and syncs.
