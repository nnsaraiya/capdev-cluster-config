# Implementation Guide: ArgoCD-per-Spoke (Pull Model) for Globe

## What this gives you
A hub-orchestrated, ACM Policy-based mechanism that installs a real,
independent OpenShift GitOps Operator on each spoke and keeps it (and a
demo Application) self-healing — proven live in the sandbox, including
recovery from an operator deletion via the OpenShift UI. Built entirely
on primitives available since before ACM 2.11 (no dependency on the newer
`gitops-addon` ClusterManagementAddOn, which was introduced in ACM 2.13
and does not exist that early).

This guide covers the GitOps mechanism only. RBAC/identity rollout is a
separate track — see `README.md` for what's currently in `hub/rbac/`,
`spoke-capdev/rbac/`, and `spoke-capdev/namespaces/`, none of which is
wired into automation yet.

## Phase 0 — Verify prerequisites
- [ ] ACM 2.11 installed, `MultiClusterHub` healthy
- [ ] Spoke(s) imported as `ManagedCluster`, joined + available
- [ ] Real `ManagedClusterSet` name confirmed (`oc get managedclustersets`) — don't assume `default`
- [ ] OLM catalog reachability confirmed from each spoke — note the real catalog source name if disconnected/mirrored, needed in Phase 3

## Phase 1 — Get this repo into Globe's GitLab
- [ ] Create the `capdev-cluster-config` repo in GitLab
- [ ] Push the content
- [ ] Generate one SSH keypair: `ssh-keygen -t ed25519 -f capdev-deploy-key -N ""`
- [ ] Add the **public** half as a GitLab Deploy Key (read-only) on the repo

## Phase 2 — Label every spoke
```
oc label managedcluster <spoke-name> capdev.residency/role=spoke
```
This is what `argocd-per-spoke-prototype/01-placement.yaml` matches on.
No label, no Policies apply anywhere. Any future spoke just needs this
same label — no manifest edits required to onboard it.

## Phase 3 — Replace the two `TODO(globe)` repoURLs
Currently pointing at the sandbox's real public GitHub repos (intentional,
so this runs live there). **They point at two different repos** — replace
each with the matching GitLab **SSH** URL:

| File | Points at |
|---|---|
| `bootstrap/argocd-per-spoke-bootstrap.yaml` | `capdev-cluster-config` (the Policies) |
| `argocd-per-spoke-prototype/03-policy-bootstrap-local-argocd.yaml` | `capdev-business-apps` (the app manifests) |

SSH form (`git@<gitlab-host>:<group>/<repo>.git`), not `https://` — auth
is the deploy key, not a token. Commit and push.

If Phase 0 found a disconnected/mirrored catalog, also update
`source`/`sourceNamespace` in `argocd-per-spoke-prototype/02-policy-install-gitops-operator.yaml`
to point at Globe's mirror instead of `redhat-operators`/`openshift-marketplace`.

## Phase 4 — Repository-credential Secret, in **two places, for different repos**
Private GitLab means no ArgoCD instance can clone by default. Each cluster
needs a Secret for the repo *it* must clone:

| Cluster | Needs creds for | Why |
|---|---|---|
| **Hub** | `capdev-cluster-config` | to clone the bootstrap Application's source |
| **Every spoke** | `capdev-business-apps` | to clone the app manifests its local Application syncs |

One deploy keypair can cover both repos — add its public half as a Deploy
Key on each. Same Secret shape in both places, `openshift-gitops`
namespace, with `url` set to whichever repo that cluster clones:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: capdev-repo-creds
  namespace: openshift-gitops
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  type: git
  url: git@<gitlab-host>:<group>/<repo>.git   # per table above
  sshPrivateKey: |
    -----BEGIN OPENSSH PRIVATE KEY-----
    ... never commit the real key ...
    -----END OPENSSH PRIVATE KEY-----
type: Opaque
```
Never commit the private key. Out-of-band, per cluster, same treatment as
any other credential in this project.

**This is the one part of the design that does not scale automatically.**
Everything else reaches a new spoke the moment it's labeled; this Secret
must be provisioned by hand on every new spoke, indefinitely, until a real
secrets-management layer (the deferred PAM/ExternalSecrets work) exists.

## Phase 5 — Apply the one manual step
```
oc apply -f bootstrap/argocd-per-spoke-bootstrap.yaml
```
This is the entire manual footprint. Everything downstream — the
`capdev-policies` namespace, the RBAC that lets the hub's ArgoCD manage
Policy/Placement objects, the `Placement`, and both Policies — is created
by this one Application. Same "ArgoCD is the sole manual install"
principle used throughout this project.

## Phase 6 — Verify
```
# On the hub
oc get application argocd-per-spoke-bootstrap -n openshift-gitops   # Synced/Healthy
oc get policy -n capdev-policies                                     # both Compliant

