---
title: use-manifest-annotation-for-object-removal
authors:
  - "@jottofar"
reviewers:
  - "@LalatenduMohanty"
  - "@wking"
  - "@smarterclayton"
  - "@deads2k"
approvers:
  - "@smarterclayton"
  - "@deads2k"
creation-date: 2020-08-03
last-updated: 2020-08-10
status: implementable
---

# Use Manifest Annotation For Object Removal

## Release Signoff Checklist

- [X] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [openshift-docs](https://github.com/openshift/openshift-docs/)

## Summary

Currently CVO managed object removal is handled by jobs.
Jobs have many immutable components, so they aren't a great match for CVO managed manifests.
This enhancement replaces that approach with a new manifest annotation requesting the CVO delete the in-cluster object instead of creating/updating it.
This will provide a more straightforward way for engineering to remove content.

## Motivation

There should be a straightforward way to remove CVO managed objects.

### Goals

* Developers will be able to remove any of the currently managed CVO objects by modifying an existing manifest and adding the new delete annotation.

### Non-Goals

* This enhancement does not add any new objects for management by CVO, therefore the new delete annotation only applies to the currently supported manifests and any that may be added in the future.
This enhancement neither adds nor requires any application specific delete logic within CVO.

## Proposal

When the following annotation appears in a CVO supported manifest and is set to "true" the associated object will be removed from the cluster by the CVO.
Values other than `true` will result in CVO logging an error and no further processing of the manifest.
```
apiVersion: apps/v1
...
metadata:
...
  annotations:
    release.openshift.io/delete: "true"
```
The existing CVO ordering scheme defined [here](https://github.com/openshift/cluster-version-operator/blob/master/docs/dev/operators.md#what-is-the-order-that-resources-get-createdupdated-in) will also be used for object removal.
This both minimizes CVO code changes and provides a simple method of deleting multiple objects by reversing the order in which they were created.
It is the developer's responsibility to ensure proper deletion ordering and to ensure that all items originally created by an object are deleted when that object is deleted.
For example, an operator may have to be modified, or a new operator created, to take explicit actions to remove external resources.
The modified or new operator would then be removed in a subsequent update.

### Delete Finalization

Two delete finalization options are described below to help reach a consensus on the best approach.

#### Finalization Option 1

The first approach is to handle deletion requests similar to how CVO handles create/update requests.
This is in a non-blocking manner whereby CVO issues the initial request to delete an object kicking off resource finalization and after which resource removal.
CVO does not wait for actual resource removal but instead continues.
As CVO manifest processing continues the object's `.metadata.deletionTimestamp`, set when the delete was originally requested, informs CVO that this object has already been processed for deletion.
If an object cannot be successfully removed CVO will set `Upgradeable=false` which in turn blocks cluster update to the next minor release.

The advantage of this approach is that unfinalized manifests do not block the remainder of the current manifest graph's application.
Also, since this approach is similar to current CVO manifest graph processing it will most likely be easier to implement and maintain.

#### Finalization Option 2

The second approach is to handle deletion requests synchronously whereby CVO will wait for confirmation that the given object has been removed before continuing through the manifest graph.

The advantage of this approach is that it is deterministic - CVO will block until the given object is removed or we give up and fail the upgrade.
But with this approach unfinalized manifests may block the remainder of the current manifest graph's application.

### User Stories

The user is an OpenShift developer responsible for the development and maintenance of an OpenShift component.

#### Story 1

Remove the cluster-autoscaler-operator deployment.
The existing cluster-autoscaler-operator deployment manifest 0000_50_cluster-autoscaler-operator_07_deployment.yaml is modified to contain the delete annotation:
```
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-autoscaler-operator
  namespace: openshift-machine-api
  annotations:
    release.openshift.io/delete: "true"
...
```
Additional manifest properties such as `spec` may be set if convenient (e.g. because you are looking to make a minimal change vs. a previous version of the manifest), but those properties have no affect on manifests with the delete annotation.

#### Story 2

In release 4.5 two jobs, openshift-service-catalog-controller-manager-remover and openshift-service-catalog-apiserver-remover, were created to remove the Service Catalog.
Now, for release 4.6, these jobs and all their supporting cluster objects must also be removed.
This User Story shows how Service Catalog removal would have been executed had this enhancement been in place.
The Service Catalog is composed of two components, the cluster-svcat-apiserver-operator and the cluster-svcat-controller-manager-operator.
Each of these components use manifests for creation/update of the component's required resources: namespace, roles, operator deployment, etc.
For example, the cluster-svcat-apiserver-operator has the following associated manifests:
* 0000_50_cluster-svcat-apiserver-operator_00_namespace.yaml
* 0000_50_cluster-svcat-apiserver-operator_02_config.crd.yaml
* 0000_50_cluster-svcat-apiserver-operator_03_config.cr.yaml
* 0000_50_cluster-svcat-apiserver-operator_03_configmap.yaml
* 0000_50_cluster-svcat-apiserver-operator_03_version-configmap.yaml
* 0000_50_cluster-svcat-apiserver-operator_04_roles.yaml
* 0000_50_cluster-svcat-apiserver-operator_05_serviceaccount.yaml
* 0000_50_cluster-svcat-apiserver-operator_06_service.yaml
* 0000_50_cluster-svcat-apiserver-operator_07_deployment.yaml
* 0000_50_cluster-svcat-apiserver-operator_08_cluster-operator.yaml

Assuming all the above created resources are `namespace-scoped` and in the cluster-svcat-apiserver-operator namespace openshift-service-catalog-apiserver-operator, all could be deleted by simply deleting the namespace openshift-service-catalog-apiserver-operator.
And, given this enhancement, the namespace delete would be done by adding the delete annotation to the manifest 0000_50_cluster-svcat-apiserver-operator_00_namespace.yaml.

If any `cluster-scoped` resources do exist or resources in another namespace exist, each would have to be deleted explicitly using the same method of adding the delete annotation to the relevant manifest.
If multiple delete manifests are required it is up to the developer to name the manifests such that deletions occur in the correct order.

If resources external to kubernetes must be removed the developer must provide the means to do so.
This is expected to be done through modification of an operator to do the removals during it's finalization.
If operator modification for object removal is necessary that operator would be deleted in a subsequent update.
This eliminates the need for new, possibly complex, CVO logic to handle both an update and a delete of the same object.

Assuming that all Service Catalog resources can be deleted by deleting the relevant cluster-svcat-apiserver-operator and cluster-svcat-controller-manager-operator namespaces, to accomplish Service Catalog deletion release 4.5 would have the following Service Catalog related deletion manifests:
* 0000_50_cluster-svcat-apiserver-operator_00_namespace.yaml
* 0000_50_cluster-svcat-controller-manager-operator_00_namespace.yaml

These two manifests would be removed from release 4.6.
Subsequent 4.5.z releases could still contain these deletion manifests.
CVO would process the manifests and, if the namespaces had already been removed, get an error, log a warning, and continue with the update.
See the [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy) section for details.

### Implementation Details/Notes/Constraints

CVO will use its existing logic to discover and identify given manifests.
Once the manifest has been properly identified CVO will check for the delete annotation release.openshift.io/delete.
If found and set to "true" the associated object will be removed from the cluster.
All other annotation values will result in a CVO error and no further processing of the manifest.

### Risks and Mitigations

A common risk is human error resulting in an incorrect annotation being placed in a manifest.
This results in a CVO error and no further processing of the manifest.

Another risk is that object deletion results in incomplete object removal.
This risk is mitigated by the fact that the object is no longer needed, hence the deletion request, so its continued presence, or partial presence, should not affect cluster operation.

## Design Details

### Test Plan

To be done

### Graduation Criteria

GA. When it works, we ship it.

### Upgrade / Downgrade Strategy

Special consideration must be given to subsequent updates which may still contain manifests with the delete annotation. 
These manifests will result in `object no longer exists` errors assuming the current release had properly and fully removed the given objects.
This enhancement proposes that it is acceptable for subsequent z-level updates to still contain the delete manifests but minor level updates should not and therefore the handling of the delete error will differ between these update levels.
A z-level update will be allowed to proceed while a minor level update will be blocked.
This will be accomplished through the existing CVO precondition mechanism which already behaves in this manner with regard to z-level and minor updates.

### Version Skew Strategy

No special consideration.

## Implementation History

Major milestones in the life cycle of a proposal should be tracked in `Implementation
History`.

## Drawbacks

The idea is to find the best form of an argument why this enhancement should _not_ be implemented.

## Alternatives

To be done
