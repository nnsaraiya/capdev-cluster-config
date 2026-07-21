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

## Why it's not wired into the RBAC/namespace content in this repo
(`hub-bootstrap`/`spoke-capdev-bootstrap`, the two Applications that used
to own this content under the retired hub-push model, are gone now — see
the top-level README.md. The reasoning below still applies to why this
prototype uses its own throwaway payload rather than `hub/rbac`/
`spoke-capdev/`.) Two independent ArgoCD instances both trying to own and
prune the same live RBAC objects is a real risk of ownership flapping or
unexpected deletion, not just a style concern. Instead, `03-policy-bootstrap-local-argocd.yaml` points the
spoke's new local ArgoCD at `nginx-demo-local/` in *this* repo path — the
same nginx-unprivileged workload as `capdev-business-apps`'s `nginx-demo`,
deployed a second, independent way, into its own non-colliding namespace —
purely to prove the mechanism (operator installs → local ArgoCD comes up →
Policy creates a local Application → that Application syncs a real
workload from git, entirely on its own, independent of the hub) without
touching anything already relied upon. (An earlier iteration used a bare
ConfigMap for the same proof; replaced with a real workload for a more
concrete side-by-side comparison against the hub-push nginx-demo.)

## What's here
- `00-namespace-and-binding.yaml` — dedicated `capdev-policies` namespace
  + ManagedClusterSetBinding (Policy/Placement/PlacementBinding must live
  in the same namespace, and that namespace needs its own binding into the
  `default` ManagedClusterSet — same requirement as
  `hub/gitops-integration/01-managedclustersetbinding.yaml`, just scoped to
  a different namespace so this prototype doesn't share objects with the
  real GitOps integration).
- `01-placement.yaml` — matches on a label
  (`capdev.residency/role: spoke`), not one hardcoded cluster name, so any
  current or future spoke gets this automatically once labeled. Every
  spoke `ManagedCluster` needs this label applied on the hub before
  applying: `oc label managedcluster <name> capdev.residency/role=spoke`.
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
- `nginx-demo-local/` — the same nginx-unprivileged Deployment/Service/
  Route as `capdev-business-apps`'s `nginx-demo`, in its own namespace
  (`capdev-argocd-per-spoke-test`) so it can't collide with anything the
  existing push-model setup owns. This is what the local Application
  actually syncs.

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

## Known finding: the default ArgoCD instance can't deploy workloads out of the box
Red Hat's OpenShift GitOps operator (as of 1.21.1, this sandbox) grants its
default ArgoCD instance's application-controller a narrow ClusterRole by
default: read (`get`/`list`/`watch`) on everything, but *write* only to a
curated allow-list of API groups (`operators.coreos.com`,
`config.openshift.io`, `user.openshift.io`, etc.) — not core, `apps`, or
`route.openshift.io`. Confirmed live: the first sync attempt of
`nginx-demo-local/` failed with `Service/Deployment/Route ... is
forbidden`. `03-policy-bootstrap-local-argocd.yaml` now also creates a
`RoleBinding` (admin, scoped to `capdev-argocd-per-spoke-test` only) for
the ArgoCD app-controller SA before the Application's first sync. This is
a real, generalizable finding: **any** future spoke's local ArgoCD will
need an equivalent per-namespace RoleBinding before it can deploy anything
beyond the operator's own default allow-list — worth remembering if this
pattern gets adopted for real business apps later.

## Before applying this to Globe
1. **Label every spoke `ManagedCluster`** with `capdev.residency/role=spoke`
   on the hub (see `01-placement.yaml`) — without this, the Placement
   matches nothing and none of these Policies will apply anywhere.
2. **Replace `<GITLAB_REPO_URL>`** in
   `03-policy-bootstrap-local-argocd.yaml` with this repo's real GitLab
   URL once it exists.
3. **If the GitLab repo is private (assume it is)**, the spoke's local
   ArgoCD needs a repository-credential `Secret` before it can clone
   anything — it has no credentials configured by default, unlike this
   sandbox's public GitHub repos. Standard shape (HTTPS token; adjust if
   using an SSH deploy key instead):
   ```yaml
   apiVersion: v1
   kind: Secret
   metadata:
     name: capdev-cluster-config-repo-creds
     namespace: openshift-gitops
     labels:
       argocd.argoproj.io/secret-type: repository
   stringData:
     url: <GITLAB_REPO_URL>/capdev-cluster-config.git
     username: <token-username-or-placeholder>
     password: <TOKEN — never commit the real value>
   type: Opaque
   ```
   This needs to exist on **every** spoke, same "every current and future
   spoke" scaling concern as everything else here — so it's worth
   delivering the same way (a `musthave` object-template in a Policy bound
   to the same Placement), with the real token value populated out-of-band
   per spoke, never committed to git — same treatment as the HTPasswd
   secret.
4. **Confirm OLM catalog reachability** — `02-policy-install-gitops-operator.yaml`
   depends on the `redhat-operators` `CatalogSource` being reachable from
   each spoke. If Globe's spokes are disconnected/mirrored, `source`/
   `sourceNamespace` need to point at Globe's mirror instead.
5. **Confirm the real `ManagedClusterSet` name** in both
   `00-namespace-and-binding.yaml` and `01-placement.yaml` — don't assume
   `default` carries over.

## What this does NOT decide yet
This proves the *mechanism*. It does not migrate `capdev-cluster-config`
or `capdev-business-apps` off the current hub-push model — that's a
separate decision once this pattern is validated, given it changes where
every Application actually lives and syncs.
