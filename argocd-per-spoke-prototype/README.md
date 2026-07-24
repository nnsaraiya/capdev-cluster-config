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

## What the spoke's local ArgoCD actually syncs
`03-policy-bootstrap-local-argocd.yaml` points the spoke's local ArgoCD at
**`capdev-business-apps`** (`spoke-capdev/nginx-demo`) — the real,
canonical home for business-app workloads. `nginx-demo` is a genuine first
onboarded app and the reference pattern for future ones, not throwaway
scaffolding. It deploys into its own dedicated `nginx-demo` namespace.

It deliberately does **not** point at this repo's `hub/rbac/` or
`spoke-capdev/` content: those describe RBAC/namespace state that is still
live on both clusters but not currently reconciled by any automation.
Pointing a second ArgoCD at them would risk two controllers trying to own
and prune the same objects — real ownership flapping, not a style concern.
Bringing that content under the pull model is separate, deliberately
deferred work (see the top-level README.md).

(History, for anyone reading old commits: this previously pointed at a
`nginx-demo-local/` copy of the app kept inside this folder, purely to
prove the mechanism in isolation before `capdev-business-apps` was wired
up. An even earlier iteration used a bare ConfigMap. Both are gone.)

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
  of the hub" actually works. It also creates the app's target namespace
  and the RoleBinding the local ArgoCD needs to deploy into it.

**Note:** this folder must stay workload-free — everything in it is
applied directly to the **hub** by `bootstrap/argocd-per-spoke-bootstrap.yaml`.
The actual business-app manifests live in `capdev-business-apps`
(`spoke-capdev/nginx-demo/`) and only ever reach the spoke through the
Application that Policy 03 creates there.

## Applying this: two options

**Option A — automated (recommended once the repo exists on Globe's GitLab)**:
apply the one Application in `../bootstrap/argocd-per-spoke-bootstrap.yaml`
on the hub. It owns this whole folder from then on — sync-wave annotations
on the individual objects (-2 namespace/binding, -1 placement, 0 policies)
handle ordering, and any future edit to these files just needs a git push,
not a re-apply. This is the same "ArgoCD is the sole manual install"
principle as every other root Application in this project — one manual
apply, not four.
```
oc apply -f bootstrap/argocd-per-spoke-bootstrap.yaml
```

**Option B — fully manual** (useful for a first dry run before the repo
exists anywhere, or for debugging step by step): apply in order, on the
**hub**:
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
`route.openshift.io`. Confirmed live: the demo app's first sync attempt
failed with `Service/Deployment/Route ... is forbidden`.
`03-policy-bootstrap-local-argocd.yaml` now also creates a `RoleBinding`
(admin, scoped to the app's own `nginx-demo` namespace only) for the
ArgoCD app-controller SA before the Application's first sync. This is a
real, generalizable finding and it applies directly to app onboarding:
**every** business app's target namespace needs an equivalent
per-namespace RoleBinding before the spoke's local ArgoCD can deploy into
it. Onboarding a new app therefore means adding its namespace +
RoleBinding + Application to Policy 03, not just its manifests to
`capdev-business-apps`.

## Known finding: the hub's ArgoCD has the same narrow default (Option A only)
The **same** narrow-allow-list behavior above applies to the hub's
existing ArgoCD, not just the spoke's — confirmed live when first testing
`bootstrap/argocd-per-spoke-bootstrap.yaml`: it couldn't create
`ManagedClusterSetBinding`/`Placement`/`Policy`/`PlacementBinding`
(`is forbidden`), and `ManagedClusterSetBinding` additionally needs the
`managedclustersets/bind` admission-webhook grant — the same one the old
push-model setup needed, which got deleted along with it during cleanup.
`00-namespace-and-binding.yaml` now grants both. This RBAC is **only**
needed for Option A; Option B (manual `oc apply` as `kube:admin`) already
has all of it.

## Known finding: automated sync re-enforces whatever is committed
Confirmed live: with Option A applied, `03-policy-bootstrap-local-argocd.yaml`'s
`ConfigurationPolicy` re-enforces the nested Application's `repoURL` from
whatever is actually committed in git. Both `repoURL`s in this folder are
currently set to the sandbox's real public GitHub repo (marked
`TODO(globe)`) specifically so this runs live for real here — if either
were ever left as an unreachable placeholder instead, the affected
Application would reset to `Unknown` sync status on every reconcile
(though already-deployed resources wouldn't be pruned, since ArgoCD never
completes a new sync to act on). Worth remembering when the `TODO(globe)`
swap happens: an unreachable URL breaks sync silently-ish, it doesn't
just get ignored.

## Before applying this to Globe
1. **Label every spoke `ManagedCluster`** with `capdev.residency/role=spoke`
   on the hub (see `01-placement.yaml`) — without this, the Placement
   matches nothing and none of these Policies will apply anywhere.
2. **Replace both `TODO(globe)` `repoURL`s — they point at two _different_
   repos.** `../bootstrap/argocd-per-spoke-bootstrap.yaml` →
   `capdev-cluster-config`; `03-policy-bootstrap-local-argocd.yaml` →
   `capdev-business-apps` (that's where the app manifests live). Both in
   SSH form (`git@<gitlab-host>:<group>/<repo>.git`, not `https://...`),
   since auth there is an SSH deploy key, not a token.
3. **GitLab is private, auth is an SSH deploy key** — no ArgoCD instance
   has credentials configured by default (unlike this sandbox's public
   GitHub repos), so each one needs a repository-credential `Secret`
   before it can clone anything. **Two clusters, and they need creds for
   different repos:** the **hub's** existing ArgoCD needs
   `capdev-cluster-config` (to clone the bootstrap Application's source),
   and the **spoke's** local ArgoCD needs `capdev-business-apps` (to clone
   the app manifests the `nginx-demo` Application points at). One deploy
   keypair can cover both repos if you add its public half to each.
   Same Secret shape in both places, `openshift-gitops` namespace on
   whichever cluster — set `url` to the repo that cluster must clone:
   ```yaml
   apiVersion: v1
   kind: Secret
   metadata:
     name: capdev-cluster-config-repo-creds
     namespace: openshift-gitops
     labels:
       argocd.argoproj.io/secret-type: repository
   stringData:
     type: git
     url: git@<gitlab-host>:<group>/capdev-cluster-config.git
     sshPrivateKey: |
       -----BEGIN OPENSSH PRIVATE KEY-----
       ... never commit the real key ...
       -----END OPENSSH PRIVATE KEY-----
   type: Opaque
   ```
   Generate the keypair once (`ssh-keygen -t ed25519 -f capdev-deploy-key -N ""`),
   add the **public** half as a GitLab "Deploy Key" on the repo (read-only
   is enough), and only the **private** half goes in the Secret above —
   out-of-band, per spoke, never committed to git, same treatment as the
   HTPasswd secret. One keypair can be reused across every spoke (it's
   read-only access to one repo, not a sensitive per-cluster credential),
   which simplifies distribution: generate it once, deliver the same
   private key to every spoke via the same Policy mechanism (a `musthave`
   object-template bound to the same Placement) rather than one per spoke.
   Also add `capdev-business-apps` and `capdev-shared-bases` as Deploy Keys
   on the same keypair if this ArgoCD instance needs to read those too —
   or generate a separate keypair per repo if you'd rather keep read access
   scoped tightly.
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