# On each spoke
oc get csv -n openshift-operators                                    # gitops operator Succeeded
oc get argocd openshift-gitops -n openshift-gitops -o yaml            # status.phase: Available
oc get route -n openshift-gitops                                     # openshift-gitops-server present
oc get application nginx-demo -n openshift-gitops                     # Synced/Healthy
oc get pods,route -n nginx-demo                                       # the app itself, running
```

---

## Gotchas — all found by breaking things live in the sandbox, not theoretical

1. **ArgoCD's default RBAC is narrow** (both hub and spoke instances,
   confirmed on OpenShift GitOps Operator 1.21.1) — read-everything,
   write only to a curated allow-list of API groups
   (`operators.coreos.com`, `config.openshift.io`, `user.openshift.io`,
   etc.), not core/`apps`/`route.openshift.io`. Any new namespace or
   resource type an ArgoCD instance needs to manage requires an explicit
   `RoleBinding`/`ClusterRole` grant first, or sync fails outright with
   `is forbidden`.

2. **`ManagedClusterSetBinding` needs a separate admission-webhook grant**
   (`managedclustersets/bind`) beyond normal RBAC verbs — ACM enforces
   this as its own authorization check regardless of which identity is
   doing the binding. Already handled in `00-namespace-and-binding.yaml`.

3. **Policy `musthave` only covers exactly what's declared, nothing
   implied.** Deleting the `Subscription` self-heals (near-instantly,
   watch-based, not slow polling) because that object is explicitly
   `musthave`. Deleting just the CSV does **not** self-heal on its own —
   the Policy never checks the CSV, only the Subscription, so it stays
   `Compliant` even with the operator's controller pod gone. Only found
   this by testing both deletion paths separately.

4. **Deleting the `ArgoCD` custom resource itself used to not self-heal
   at all.** The operator only auto-creates its default instance once, on
   first-ever install — reinstalling the operator does not recreate an
   instance it believes already existed. Confirmed live via an actual
   incident (operator deleted through the OpenShift console). Fixed by
   adding the `ArgoCD` resource as a second `musthave` object-template in
   `install-gitops-operator`.

5. **A bare `spec: {}` recreation is not equivalent to the operator's own
   defaults.** Specifically, `spec.server.route.enabled` defaults to
   `false` in the raw CRD schema, but the operator's own first-time
   bootstrap logic sets it `true`. Missing this silently drops the ArgoCD
   server's own external reachability (the Application still syncs fine
   — only the ArgoCD UI/API route disappears). The Policy template now
   sets this explicitly, and it's proven to work: recreating the `ArgoCD`
   resource a second time after the fix restored the route with zero
   manual intervention.

6. **Keep `argocd-per-spoke-prototype/` free of workload manifests.**
   Everything in that folder is applied directly to the **hub** by the
   bootstrap Application's recursive source. A copy of the demo app once
   lived there and had to be excluded via `directory.exclude`, because
   without it the Application tried to deploy the app's
   Deployment/Service/Route onto the hub. The app now lives in
   `capdev-business-apps` where it belongs, so no exclude is needed —
   but adding any workload back into that folder would reintroduce the
   same bug.

   **Onboarding a new business app therefore takes two edits, not one:**
   its manifests go in `capdev-business-apps`, *and* its namespace +
   RoleBinding + Application entries go into Policy 03 (see gotcha 1 —
   the local ArgoCD can't deploy into a namespace it has no RoleBinding
   for).

7. **Automated sync re-enforces whatever is actually committed in git**,
   including a broken placeholder URL. Don't enable the automated
   bootstrap Application before Phase 3's `TODO(globe)` swap is done, or
   the affected Application resets to `Unknown` sync status on every
   reconcile (though already-deployed resources aren't pruned, since no
   new sync ever completes to act on them).

## Business app onboarding — the established pattern
`capdev-business-apps` is the real home for workload manifests, and
`nginx-demo` (in `spoke-capdev/nginx-demo/`) is the working reference. To
onboard another app:
1. Add its manifests under `capdev-business-apps/spoke-capdev/<app>/`.
2. Add three entries to `03-policy-bootstrap-local-argocd.yaml`: its
   target `Namespace`, an `argocd-app-controller-admin` `RoleBinding` in
   that namespace, and an `Application` pointing at its path.
3. Commit both repos and push — the hub's bootstrap Application picks up
   the Policy change and every matched spoke converges automatically.

## What this does NOT cover yet
- RBAC/identity content (`hub/rbac/`, `spoke-capdev/rbac/`,
  `spoke-capdev/namespaces/`) is real and already matches the actual
  team, but nothing currently syncs it automatically. Extending Policy 03
  (or a sibling Policy) to manage it is separate, deliberately deferred
  work — including deciding whether business apps should eventually live
  in the RBAC-managed `capdev-workloads` namespace rather than their own.
- An app-of-apps / ApplicationSet layer, if per-app Policy entries ever
  become too repetitive at scale.
- Hub-federated identity broker (RHSSO/Keycloak + real LDAP) — still
  Sprint 4 per the original residency plan, independent of everything in
  this guide.
