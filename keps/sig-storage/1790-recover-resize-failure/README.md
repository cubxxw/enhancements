# Recovery from volume expansion failure

## Table of Contents

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [Making allocatedResourceStatus not change unnecessarily for every error in 1.31](#making-allocatedresourcestatus-not-change-unnecessarily-for-every-error-in-131)
  - [Handling of RWX volumes that don't require node expansion](#handling-of-rwx-volumes-that-dont-require-node-expansion)
  - [Making resizeStatus more general in v1.28](#making-resizestatus-more-general-in-v128)
    - [User flow stories](#user-flow-stories)
      - [Case 0 (default PVC creation):](#case-0-default-pvc-creation)
      - [Case 1 (controller+node expandable):](#case-1-controllernode-expandable)
      - [Case 2 (node only expandable volume):](#case-2-node-only-expandable-volume)
      - [Case 3 (Malicious user)](#case-3-malicious-user)
      - [Case 4(Malicious User and rounding to GB/GiB bounaries)](#case-4malicious-user-and-rounding-to-gbgib-bounaries)
      - [Case 5(Rapid successive expansion and shrink)](#case-5rapid-successive-expansion-and-shrink)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Test Plan](#test-plan)
      - [Unit tests](#unit-tests)
      - [Integration tests](#integration-tests)
      - [E2E tests](#e2e-tests)
  - [Graduation Criteria](#graduation-criteria)
    - [Alpha](#alpha)
    - [Beta](#beta)
    - [GA](#ga)
  - [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy)
  - [Version Skew Strategy](#version-skew-strategy)
  - [Monitoring](#monitoring)
- [Production Readiness Review Questionnaire](#production-readiness-review-questionnaire)
  - [Feature enablement and rollback](#feature-enablement-and-rollback)
  - [Rollout, Upgrade and Rollback Planning](#rollout-upgrade-and-rollback-planning)
  - [Monitoring requirements](#monitoring-requirements)
  - [Dependencies](#dependencies)
  - [Scalability](#scalability)
  - [Troubleshooting](#troubleshooting)
- [Implementation History](#implementation-history)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
  - [Why not use pvc.Status.Capacity for tracking quota?](#why-not-use-pvcstatuscapacity-for-tracking-quota)
  - [Allow admins to manually fix PVCs which are stuck in resizing condition](#allow-admins-to-manually-fix-pvcs-which-are-stuck-in-resizing-condition)
  - [Solving limitation of allowing restore all the way to original size](#solving-limitation-of-allowing-restore-all-the-way-to-original-size)
- [Infrastructure Needed [optional]](#infrastructure-needed-optional)
<!-- /toc -->

## Release Signoff Checklist

- [x] (R) Enhancement issue in release milestone, which links to KEP dir in [kubernetes/enhancements] (not the initial KEP PR)
- [x] (R) KEP approvers have approved the KEP status as `implementable`
- [x] (R) Design details are appropriately documented
- [x] (R) Test plan is in place, giving consideration to SIG Architecture and SIG Testing input
  - [x] e2e Tests for all Beta API Operations (endpoints)
  - [ ] (R) Ensure GA e2e tests meet requirements for [Conformance Tests](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/conformance-tests.md) 
  - [ ] (R) Minimum Two Week Window for GA e2e tests to prove flake free
- [x] (R) Graduation criteria is in place
- [ ] (R) Production readiness review completed
- [ ] Production readiness review approved
- [x] "Implementation History" section is up-to-date for milestone
- [x] User-facing documentation has been created in [kubernetes/website], for publication to [kubernetes.io]
- [x] Supporting documentation e.g., additional design documents, links to mailing list discussions/SIG meetings, relevant PRs/issues, release notes


## Summary

A user may expand a PersistentVolumeClaim(PVC) to a size which may not be supported by
underlying storage provider. In which case - typically expansion controller forever tries to
expand the volume and keeps failing. We want to make it easier for users to recover from
expansion failures, so that user can retry volume expansion with values that may succeed.

To enable recovery, this proposal relaxes API validation for PVC objects to allow reduction in
requested size. This KEP also adds a new field called `pvc.Status.AllocatedResources` to allow resource quota
to be tracked accurately if user reduces size of their PVCs.

## Motivation

Sometimes a user may expand a PVC to a size which may not be supported by
underlying storage provider. Lets say PVC size was 10GB and user expanded
it to 500GB but underlying storage provider can not support 500GB volume
for some reason. We want to make it easier for user to retry re-sizing by
requesting more reasonable value say - 100GB.

Problem in allowing users to reduce size is - a malicious user can use
this feature to abuse the quota system. They can do so by rapidly
expanding and then shrinking the PVC. In general - both CSI
and in-tree volume plugins have been designed to never perform actual
shrink on underlying volume, but it can fool the quota system in believing
the user is using less storage than he/she actually is.

To solve this problem - we propose that although users are allowed to reduce size of their
PVC (as long as requested size > `pvc.Status.Capacity`), quota calculation
will use `max(pvc.Spec.Capacity, pvc.Status.AllocatedResources)` and reduction in `pvc.Status.AllocatedResources`
is only done by resize-controller after previously issued expansion has failed with some kind of terminal failure.

### Goals

- Allow users to cancel previously issued volume expansion requests, assuming they are not yet successful or have failed.
- Allow users to retry expansion request with smaller value than original requested size in `pvc.Spec.Resources`, assuming previous requests are not yet successful or have failed.

### Non-Goals

- As part of this KEP we do not intend to actually support shrinking of underlying volume.
- As part of this KEP when user intends to recover from volume expansion failures, they must recover to a size slightly larger than original size of the volume. For example - if user expanded a volume from `10Gi` to `100Gi` and expansion to `100Gi` is failing, then we only permit recovery to a size greater than `10Gi`, for example - `11Gi` or anything higher than `10Gi`.

## Proposal

As part of this proposal, we are mainly proposing three changes:

1. Relax API validation on PVC update so as reducing `pvc.Spec.Resoures` is allowed, as long as requested size is `>pvc.Status.Capacity`.
2. Add a new field in PVC API called - `pvc.Status.AllocatedResources` for the purpose of tracking used storage size. By default only api-server or resize-controller can set this field.
3. Add a new field in PVC API called - `pvc.Status.ResizeStatus` for purpose of tracking status of volume expansion. Following are possible values of newly introduced `ResizeStatus` field:
   - nil // General default because type is a pointer.
   - empty // When expansion is complete, the empty string is set by resize controller or kubelet.
   - ControllerExpansionInProgress // state set when external resize controller starts expanding the volune.
   - ControllerExpansionFailed // state set when expansion has failed in external resize controller with a terminal error. Transient errors don't set ControllerExpansionFailed.
   - NodeExpansionPending // state set when external resize controller has finished expanding the volume but further expansion is needed on the node.
   - NodeExpansionInProgress // state set when kubelet starts expanding the volume.
   - NodeExpansionFailed // state set when expansion has failed in kubelet with a terminal error. Transient errors don't set NodeExpansionFailed.
3. Update quota code to use `max(pvc.Spec.Resources, pvc.Status.AllocatedResources)` when evaluating usage for PVC.

### Making allocatedResourceStatus not change unnecessarily for every error in 1.31

We are trying to reduce number of state changes which can happen when volume expansion on either the kubelet or external-resizer fails.

We are considering following gRPC error codes as "infeasible":
- INVALID_ARGUMENt
- OUT_OF_RANGE
- NOT_FOUND

In the external-resizer if `ControllerExpandVolume` fails with any of the error codes above, controller expansion will be marked as failed and resizing will be retried at slower rate. For all the other errors - an event will be generated and a condition will be added to PVC that expansion has failed, but state change will not be recorded in `allocatedResourceStatus`.


On the node side - `allocatedResourceStatus` will only be updated with failed expansion if:
- `NodeExpandVolume` failed with one of the `infeasible` error codes from above.
- `NodeExpandVolume` failed with a final error and there is a pending pvc size request change from the user.

This will allow external-resizer to recover safely from node expansion failures too.

![New flow kubelet](./Expanding volume - Kubelet Loop.png)

### Handling of RWX volumes that don't require node expansion

There are CSI drivers which return `NodeExpansionRequired: false` after `ControllerExpandVolume` is finished, but in kubelet to handle the case of node-expansion of RWX volumes, we usually MUST call `NodeExpandVolume` on each node where volume is attached, even if `ControllerExpandVolume` is finished and no node-expansion is required. This special case *only* applies to RWX volumes.

To avoid calling `NodeExpandVolume` for such volumes, we are proposing that we add an annotation to PVC called - `volume.kubernetes.io/node-expansion-not-required` when `ControllerExpandVolume` is finished and `NodeExpansionRequired` is set to `false` - https://github.com/kubernetes-csi/external-resizer/pull/496/files .

In kubelet if a RWX volume has this annotation, no `NodeExpandVolume` will be called and and updated size of the volume will simply be recorded in actual state of the world.This is implemented in - https://github.com/kubernetes/kubernetes/pull/131907 . 

The proposed mechanism is fully backward compatible and only applies to RWX volumes.


### Making resizeStatus more general in v1.28

After [some discussion](https://github.com/kubernetes/kubernetes/pull/116335#issuecomment-1624566731) with sig-storage folks, we are proposing that we rename `pvc.Status.ResizeStatus` to `pvc.Status.AllocatedResourceStatus` and make it a map.

So basically new API will look like:

```go
type ClaimResourceStatus string

const (
    PersistentVolumeClaimControllerResizeProgress ClaimResourceStatus = "ControllerResizeInProgress"
    PersistentVolumeClaimControllerResizeFailed ClaimResourceStatus = "ControllerResizeFailed"

    PersistentVolumeClaimNodeResizePending ClaimResourceStatus = "NodeResizePending"
    PersistentVolumeClaimNodeResizeInProgress ClaimResourceStatus = "NodeResizeInProgress"
    PersistentVolumeClaimNodeResizeFailed ClaimResourceStatus = "NodeResizeFailed"
)

// PersistentVolumeClaimStatus represents the status of PV claim
type PersistentVolumeClaimStatus struct {
    // Phase represents the current phase of PersistentVolumeClaim
    // +optional
    Phase PersistentVolumeClaimPhase
    // AccessModes contains all ways the volume backing the PVC can be mounted
    // +optional
    AccessModes []PersistentVolumeAccessMode
    // Represents the actual resources of the underlying volume
    // +optional
    Capacity ResourceList
    // +optional
    AllocatedResources ResourceList
    // AllocatedResourceStatus stores a map of resource that is being expanded
    // and possible status that the resource exists in.
    // Some examples may be:
    //  pvc.status.allocatedResourceStatus["storage"] = "ControllerResizeInProgress"
    AllocatedResourceStatus map[ResourceName]ClaimResourceStatus
}
```




#### User flow stories

##### Case 0 (default PVC creation):
- User creates a 10Gi PVC by setting - `pvc.spec.resources.requests["storage"] = "10Gi"`.

##### Case 1 (controller+node expandable):
- User increases 10Gi PVC to 100Gi by changing - `pvc.spec.resources.requests["storage"] = "100Gi"`.
- Quota controller uses `max(pvc.Status.AllocatedResources, pvc.Spec.Resources)` and adds `90Gi` to used quota.
- Expansion controller starts expanding the volume and sets `pvc.Status.AllocatedResources` to `100Gi`.
- Expansion controller also sets `pvc.Status.AllocatedResourceStatus['storage']` to `ControllerResizeInProgress`.
- Expansion to 100Gi fails and hence `pv.Spec.Capacity` and `pvc.Status.Capacity `stays at 10Gi.
- Expansion controller sets `pvc.Status.AllocatedResourceStatus['storage']` to `ControllerResizeFailed`.
- User requests size to 20Gi.
- Expansion controler notices that previous expansion to last known `allocatedresources` failed, so it sets new `allocatedResources` to `20G`
- Expansion succeeds and `pvc.Status.Capacity` and `pv.Spec.Capacity` report new size as `20Gi`.
- Quota controller sees a reduction in used quota because `max(pvc.Spec.Resources, pvc.Status.AllocatedResources)` is 20Gi.

##### Case 2 (node only expandable volume):
- User increases 10Gi PVC to 100Gi by changing - `pvc.spec.resources.requests["storage"] = "100Gi"`
- Quota controller uses `max(pvc.Status.AllocatedResources, pvc.Spec.Resources)` and adds `90Gi` to used quota.
- Expansion controller starts expanding the volume and sets `pvc.Status.AllocatedResources` to `100Gi`.
- Expansion controller also sets `pvc.Status.AllocatedResourceStatus['storage']` to `ControllerResizeInProgress`.
- Since expansion operations in control-plane are NO-OP, expansion in control-plane succeeds and `pv.Spec` is set to `100G`.
- Expansion controller also sets `pvc.Status.AllocatedResourceStatus['storage']` to `NodeResizePending`.
- Expansion starts on the node and kubelet sets `pvc.Status.AllocatedResourceStatus['storage']` to `NodeResizeInProgress`.
- Expansion fails on the node with a final error.
- Kubelet sets `pvc.Status.AllocatedResourceStatus['storage']` to `NodeResizeFailed`.
- Since pvc has `pvc.Status.AllocatedResourceStatus['storage']` set to `NodeResizeFailed` - kubelet will stop retrying node expansion.
- At this point Kubelet will wait for `pvc.Status.AllocatedResourceStatus['storage']` to be `NodeResizePending`.
- User requests size to 20Gi.
- Expansion controller starts expanding the volume and sees that last expansion failed on the node and driver does not have control-plane expansion.
- Expansion controller sets `pvc.Status.AllocatedResources` to `20G`.
- Expansion controller also sets `pvc.Status.AllocatedResourceStatus['storage']` to `ControllerResizeInProgress`.
- Since expansion operations in control-plane are NO-OP, expansion in control-plane succeeds and `pv.Spec` is set to `20G`.
- Expansion succeed on the node with latest `allocatedResources` and `pvc.Status.Size` is set to `20G`.
- Expansion controller also sets `pvc.Status.AllocatedResourceStatus['storage']` to `NodeResizePending`.
- Kubelet can now retry expansion and expansion on node succeeds.
- Kubelet sets `pvc.Status.AllocatedResourceStatus['storage']` to empty string and `pvc.Status.Capacity` to new value.
- Quota controller sees a reduction in used quota because `max(pvc.Spec.Resources, pvc.Status.AllocatedResources)` is 20Gi.


##### Case 3 (Malicious user)
- User increases 10Gi PVC to 100Gi by changing `pvc.spec.resources.requests["storage"] = "100Gi"`
- Quota controller uses `max(pvc.Status.AllocatedResources, pvc.Spec.Resources)` and adds `90Gi` to used quota.
- Expansion controller slowly starts expanding the volume and sets `pvc.Status.AllocatedResources` to `100Gi` (before expanding).
- Expansion controller also sets `pvc.Status.AllocatedResourceStatus['storage']` to `ControllerResizeInProgress`.
- At this point -`pv.Spec.Capacity` and `pvc.Status.Capacity` stays at 10Gi until the resize is finished.
- While the storage backend is re-sizing the volume, user requests size 20Gi by changing `pvc.spec.resources.requests["storage"] = "20Gi"`
- Expansion controller notices that previous expansion to last known `allocatedresources` is still in-progress.
- Expansion controller keeps expanding the volume to `allocatedResources`.
- Expansion to `100G` succeeds and `pv.Spec.Capacity` and `pvc.Status.Capacity` report new size as `100G`.
- Although `pvc.Spec.Resources` reports size as `20G`, expansion to `20G` is never attempted.
- Quota controller sees no change in storage usage by the PVC because `pvc.Status.AllocatedResources` is 100Gi.

##### Case 4(Malicious User and rounding to GB/GiB bounaries)
- User requests a PVC of 10.1GB but underlying storage provides provisions a volume of 11GB after rounding.
- Size recorded in `pvc.Status.Capacity` and `pv.Spec.Capacity` is however still 10.1GB.
- User expands expands the PVC to 100GB.
- Quota controller uses `max(pvc.Status.AllocatedResources, pvc.Spec.Resources)` and adds `89.9GB` to used quota.
- Expansion controller starts expanding the volume and sets `pvc.Status.AllocatedResources` to `100GB` (before expanding).
- Expansion controller also sets `pvc.Status.AllocatedResourceStatus['storage']` to `ControllerResizeInProgress`.
- At this point -`pv.Spec.Capacity` and `pvc.Status.Capacity` stays at 10.1GB until the resize is finished.
- while resize was in progress - expansion controler crashes and loses state.
- User reduces the size of PVC to 10.5GB.
- Expansion controller notices that previous expansion to last known `allocatedresources` is still in-progress.
- Expansion controller starts expanding the volume to last known `allocatedResources` (which is `100GB`).
- Expansion to `100G` succeeds and `pv.Spec.Capacity` and `pvc.Status.Capacity` report new size as `100G`.
- Although `pvc.Spec.Resources` reports size as `10.5GB`, expansion to `10.5GB` is never attempted.
- Quota controller sees no change in storage usage by the PVC because `pvc.Status.AllocatedResources` is 100Gi.

##### Case 5(Rapid successive expansion and shrink)
- User increases 10Gi PVC to 100Gi by changing `pvc.spec.resources.requests["storage"] = "100Gi"`
- Quota controller uses `max(pvc.Status.AllocatedResources, pvc.Spec.Resources)` and adds `90Gi` to used quota.
- Expansion controller slowly starts expanding the volume and sets `pvc.Status.AllocatedResources` to `100Gi` (before expanding).
- Expansion controller also sets `pvc.Status.AllocatedResourceStatus['storage']` to `ControllerResizeInProgress`.
- At this point -`pv.Spec.Capacity` and `pvc.Status.Capacity` stays at 10Gi until the resize is finished.
- While the storage backend is re-sizing the volume, user requests size 200Gi by changing `pvc.spec.resources.requests["storage"] = "200Gi"`
- Quota controller uses `max(pvc.Status.AllocatedResources, pvc.Spec.Resources)` and adds `100Gi` to used quota.
- Since `pvc.Status.AllocatedResourceStatus['storage']` is in `ControllerResizeInProgress` - expansion controller still chooses last `pvc.Status.AllocatedResources` as new size.
- User reduces size back to `20Gi`.
- Quota controller uses `max(pvc.Status.AllocatedResources, pvc.Spec.Resources)` and *returns* `100Gi` to used quota.
- Expansion controller notices that previous expansion to last known `allocatedresources` is still in-progress.
- Expansion controller keeps expanding the volume to `allocatedResources`.
- Expansion to `100G` succeeds and `pv.Spec.Capacity` and `pvc.Status.Capacity` report new size as `100G`.
- Although `pvc.Spec.Resources` reports size as `20G`, expansion to `20G` is never attempted.
- Quota controller sees no change in storage usage by the PVC because `pvc.Status.AllocatedResources` is 100Gi.


### Risks and Mitigations

- Once expansion is initiated, the lowering of requested size is only allowed upto a value *greater* than `pvc.Status`. It is not possible to entirely go back to previously requested size. This should not be a problem however in-practice because user can retry expansion with slightly higher value than `pvc.Status` and still recover from previously failing expansion request.

## Design Details

We propose that by relaxing validation on PVC update to allow users to reduce `pvc.Spec.Resources`, it becomes possible to cancel previously issued expansion requests or retry expansion with a lower value if previous request has not been successful. In general - we know that volume plugins are designed to never perform actual shrinking of the volume, for both in-tree and CSI volumes. Moreover if a previously issued expansion has been successful and user
reduces the PVC request size, for both CSI and in-tree plugins they are designed to return a successful response with NO-OP. So, reducing requested size will be a safe operation and will never result in data loss or actual shrinking of volume.

We however do have a problem with quota calculation because if a previously issued expansion is successful but is not recorded(or partially recorded) in api-server and user reduces requested size of the PVC, then quota controller will assume it as actual shrinking of volume and reduce used storage size by the user(incorrectly). Since we know actual size of the volume only after performing expansion(either on node or controller), allowing quota to be reduced on PVC size reduction will allow an user to abuse the quota system.

To solve aforementioned problem - we propose that, a new field will be added to PVC, called `pvc.Status.AllocatedResources`. When user expands the PVC, and when expansion-controller starts volume expansion - it will set `pvc.Status.AllocatedResources` to user requested value in `pvc.Spec.Resources` before performing expansion and it will set `pvc.Status.AllocatedResourceStatus[storage]` to `ControllerResizeInProgress`. The quota calculation will be updated to use `max(pvc.Spec.Resources, pvc.Status.AllocatedResources)` which will ensure that abusing quota will not be possible.

Resizing operation in external resize controller will always work towards full-filling size recorded in `pvc.Status.AllocatedResources` and only when previous operation has finished(i.e `pvc.Status.AllocatedResourceStatus[storage]` is nil) or when previous operation has failed with a terminal error - it will use new user requested value from `pvc.Spec.Resources`.

Kubelet on the other hand will only expand volumes for which `pvc.Status.AllocatedResourceStatus[storage]` is in `NodeResizePending` or `NodeResizeInProgress` state and `pv.Spec.Cap > pvc.Status.Cap`. If a volume expansion fails in kubelet with a terminal error(which will set `NodeResizeFailed` state) - then it must wait for resize controller in external-resizer to reconcile the state and put it back in `NodeResizePending`.

When user reduces `pvc.Spec.Resources`, expansion-controller will set `pvc.Status.AllocatedResources` to lower value only if one of the following is true:

1. If `pvc.Status.AllocatedResourceStatus[storage]` is `ControllerResizeFailed` (indicating that previous expansion to last known `allocatedResources` failed with a final error) and previous control-plane has not succeeded.
2. If `pvc.Status.AllocatedResourceStatus[storage]` is `NodeResizeFailed` and SP supports node-only expansion (indicating that previous expansion to last known `allocatedResources` failed on node with a final error).
3. If `pvc.Status.AllocatedResourceStatus[storage]` is `nil` or `empty` and previous `ControllerExpandVolume` has not succeeded.

![Determining new size](./get_new_size.png)

**Note**: Whenever resize controller or kubelet modifies `pvc.Status` (such as when setting both `AllocatedResources` and `pvc.Status.AllocatedResourceStatus`) - it is expected that all changes to `pvc.Status` are submitted as part of same patch request to avoid race conditions.

The complete expansion and recovery flow of both control-plane and kubelet is documented in attached PDF. The attached pdf - documents complete volume expansion flow via state diagrams and is much more exhaustive than the text above.

### Test Plan

[x] I/we understand the owners of the involved components may require updates to
existing tests to make this code solid enough prior to committing the changes necessary
to implement this enhancement.

##### Unit tests

In external-resizer:
- pkg/controller/expand_and_recover.go: 09/30/2024 - 78.3%

##### Integration tests

None

##### E2E tests

There are already e2e tests running that verify correctness of this feature - https://testgrid.k8s.io/presubmits-kubernetes-nonblocking#pull-kubernetes-e2e-gce-cos-alpha-features

### Graduation Criteria

#### Alpha

- *Alpha* in 1.23 behind `RecoverVolumeExpansionFailure` feature gate with set to a default of `false`.

#### Beta

- *Beta* in 1.29: We are going to move this to beta with enhanced e2e and more stability improvements.
  -  There are already e2e tests running that verify correctness of this feature -
  https://testgrid.k8s.io/presubmits-kubernetes-nonblocking#pull-kubernetes-e2e-gce-cos-alpha-features
  
#### GA  

- The feature has been extensively tested in CI and deployed with drivers in real world. For example - https://github.com/kubernetes-sigs/azuredisk-csi-driver/blob/master/charts/latest/azuredisk-csi-driver/templates/csi-azuredisk-controller.yaml#L166
- The test grid already has tests for the feature.

### Upgrade / Downgrade Strategy

- For the case of older kubelet running with newer resizer, the kubelet must handle the newer fields introduced by this feature even if feature-gate is not enabled. Having said that, if kubelet is older and has no notion of this feature at all and `api-server` and `external-resizer` is newer, it is possible that kubelet will not properly update `allocatedResourceStatus` after expansion is finished on the node. This is a known limitation and if user wants to keep running older version of kubelet (older than v1.31 as of this writing and hence unsupported), then they MUST not update external-resizer.
- In general `external-resizer` should only be upgraded(and `RecoverVolumeExpansionFailure` enabled), after `kubelet`, `api-server` and `kube-controller-manager` already has this feature enabled.

### Version Skew Strategy

To use this feature `RecoverVolumeExpansionFailure` should be enabled in `kubelet`, `external-resizer`, `api-server` and
`kube-controller-manager`. 

If for some reason - `external-resizer` is running with older version and while core kubernetes components have been upgraded, then volume expansion will follow old behaviour and no recovery from volume expansion failure will be possible.

If `external-resizer` has been upgraded to latest version (and feature-gate is enabled), but core kubernetes components such as `api-server` or `kubelet` has not been updated, then as well no recovery will be possible, because `api-server` will not allow users to reduce size of `pvc`. 

If `api-server` and `external-resizer` has been upgraded and has feature-gate enabled but `kubelet` is still 
older then for volume types which require expansion on kubelet, resize status will not be updated correctly until `kubelet` version is upgraded. This will not block volume from being mounted but it does mean that resize operation will not fully finish.

### Monitoring

* We will add events that are added to both PV and PVC objects for user reducing PVC size.

## Production Readiness Review Questionnaire

### Feature enablement and rollback

* **How can this feature be enabled / disabled in a live cluster?**
  - [x] Feature gate (also fill in values in `kep.yaml`)
    - Feature gate name: `RecoverVolumeExpansionFailure`
    - Components depending on the feature gate: kube-apiserver, kube-controller-manager, csi-external-resizer, kubelet
  - [ ] Other
    - Describe the mechanism:
    - Will enabling / disabling the feature require downtime of the control
      plane?
    - Will enabling / disabling the feature require downtime or reprovisioning
      of a node?

* **Does enabling the feature change any default behavior?**
  Allow users to reduce size of pvc in `pvc.spec.resources`. In general this was not permitted before,
  so it should not break any of existing automation. This means that if `pvc.Status.AllocatedResources` is available it will be
  used for calculating quota.

  To facilitate older kubelet - external resize controller will set `pvc.Status.AllocatedResourceStatus[storage]` to "''" after entire expansion process is complete. This will ensure that `ResizeStatus` is updated
after expansion is complete even with older kubelet. No recovery from expansion failure will be possible in this case and the workaround will be removed once feature goes GA.

  One more thing to keep in mind is - enabling this feature in kubelet while keeping it disabled in external-resizer will cause
  all volume expansions operations to get stuck(similar thing will happen when feature moves to beta and kubelet is newer but external-resizer sidecar is older).
  This limitation will be documented in external-resizer.

* **Can the feature be disabled once it has been enabled (i.e. can we rollback
  the enablement)?**
  Yes the feature gate can be disabled once enabled. However quota resources present in the cluster will be based off a field(`pvc.Status.AllocatedResources`) which is no longer updated.
  Currently without this feature - quota calculation is entirely based on `pvc.Spec.Resources` and when feature is enabled it will based off `max(pvc.Spec.Resources, pvc.Status.AllocatedResources)`
  so when the feature is disabled, cluster might be reporting stale quota. To fix this issue - cluster admins can re-create `ResourceQuota` objects so as quota controller can recompute the
  quota using `pvc.Spec.Resources`.

* **What happens if we reenable the feature if it was previously rolled back?**
  It should be possible to re-enable the feature after disabling it. When feature is disabled and re-enabled, users will be able to
  reduce size of `pvc.Spec.Resources` to cancel previously issued expansion but in case `pvc.Spec.Resources` reports lower value than
  what was reported in `pvc.Status.AllocatedResources` (which would mean resize controller tried to expand this volume to bigger size previously)
  quota calculation will be based off `pvc.Status.AllocatedResources`.

* **Are there any tests for feature enablement/disablement?**
  The e2e framework does not currently support enabling and disabling feature
  gates. However, we will write extensive unit tests to test behaviour of code when feature gate is disabled and when feature gate is enabled.

### Rollout, Upgrade and Rollback Planning

* **How can a rollout fail? Can it impact already running workloads?**
  This change should not impact existing workloads and requires user interaction via reducing pvc capacity.

* **What specific metrics should inform a rollback?**
  No specific metric but if expansion of PVCs are being stuck (can be verified from `pvc.Status.Conditions`)
  then user should plan a rollback.

* **Were upgrade and rollback tested? Was upgrade->downgrade->upgrade path tested?**
  We have not fully tested upgrade and rollback but as part of beta process we will have it tested.

* **Is the rollout accompanied by any deprecations and/or removals of features,
  APIs, fields of API types, flags, etc.?**
  This feature deprecates no existing functionality.

### Monitoring requirements

###### How can an operator determine if the feature is in use by workloads?**

The Recovery feature as such designed does not require if external operator must be able to observe
if the feature is in-use by the workloads. The reason is, proposed changes already enhance
user experience of resizing workflow by providing additional states such as `allocatedResourceStatus`
and `allocatedResources` in pvc's status and there is nothing special about recovery in itself. 

Operators can already observe if volume's are being resized by using `pvc.Status.Conditions` and other 
metrics documented below.

###### How can someone using this feature know that it is working for their instance?  

Since feature requires user interaction, reducing the size of the PVC is only supported
if this feature-gate is enabled.
  
###### What are the SLIs (Service Level Indicators) an operator can use to determine the health of the service

  - [X] Metrics
    - controller expansion operation duration:
        - Metric name: csi_sidecar_operations_seconds{method_name="/csi.v1.Controller/ControllerExpandVolume}
        - [Optional] Aggregation method: percentile
        - Components exposing the metric: external-resizer
    - controller expansion operation errors:
        - Metric name: csi_sidecar_operations_seconds{method_name="/csi.v1.Controller/ControllerExpandVolume, grpc_status_code!="OK"}
        - [Optional] Aggregation method: cumulative counter
        - Components exposing the metric: external-resizer
    - CSI node expansion operation durations:
        - Metric name: csi_operations_seconds{method_name="/csi.v1.Controller/NodeExpandVolume}
        - [Optional] Aggregation method: cumulative counter
        - Components exposing the metric: kubelet
    - node expansion operation duration:
        - Metric name: storage_operation_duration_seconds{operation_name=volume_fs_resize}
        - [Optional] Aggregation method: percentile
        - Components exposing the metric: kubelet
    - node expansion operation errors:
        - Metric name: storage_operation_errors_total{operation_name=volume_fs_resize}
        - [Optional] Aggregation method: cumulative counter
        - Components exposing the metric: kubelet
  - [ ] Other (treat as last resort)
    - Details:

###### What are the reasonable SLOs (Service Level Objectives) for the above SLIs?

  After this feature is rolled out, there should not be any increase in 95-99 percentile of
  both `csi_sidecar_operations_seconds{method_name="/csi.v1.Controller/ControllerExpandVolume"}` and `storage_operation_duration_seconds{operation_name=volume_fs_resize}` durations. 
  
  Also the error rate should not increase for `csi_sidecar_operations_seconds{method_name="/csi.v1.Controller/ControllerExpandVolume"}` metric.

###### Are there any missing metrics that would be useful to have to improve observability of this feature?

We think that existing resizing related metrics are sufficient and we do as such think we need to introduce new
metrics for tracking this feature.

### Dependencies

* **Does this feature depend on any specific services running in the cluster?**
  For CSI volumes this feature requires users to run with latest version of `external-resizer`
  sidecar which also should have `RecoverVolumeExpansionFailure` feature enabled. 
  
### Scalability

_For beta, this section is required: reviewers must answer these questions._

_For GA, this section is required: approvers should be able to confirms the
previous answers based on experience in the field._

###### Will enabling / using this feature result in any new API calls

  Yes - if user recovers a volume from failed expansion. When user expands a PVC and expansion fails on the node, then
  expansion controller in control-plane must intervene and verify if it is safe
  to retry expansion on the kubelet.This requires round-trips between kubelet
  and control-plane and hence more API calls.

###### Will enabling / using this feature result in introducing new API types
  None

###### Will enabling / using this feature result in any new calls to cloud provider

  Since expansion operation is idempotent and expansion controller
  must verify if it is safe to retry expansion with lower values, there could be 
  additional calls to CSI drivers (and hence cloudproviders), when user attempts recovery from 
  failed expansion.
  
  On the other hand - previously we retried expansion of failed volumes infinitiely and hence incurred
  significant api usage of both k8s and cloudprovider. This enhancement significantly reduces this by 
  allowing recovery and slower pace of retries for infeasible expansion operations.

###### Will enabling / using this feature result in increasing size or count of the existing API objects?

Since this feature adds a new field to PersistentVolumeClaim object it will increase size of the object.
  Describe them providing:
  - API type(s): PersistentVolumeClaim
  - Estimated increase in size: 40B

###### Will enabling / using this feature result in increasing time taken by any operations covered by [existing SLIs/SLOs]?

In some cases if expansion fails on the node, then since it requires now correction
  from control-plane, it may require longer time for entire expansion.

###### Will enabling / using this feature result in non-negligible increase of resource usage (CPU, RAM, disk, IO, ...) in any components

It should not result in increase of resource usage.
  
###### Can enabling / using this feature result in resource exhaustion of some node resources (PIDs, sockets, inodes, etc.)?

Enabling this feature should not result in resource exhaustion of node resources.


### Troubleshooting

Troubleshooting section serves the `Playbook` role as of now. We may consider
splitting it into a dedicated `Playbook` document (potentially with some monitoring
details). For now we leave it here though.

_This section must be completed when targeting beta graduation to a release._

* **How does this feature react if the API server and/or etcd is unavailable?**
  If API server or etcd is not available midway through a request then some of the
  modifications to API objects can not be saved. But this should be a recoverable state  because
  expansion operation is idempotent and controller will retry the operation and succeed
  whenever API server does come back.

* **What are other known failure modes?**
  For each of them fill in the following information by copying the below template:
  - No recovery is possible if volume has been expanded on control-plane and only failing on node.
    - Detection: Expansion is stuck with `AllocatedResourceStatus[storage]` - `NodeResizePending` or `NodeResizeFailed`.
    - Mitigations: This should not affect any of existing PVCs but this was already broken in some sense and if volume has been 
      expanded in control-plane then we can't allow users to shrink their PVCs because that would violate the quota.
    - Diagnostics: Expansion is stuck with `AllocatedResourceStatus['storage']` - `NodeResizePending` or `NodeResizeFailed`.
    - Testing: There are some unit tests for this failure mode.

* **What steps should be taken if SLOs are not being met to determine the problem?**
  If admin notices an increase in expansion failure operations via aformentioned metrics or 
  by observing `pvc.Status.AllocatedResourceStatus['storage']` then:
      - Check events on the PVC and observe why is PVC expansion failing.
      - Gather logs from kubelet and external-resizer and check for problems.

[supported limits]: https://git.k8s.io/community//sig-scalability/configs-and-limits/thresholds.md
[existing SLIs/SLOs]: https://git.k8s.io/community/sig-scalability/slos/slos.md#kubernetes-slisslos


## Implementation History

- 2020-01-27 Initial KEP pull request submitted
- 2023-02-03 Changing the APIs of `pvc.Status.ResizeStatus` by renaming it to `pvc.Status.AllocatedResourceStatus` and converting it to a map.
- 2024-09-12 Propose move to beta status.

## Drawbacks

- One drawback is while this KEP allows an user to reduce requested PVC size, it does not allow reduction in size all the way to same value as original size recorded in `pvc.Status.Cap`. The reason is currently resize controller and kubelet work on a volume one after the other and we need special logic in both kubelet and control-plane to process reduction in size all the way upto original value. This is somewhat racy and we need a signaling mechanism between control-plane and kubelet when `pvc.Status.AllocatedResources` needs to be adjusted. We plan to revisit and address this in next release.


## Alternatives

There were several alternatives considered before proposing this KEP.

### Why not use pvc.Status.Capacity for tracking quota?


`pvc.Status.Capacity` can't be used for tracking quota because pvc.Status.Capacity is calculated after binding happens, which could be when pod is started. This would allow an user to overcommit because quota won't reflect accurate value until PVC is bound to a PV.

### Allow admins to manually fix PVCs which are stuck in resizing condition

We also considered if it is worth leaving this problem as it is and allow admins to fix it manually since the problem that this KEP fixes is not very frequent(there have not been any reports from end-users about this bug). But there are couple of problems with this:
- A workflow that depends on restarting the pods after resizing is successful - can't complete if resizing is forever stuck. This is the case with current design of statefulset expansion KEP.
- If a storage type only supports node expansion and it keeps failing then PVC could become unusable and prevent its usage.

### Solving limitation of allowing restore all the way to original size

As mentioned above - one limitation of this KEP is, users can only recover upto a size greater than previous value recorded in `pvc.Status.Cap`.


## Infrastructure Needed [optional]
