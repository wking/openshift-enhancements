---
title: preflight-from-target-release
authors:
  - "@wking"
  - "@fao89"
reviewers: # Include a comment about what domain expertise a reviewer is expected to bring and what area of the enhancement you expect them to focus on. For example: - "@networkguru, for networking aspects, please look at IP bootstrapping aspect"
  - "@hongkailiu, for accepted risks integration and conditional update aspects"
approvers:
  - "@PratikMahajan"
api-approvers:
  - "@JoelSpeed"
creation-date: 2026-01-14
last-updated: 2026-01-16
status: provisional
tracking-link:
  - https://issues.redhat.com/browse/OCPSTRAT-2843
see-also:
  - "/enhancements/update/accepted-risks.md"
replaces:
superseded-by:
---

# Preflight checks from target release

## Summary

Give cluster administrators a way to run preflight checks from a target release before launching an update.

## Motivation

The existing cluster update validation mechanisms have limitations that this enhancement addresses:

1. **Skip-level update challenges**: With skip-level updates on the horizon [KEP-4330](https://github.com/kubernetes/enhancements/tree/master/keps/sig-architecture/4330-compatibility-versions), the existing `Upgradeable` tooling does not allow a 5.0 cluster to distinguish between risks for updating to 5.1 and risks for updating skip-level directly to 5.2.
    This preflight enhancement would allow that 5.0 cluster to run checks from the 5.2.z target release to report about any concerns that target release had with the current version's state.
1. **Component maintainer workflow**: Previously explored in [PR #363](https://github.com/openshift/enhancements/pull/363), operators needed complex backporting strategies to warn about future incompatibilities.
    This approach allows components to define compatibility checks in their target release rather than backporting knowledge to previous versions.

This preflight enhancement allows clusters to run compatibility checks from a target release without committing to the update, enabling administrators to understand and plan for potential issues before beginning an update.

### User Stories

#### As a cluster administrator operating a production OpenShift cluster

I want to proactively check compatibility with a target release without committing to an update, so that I can:
- Assess risks for skip-level updates (e.g., from 5.0 directly to 5.2).
- Validate that my cluster configuration and workloads are compatible before scheduling an update, optionally well before, so I have plenty of time to calmly address any detected issues.
- Review specific risk names that can be accepted using the accepted-risks mechanism introduced in [accepted-risks](accepted-risks.md)

#### As a component maintainer developing OpenShift operators

I want to write compatibility checks in my target release rather than backporting compatibility logic, so that I can:
- Define what configurations from previous releases are incompatible with my new version.
- Leverage the latest understanding of compatibility requirements without backporting knowledge to older releases.
- Reduce the maintenance burden of keeping compatibility checks synchronized across multiple release branches.
- Focus compatibility validation logic in the release where breaking changes are introduced, which may include additional context like release manifest YAML that is not available in the older release.

**Example**: A 5.2 networking operator can check if a 5.0 cluster's dual-stack configuration is compatible with 5.2's networking changes, without requiring the 5.0 operator to know about future 5.2 requirements.

#### As a cluster lifecycle engineer

I want to integrate preflight checks into automated update workflows, so that I can:
- Run preflight validations as part of CI/CD pipelines before approving cluster updates.
- Generate reports on fleet-wide compatibility for upcoming releases.
- Implement automated update policies that only proceed when preflight checks pass.

### Goals

* **Proactive risk assessment**: Enable cluster administrators to identify potential update risks before committing to an update, particularly for skip-level updates.
    This aligns with upstream Kubernetes work on compatibility versions in [KEP-4330](https://github.com/kubernetes/enhancements/tree/bb6bf298fdc524454b6fd477c84f5760b0f98c40/keps/sig-architecture/4330-compatibility-versions).
* **Target release compatibility checks**: Allow components to define compatibility checks in their target release rather than requiring backports to previous releases.
* **Integration with accepted-risks workflow**: Results from preflight checks should integrate with the existing `conditionalUpdateRisks` and accepted-risks mechanism to provide a unified risk management experience.
* **Non-disruptive validation**: Preflight checks must be read-only operations that do not modify cluster state or affect running workloads.
* **Flexible execution model**: Support both one-time preflight validation and continuous preflight monitoring for target releases.

Success criteria:
- Administrators can run `oc adm upgrade --preflight --to=<version>` to check compatibility.
- Preflight results appear in ClusterVersion `status` alongside other conditional update risks.
- Component maintainers can write compatibility checks into the target release, without backporting logic to earlier releases.

### Non-Goals

* **Operator-level preflight framework**: This enhancement focuses on cluster-level preflight orchestration.
    Individual operator preflight implementations are out of scope (those would be developed separately by component teams).
    * For the initial enhancement, even the cluster-level interface between the target-release CVO and the target-release operators is out of scope.
      We need rapid agreement on the interface between the user and the cluster-managing CVO, and between the cluster-managing CVO and the target-release CVO to set a solid launch pad in the initial release.
      The details of the interface bewtween the target-release CVO and target-release operators can be deferred to the target release, and we have more time to plan that out.
* **Automatic remediation**: Preflight checks identify risks but do not automatically fix configuration issues.
    Remediation remains a manual administrative task.
* **Performance impact analysis**: This enhancement identifies compatibility risks but does not assess performance impact or resource consumption changes in target releases.
* **Rollback planning**: While preflight checks may identify update risks, planning rollback strategies for failed updates is out of scope.
* **HyperShift** or **Web-console integration**: For the initial implementation, we will focus on standalone clusters, the API, and `oc`.
    Integration with HyperShift and the in-cluster web console can happen in subsequent phases.
* **External plugins**: This enhancement does not give cluster admins the ability to plug in additional checks specific to a given target version.
    They retain the ability to:
    * [Create `critical` platform alerts][create-platform-alert] which [existing checks will surface pre-update][recommend-critical-alert].
    * [Create a custom ClusterOperator with an `Upgradeable=False` condition][ClusterOperator-Upgradeable] which existing logic will propagate through to major and minor updates (`Upgradeable` does not block patch updates from x.y.z to x.y.z' within the current z stream).

## Proposal

[The accepted-risks proposal](accepted-risks.md) added `clusterversion.status.conditionalUpdateRisks` to ClusterVersion to discuss risks that the cluster is concerned about.
This gives us an existing location where we can discuss any concerns a preflight turns up.
The remaining piece, proposed in this enhancement, is a way to request a preflight for a specific target release.
We will add a new `mode` property to `spec.desiredUpdate` to mark preflight requests.

### Workflow Description

**Cluster Administrator** is responsible for managing OpenShift cluster updates and maintenance.

**Component Developer** writes OpenShift operators and defines compatibility checks for their components.

#### Requesting a Preflight Check

1. **Starting State**: A cluster administrator wants to evaluate risks for upgrading from version 4.22.0 to version 4.24.0 (skip-level update) before scheduling a maintenance window.
1. **Request Preflight Check**: Administrator uses `oc` to request a preflight check: `oc adm upgrade --mode=preflight --to 4.24.0`.
1. **CVO Processes Request**: The Cluster Version Operator detects the preflight request and:
    - Launches target CVO as a Deployment with `preflight` argument, instead of performing an actual update.
    - Uses a shared volume to share preflight results between the preflight CVO and the cluster-managing CVO.
1. **Target Release Validation**: The target release CVO (4.24.0) runs in preflight mode:
    - Examines current cluster configuration, operators, and workloads.
    - Executes compatibility checks defined by operators in the 4.24.0 release.
    - Generates risk assessment without modifying cluster state.
1. **Results Integration**: Preflight results are reported back to the running CVO and integrated into the ClusterVersion status:
    ```yaml
    status:
      conditionalUpdateRisks:
      - name: "DualStackIncompatible"
        message: "Cluster uses dual-stack networking configuration incompatible with 4.24.0."
        conditions:
        - type: Applies
          status: True
          reason: "PreflightValidation"
          message: "Risk identified during preflight check for 4.24.0."
    ```
1. **Administrator Review**: Administrator reviews risks and can either:
   - Address configuration issues before updating.
   - Accept specific risks using [the established accepted-risks workflow](accepted-risks.md).
   - Choose a different update path.

### API Extensions

#### ClusterVersion spec.desiredUpdate.mode

The existing [ClusterVersion `spec.desiredUpdate` property][ClusterVersion-desiredUpdate] would have its [`Update` type][Update-API] extended with a new `mode` property:

```go
// mode allows an update to be checked for compatibility without committing to updating the cluster.
// Allowed values are "Preflight" and omitted.
// Optional mode allows existing clients to request updates.
// Preflight mode allows clients to request preflight compatibility checks.
// +kubebuilder:validation:Enum:=Preflight;""
// +optional
mode UpdateModePolicy `json:mode,omitempty`
```

allowing preflight requests like:

```yaml
spec:
  desiredUpdate:
    mode: Preflight
    version: 5.2.0
```

### Topology Considerations

#### Hypershift / Hosted Control Planes

HyperShift is out of scope for now, as we rush to get something tech-preview for standalone.
We'll come back later and figure out how this could fit into the HostedCluster APi.

#### Standalone Clusters

Yes, standalone is the focus.

#### Single-node Deployments or MicroShift

Single-node will have the same support as standalone.
Running preflight checks will come with the usual resource cost of long-running workload.
But cluster-admins have the ability to clear `desiredUpdate` if they want to stop running preflights, and they can enable or disable preflights as they see fit, to balance the cost vs. the benefit.

MicroShift is out of scope, because it doesn't run a cluster-version operator.
I'm not sure if MicroShift has a mechanism for checking for update compatibility or conditional update issues or not.

#### OpenShift Kubernetes Engine

This functionality will be implemented in component layers that are part of the OpenShift Kubernetes Engine (OKE), so it will function there the same way it does in OCP.

### Implementation Details/Notes/Constraints

#### Requesting preflight checks

Cluster adminstrators can request preflight checks via [the new `mode` property](#clusterversion-spec-desiredupdate-mode).
The `mode` property will also be wrapped in the existing `oc adm upgrade` command, so cluster administrators can use `oc adm upgrade --mode=preflight ...` to request preflight updates.

#### Evaluating preflight checks

FIXME
When the cluster-version operator (CVO) sees a `mode: Preflight` request, it retrieves the target release pullspec in the usual way as for an update request.
But instead of launching a `version-*` Pod to retrieve release manifests from the target release (FIXME: https://github.com/openshift/cluster-version-operator/blob/83243780aed4e0d9c4ebff528e54b918d4170fd3/pkg/cvo/updatepayload.go#L189-L297), it runs the target release with `args` set to `preflight`.
This way, the old CVO doesn't need to understand the details of how to query components for preflight checks; that's all deferred to the target CVO.

When the target release CVO is invoked with the `preflight --format=preflight-v1-json` argument, it runs preflight checks, and reports the results to the cluster's running CVO via a host-mounted volume (just like the `version-*` Pod FIXME https://github.com/openshift/cluster-version-operator/blob/83243780aed4e0d9c4ebff528e54b918d4170fd3/pkg/cvo/updatepayload.go#L271-L275).
The results will be JSON.
Because they will be [propagated into `conditionalUpdateRisks`](#retrieving-preflight-check-results), we'll use that structure:

```json
{
  "format": "FIXME: media type for preflight v1 JSON",
  "preflightID": ...
  "risks": [
    {
      "name": "ConcerningThingA",
      "message": "FIXME: Upgrade can get stuck on clusters that use multiple networks together with dual stack.
      "url": "FIXME: https://issues.redhat.com/browse/SDN-3996
      "matchingRules": FIXME probably don't set this, because components may not want to write PromQL to help the current cluster check whatever the future-CVO found
      "conditions": FIXME probably don't set this.  If we return a risk, it's because we think it applies to this cluster.
      "cacheTime": TIMESTAMP?  Do components get to tell us how long the results are valid for?
    },
    ...more concerning things, if found...
  ]
}
```

When the preflight CVO Pod completes, the cluster's running CVO lifts those identified risks up into ClusterVersion's `status.conditionalUpdateRisks`, merging with risks detected via other mechanisms (the OpenShift Update Service, etc.).
It also updates `status.conditionalUpdates` to set the preflight risk names in `status.conditionalUpdates([version==checkedVersion]).riskNames` for the version that was checked.

#### Retrieving preflight check results

```yaml
  conditionalUpdateRisks:  # include every risk in the conditional updates (moved up and renamed)
  - name: ConcerningThingA
    message: Upgrade can get stuck on clusters that use multiple networks together with dual stack.
    url: https://issues.redhat.com/browse/SDN-3996
    matchingRules:
    - type: Always
    conditions:
    - status: True  # Apply=True if the risk is applied to the current cluster
      type: Applies
      reason: MatchingRule
      message: The matchingRules[0] matches
      lastTransitionTime: 2021-09-13T17:03:05Z
```

### Risks and Mitigations

What are the risks of this proposal and how do we mitigate. Think broadly. For
example, consider both security and how this will impact the larger OKD
ecosystem.

How will security be reviewed and by whom?

How will UX be reviewed and by whom?

Consider including folks that also work outside your immediate sub-project.

### Drawbacks

The idea is to find the best form of an argument why this enhancement should
_not_ be implemented.

What trade-offs (technical/efficiency cost, user experience, flexibility,
supportability, etc) must be made in order to implement this? What are the reasons
we might not want to undertake this proposal, and how do we overcome them?

Does this proposal implement a behavior that's new/unique/novel? Is it poorly
aligned with existing user expectations?  Will it be a significant maintenance
burden?  Is it likely to be superceded by something else in the near future?

## Alternatives (Not Implemented)

### One-shot checks

FIXME: text

```go
// preflight is an identifier for a preflight attempt...
// +kubebuilder:validation:FIXME""
// +optional
preflight string `json:preflight,omitempty`
```

allowing preflight requests like:

```yaml
spec:
  desiredUpdate:
    preflight: some-ID-like-a-timestamp  Maybe require a timestamp?
    version: 5.2.0
```

FIXME: results

    - status: True  # always True?
      type: Preflight
      reason: FIXME
      message: Results from preflight {ID} run {timestamp} (FIXME: live until {future timestamp}?).
      lastTransitionTime: 2021-09-13T17:03:05Z


## Open Questions [optional]

This is where to call out areas of the design that require closure before deciding
to implement the design.  For instance,
 > 1. This requires exposing previously private resources which contain sensitive
  information.  Can we do this?

## Test Plan

**Note:** *Section not required until targeted at a release.*

Consider the following in developing a test plan for this enhancement:
- Will there be e2e and integration tests, in addition to unit tests?
- How will it be tested in isolation vs with other components?
- What additional testing is necessary to support managed OpenShift service-based offerings?

No need to outline all of the test cases, just the general strategy. Anything
that would count as tricky in the implementation and anything particularly
challenging to test should be called out.

All code is expected to have adequate tests (eventually with coverage
expectations).

## Graduation Criteria

**Note:** *Section not required until targeted at a release.*

Define graduation milestones.

These may be defined in terms of API maturity, or as something else. Initial proposal
should keep this high-level with a focus on what signals will be looked at to
determine graduation.

Consider the following in developing the graduation criteria for this
enhancement:

- Maturity levels
  - [`alpha`, `beta`, `stable` in upstream Kubernetes][maturity-levels]
  - `Dev Preview`, `Tech Preview`, `GA` in OpenShift
- [Deprecation policy][deprecation-policy]

Clearly define what graduation means by either linking to the [API doc definition](https://kubernetes.io/docs/concepts/overview/kubernetes-api/#api-versioning),
or by redefining what graduation means.

In general, we try to use the same stages (alpha, beta, GA), regardless how the functionality is accessed.

[maturity-levels]: https://git.k8s.io/community/contributors/devel/sig-architecture/api_changes.md#alpha-beta-and-stable-versions
[deprecation-policy]: https://kubernetes.io/docs/reference/using-api/deprecation-policy/

**If this is a user facing change requiring new or updated documentation in [openshift-docs](https://github.com/openshift/openshift-docs/),
please be sure to include in the graduation criteria.**

**Examples**: These are generalized examples to consider, in addition
to the aforementioned [maturity levels][maturity-levels].

### Dev Preview -> Tech Preview

- Ability to utilize the enhancement end to end
- End user documentation, relative API stability
- Sufficient test coverage
- Gather feedback from users rather than just developers
- Enumerate service level indicators (SLIs), expose SLIs as metrics
- Write symptoms-based alerts for the component(s)

### Tech Preview -> GA

- More testing (upgrade, downgrade, scale)
- Sufficient time for feedback
- Available by default
- Backhaul SLI telemetry
- Document SLOs for the component
- Conduct load testing
- User facing documentation created in [openshift-docs](https://github.com/openshift/openshift-docs/)

**For non-optional features moving to GA, the graduation criteria must include
end to end tests.**

### Removing a deprecated feature

- Announce deprecation and support policy of the existing feature
- Deprecate the feature

## Upgrade / Downgrade Strategy

If applicable, how will the component be upgraded and downgraded? Make sure this
is in the test plan.

Consider the following in developing an upgrade/downgrade strategy for this
enhancement:
- What changes (in invocations, configurations, API use, etc.) is an existing
  cluster required to make on upgrade in order to keep previous behavior?
- What changes (in invocations, configurations, API use, etc.) is an existing
  cluster required to make on upgrade in order to make use of the enhancement?

Upgrade expectations:
- Each component should remain available for user requests and
  workloads during upgrades. Ensure the components leverage best practices in handling [voluntary
  disruption](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/). Any exception to
  this should be identified and discussed here.
- Micro version upgrades - users should be able to skip forward versions within a
  minor release stream without being required to pass through intermediate
  versions - i.e. `x.y.N->x.y.N+2` should work without requiring `x.y.N->x.y.N+1`
  as an intermediate step.
- Minor version upgrades - you only need to support `x.N->x.N+1` upgrade
  steps. So, for example, it is acceptable to require a user running 4.3 to
  upgrade to 4.5 with a `4.3->4.4` step followed by a `4.4->4.5` step.
- While an upgrade is in progress, new component versions should
  continue to operate correctly in concert with older component
  versions (aka "version skew"). For example, if a node is down, and
  an operator is rolling out a daemonset, the old and new daemonset
  pods must continue to work correctly even while the cluster remains
  in this partially upgraded state for some time.

Downgrade expectations:
- If an `N->N+1` upgrade fails mid-way through, or if the `N+1` cluster is
  misbehaving, it should be possible for the user to rollback to `N`. It is
  acceptable to require some documented manual steps in order to fully restore
  the downgraded cluster to its previous state. Examples of acceptable steps
  include:
  - Deleting any CVO-managed resources added by the new version. The
    CVO does not currently delete resources that no longer exist in
    the target version.

## Version Skew Strategy

How will the component handle version skew with other components?
What are the guarantees? Make sure this is in the test plan.

Consider the following in developing a version skew strategy for this
enhancement:
- During an upgrade, we will always have skew among components, how will this impact your work?
- Does this enhancement involve coordinating behavior in the control plane and
  in the kubelet? How does an n-2 kubelet without this feature available behave
  when this feature is used?
- Will any other components on the node change? For example, changes to CSI, CRI
  or CNI may require updating that component before the kubelet.

## Operational Aspects of API Extensions

Describe the impact of API extensions (mentioned in the proposal section, i.e. CRDs,
admission and conversion webhooks, aggregated API servers, finalizers) here in detail,
especially how they impact the OCP system architecture and operational aspects.

- For conversion/admission webhooks and aggregated apiservers: what are the SLIs (Service Level
  Indicators) an administrator or support can use to determine the health of the API extensions

  Examples (metrics, alerts, operator conditions)
  - authentication-operator condition `APIServerDegraded=False`
  - authentication-operator condition `APIServerAvailable=True`
  - openshift-authentication/oauth-apiserver deployment and pods health

- What impact do these API extensions have on existing SLIs (e.g. scalability, API throughput,
  API availability)

  Examples:
  - Adds 1s to every pod update in the system, slowing down pod scheduling by 5s on average.
  - Fails creation of ConfigMap in the system when the webhook is not available.
  - Adds a dependency on the SDN service network for all resources, risking API availability in case
    of SDN issues.
  - Expected use-cases require less than 1000 instances of the CRD, not impacting
    general API throughput.

- How is the impact on existing SLIs to be measured and when (e.g. every release by QE, or
  automatically in CI) and by whom (e.g. perf team; name the responsible person and let them review
  this enhancement)

- Describe the possible failure modes of the API extensions.
- Describe how a failure or behaviour of the extension will impact the overall cluster health
  (e.g. which kube-controller-manager functionality will stop working), especially regarding
  stability, availability, performance and security.
- Describe which OCP teams are likely to be called upon in case of escalation with one of the failure modes
  and add them as reviewers to this enhancement.

## Support Procedures

Describe how to
- detect the failure modes in a support situation, describe possible symptoms (events, metrics,
  alerts, which log output in which component)

  Examples:
  - If the webhook is not running, kube-apiserver logs will show errors like "failed to call admission webhook xyz".
  - Operator X will degrade with message "Failed to launch webhook server" and reason "WebhookServerFailed".
  - The metric `webhook_admission_duration_seconds("openpolicyagent-admission", "mutating", "put", "false")`
    will show >1s latency and alert `WebhookAdmissionLatencyHigh` will fire.

- disable the API extension (e.g. remove MutatingWebhookConfiguration `xyz`, remove APIService `foo`)

  - What consequences does it have on the cluster health?

    Examples:
    - Garbage collection in kube-controller-manager will stop working.
    - Quota will be wrongly computed.
    - Disabling/removing the CRD is not possible without removing the CR instances. Customer will lose data.
      Disabling the conversion webhook will break garbage collection.

  - What consequences does it have on existing, running workloads?

    Examples:
    - New namespaces won't get the finalizer "xyz" and hence might leak resource X
      when deleted.
    - SDN pod-to-pod routing will stop updating, potentially breaking pod-to-pod
      communication after some minutes.

  - What consequences does it have for newly created workloads?

    Examples:
    - New pods in namespace with Istio support will not get sidecars injected, breaking
      their networking.

- Does functionality fail gracefully and will work resume when re-enabled without risking
  consistency?

  Examples:
  - The mutating admission webhook "xyz" has FailPolicy=Ignore and hence
    will not block the creation or updates on objects when it fails. When the
    webhook comes back online, there is a controller reconciling all objects, applying
    labels that were not applied during admission webhook downtime.
  - Namespaces deletion will not delete all objects in etcd, leading to zombie
    objects when another namespace with the same name is created.

## Infrastructure Needed [optional]

Use this section if you need things from the project. Examples include a new
subproject, repos requested, github details, and/or testing infrastructure.

[ClusterOperator-Upgradeable]: https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/updating_clusters/understanding-openshift-updates-1#understanding_clusteroperator_conditiontypes_understanding-openshift-updates
[ClusterVersion-desiredUpdate]: https://github.com/openshift/api/blob/6fb7fdae95fd20a36809d502cfc0e0459550d527/config/v1/types_cluster_version.go#L56-L81
[create-platform-alert]: https://docs.redhat.com/en/documentation/monitoring_stack_for_red_hat_openshift/4.20/html/managing_alerts/managing-alerts-as-an-administrator#creating-new-alerting-rules_managing-alerts-as-an-administrator
[recommend-critical-alert]: https://github.com/openshift/oc/blob/345800dc3c4164fbca313c1cbfb383f262547903/pkg/cli/admin/upgrade/recommend/alerts.go#L109-L124
[Update-API]: https://github.com/openshift/api/blob/6fb7fdae95fd20a36809d502cfc0e0459550d527/config/v1/types_cluster_version.go#L704-L763
