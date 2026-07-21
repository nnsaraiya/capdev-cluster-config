# Identity & RBAC — Sprint 1 rollout notes (sandbox: vlo-ocp-capdev demo)

## Environment
- Hub (RHACM): https://api.openshift-install-aws.sandbox1479.opentlc.com:6443
- Spoke: https://api.spoke-cluster.sandbox3978.opentlc.com:6443
- ArgoCD (existing, on hub): https://openshift-gitops-server-openshift-gitops.apps.openshift-install-aws.sandbox1479.opentlc.com
- GitHub repos (create these three): `capdev-cluster-config`, `capdev-business-apps`, `capdev-shared-bases`

## What's in this folder
- `hub/rbac/` — Groups, ACM-governance-scoped ClusterRole, bindings (view for all, ACM edit for platform-eng).
- `hub/gitops-integration/` — ManagedClusterSetBinding, Placement, and GitOpsCluster CRs that auto-register the spoke as an ArgoCD-managed cluster.
- `hub/bootstrap/` — the two manually-applied root Applications (hub and spoke-capdev).
- `spoke-capdev/rbac/` — Groups, NFV network-edit ClusterRole, cluster-wide view bindings, namespace-scoped edit/admin bindings.
- `spoke-capdev/namespaces/` — capdev-workloads and capdev-lab namespaces + quotas (sync-wave -1 so they land before RoleBindings that reference them).

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

## Rollout order (this sandbox)
1. Push this folder's contents into the `capdev-cluster-config` GitHub repo (you push it — not me, per your git-authorship preference).
2. On the **hub**, manually apply the hub root Application:
   ```
   oc apply -f hub/bootstrap/hub-app-of-apps.yaml
   ```
   This syncs `hub/rbac/` and `hub/gitops-integration/` — including the GitOpsCluster trio.
3. **Confirm the spoke ManagedCluster name** before the Placement can match it:
   ```
   oc get managedclusters
   oc get managedclustersets
   ```
   If the name isn't `spoke-cluster`, edit `hub/gitops-integration/02-placement.yaml` accordingly (ArgoCD will re-sync the fix automatically once pushed).
4. **Verify the spoke got auto-registered with ArgoCD**:
   ```
   oc get gitopscluster -n openshift-gitops
   oc get secret -n openshift-gitops -l argocd.argoproj.io/secret-type=cluster
   argocd cluster list
   ```
5. Only once step 4 shows the spoke's secret, manually apply the spoke root Application:
   ```
   oc apply -f hub/bootstrap/spoke-capdev-app-of-apps.yaml
   ```
   This syncs `spoke-capdev/namespaces/` then `spoke-capdev/rbac/` (namespaces first, via sync-wave).
6. Provision the HTPasswd secret on both hub and spoke-capdev (out-of-band, per above).
7. Validate: each team member should be able to `oc whoami` successfully and see `view` access; spot-check that capdev-ops cannot edit resources in `openshift-gitops`.

## Known interim state (flag, don't hide)
1. **Groups are duplicated** across hub and spoke-capdev because identity is still per-cluster HTPasswd. This resolves once the hub-federated identity broker (RHSSO/Keycloak, Option B) is live and sourcing group membership from LDAP.
2. **Namespace names in `spoke-capdev/rbac/04-rolebindings.yaml`** (openshift-gitops, openshift-logging, openshift-compliance) assume Tier 1/2/3 operators land in their default install namespaces — confirm and adjust once those operators are actually installed (Sprint 2+).
3. **capdev-nfv-network-edit** only covers `NetworkPolicy` and Multus `network-attachment-definitions` today. Extend it, don't grant cluster-admin, if SR-IOV or other CNI CRDs are added later.
4. **GitOpsCluster assumptions** (ManagedClusterSet `default`, ManagedCluster name `spoke-cluster`) need confirming per step 3 above — this is sandbox-specific and will very likely differ in the actual Globe environment once real spokes are imported there.

