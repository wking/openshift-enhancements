---
title: targeted-update-edge-blocking
authors:
  - "@wking"
reviewers:
  - "@LalatenduMohanty"
  - "@sdodson"
  - "@vrutkovs"
approvers:
  - "@sdodson"
creation-date: 2020-07-07
last-updated: 2021-07-01
status: implementable
---

# Targeted Update Edge Blocking

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [x] Design details are appropriately documented from clear requirements
- [x] Test plan is defined
- [x] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [openshift-docs](https://github.com/openshift/openshift-docs/)

## Summary

This enhancement proposes a mechanism for blocking edges for the subset of clusters considered vulnerable to known issues with a particular update or target release.

## Motivation

When managing the [Cincinnati][cincinnati-spec] update graph [for OpenShift][cincinnati-for-openshift-design], we sometimes discover issues with particular release images or updates between them.
Once an issue is discovered, we [block edges][block-edges] so we no longer recommend risky updates, or updates to risky releases.

Note: as described [in the documentation][support-documentation], supported updates are still supported even if incoming edges are blocked, and Red Hat will eventually provide supported update paths from any supported release to the latest supported release in its z-stream.
And, since [docs#32091][openshift-docs-32091], [that documentation][support-documentation] also points out that updates initiated after the update recommendation has been removed are still supported.

Incoming bugs are evaluated to determine an impact statement based on [a generic template][rhbz-1858026-impact-statement-request].
Some bugs only impact specific platforms, or clusters with other specific features.
For example, rhbz#1858026 [only impacted][rhbz-1858026-impact-statement] clusters with the `None` platform which were created as 4.1 clusters and subsequently updated via 4.2, 4.3, and 4.4 to reach 4.5.
And rhbz#1957584 [only impacted][rhbz-1957584-impact-statement] clusters updating from 4.6 to 4.7 with Routes whose `spec.host` contains no dots or was otherwise an invalid domain name.

In those cases there is currently tension between wanting to protect vulnerable clusters by blocking the edge vs. wanting to avoid inconveniencing clusters which we know are not vulnerable and whose administrators may have been planning on taking the recommended update.
This enhancement aims to reduce that tension.

### Goals

* [Cincinnati graph-data][graph-data] maintainers will have the ability to block edges for a vulnerable subset of clusters.

### Non-Goals

* Exactly scoping the set of blocked clusters to those which would have been impacted by the issue.
    For example, some issues may be races where the impacted cluster set is a random subset of the vulnerable cluster set.
    Any targeting of the blocked edges will reduce the number of blocked clusters which would have not been impacted, and thus reduce the contention between protecting vulnerable clusters and inconveniencing invulnerable clusters.
* Specifying a particular update service implementation.
    This enhancement floats some ideas, but the details of the chosen approach are up to each update service's maintainers.

## Proposal

### Enhanced graph-data schema for blocking edges

[The blocked-edges schema][block-edges] will be extended with the following new properties:

* `url` (optional, [string][json-string]), with a URI documenting the blocking reason.
    For example, this could link to a bug's impact statement or knowledge-base article.
* FIXME: [also reason/message?](#metadata-to-include-when-blocking-a-conditional-request)
* `clusters` (optional, [object][json-object]), defining the subset of affected clusters.
  If any `clusters` property matches a given cluster, the edge should be blocked for that cluster.
  If `clusters` is unset, the edge is blocked for all clusters.
  * `promql` (optional, [string][json-string]), with a [PromQL][] query describing affected clusters.
    This query will be evaluated on the local cluster, so it has access to data beyond the subset that is [uploaded to Telemetry][uploaded-telemetry].
    The query should return a 0 if the update should be allowed and a 1 if the update should be blocked.
  * FIXME: [`platform` or other properties that are easier to evaluate than PromQL?](#non-promql-filters)

[The schema version][graph-data-schema-version] would also be bumped to 1.1.0, because this is a backwards-compatible change.
Consumers who only understand graph-data schema 1.0.0 would ignore the `clusters` property and block the edge for all clusters.
The alternative of failing open is discussed [here](#failing-open).

### Enhanced Cincinnati JSON representation

[The Cincinnati graph API][cincinnati-api] will be extended with a new top-level `conditionalEdges` property, with an array of conditional edge [objects][json-object] using the following schema:

* `from` (required, [string][json-string]), with the `version` of the starting node.
* `to` (required, [string][json-string]), with the `version` of the ending node.
* `qualifiers` (optional, [array][json-array], with qualifications around the recommendation.
  Each entry is an [object][json-object] with the following schema:
  * `url` (optional, [string][json-string]), with a URI documenting the issue, as described in [the blocked-edges section](#enhanced-graph-data-schema-for-blocking-edges).
  * FIXME: [also reason/message?](#metadata-to-include-when-blocking-a-conditional-request)
  * `clusters` (optional, [object][json-object]).
    If any `clusters` property matches a given cluster, the edge is not recommended for that cluster, as described in [the blocked-edges section](#enhanced-graph-data-schema-for-blocking-edges).
    If `clusters` is unset, the edge is blocked for all clusters.
    * `promql` (optional, [object][json-object]), defining the subset of affected clusters, as described in [the blocked-edges section](#enhanced-graph-data-schema-for-blocking-edges).
    * FIXME: [`platform` or other properties that are easier to evaluate than PromQL?](#non-promql-filters)

### Enhanced ClusterVersion representation

[The ClusterVersion `status`][api-cluster-version-status] will be extended with a new `conditionalUpdates` property:

```go
// conditionalUpdates contains the list of updates that are
// conditionally recommended for this cluster. Consumers interested
// in the set of updates that are actually recommended for this
// cluster should use availableUpdates. This list may be empty if no
// updates are recommended, if the update service is unavailable, or
// if an empty or invalid channel has been specified.
// +optional
conditionalUpdates []ConditionalUpdate `json:"conditionalUpdates,omitempty"`
```

The `availableUpdates` documentation will be adjusted to read:

```go
// availableUpdates contains the subset of conditionalUpdates that
// apply to this cluster.  Updates which appear in conditionalUpdates
// but not in availableUpdates may expose this cluster to known
// issues.
```

The new ConditionalUpdate type will have the following schema:

```go
// ConditionalUpdate represents an update which is recommended to some
// clusters on the version the current cluster is reconciling, but which
// may not be recommended for the current cluster.
// +k8s:deepcopy-gen=true
type ConditionalUpdate struct {
	// release is the target of the update.
	// +required
	release Release `json:"release"`

	// url contains information about this release. This URL is set by
	// the 'url' metadata property on a release or the metadata returned by
	// the update API and should be displayed as a link in user
	// interfaces. The URL field may not be set for test or nightly
	// releases.
	// +optional
	URL URL `json:"url,omitempty"`

	// FIXME: [also reason/message?](#metadata-to-include-when-blocking-a-conditional-request)

	// conditions represents the observations of the conditional update's
	// current status. Known types are:
	// * Recommended, for whether the update is recommended for the current cluster.
	// +patchMergeKey=type
	// +patchStrategy=merge
	// +listType=map
	// +listMapKey=type
	Conditions []metav1.Condition `json:"conditions,omitempty" patchStrategy:"merge" patchMergeKey:"type" protobuf:"bytes,1,rep,name=conditions"`
}
```

[ClusterVersion's `status.history` entries][api-history] will be extended with the following property:

```go
// overrides records update guards which were overriden to initiate the update.
Overrides []string `json:"overrides,omitempty"`  FIXME: or maybe just a single string?  These will be populated via Go errors; not sure how structured we want this to be.
```

### Update service support for the enhanced schema

The following recommendations are geared towards the [openshift/cincinnati][cincinnati].
Maintainers of other update service implementations may or may not be able to apply them to their own implementation.

The graph-builder's graph-data scraper should learn about [the new 1.1.0 schema](#enhanced-graph-data-schema-for-blocking-edges), and include the new properties in its blocker cache.
For each edge declared by a release image (primary metadata), the graph-builder will check the blocker cache for matching blocks.
Edges with no matching blocks are unconditionally recommended, and will be included in `edges`.
Edges with matching blocks are conditionally recommended, and will be included in `conditionalEdges`.

### Cluster-version operator support for the enhanced schema

The cluster-version operator will learn to parse [`conditionalEdges`](#enhanced-cincinnati-json-representation) into [`conditionalUpdates`](#enhanced-clusterversion-representation).
`edges` will continue to go straight into `availableUpdates`.
The operator will log an error if the same target is included in both `edges` and `conditionalEdges`, but will prefer the `conditionalEdges` entry in that case.
FIXME: maybe restructure `conditionalEdges` so duplicate entries are impossible?

Additionally, the operator will continually re-evaluate the blocking conditionals in `conditionalUpdates` and update `conditionalUpdates[].conditions` accordingly.
The timing of the evaluation and fresheness are largely internal details.
The operator can periodically poll, or use edge triggered watches, combinations of watching and polling, etc.
FIXME: [Do we need to report on recommendation freshness?](#reporting-recommendation-freshness)

To perform the PromQL request, the operator will FIXME: details about connecting to the local Thanos/Prometheus.

FIXME: [`platform` or other properties that are easier to evaluate than PromQL?](#non-promql-filters)

If there are issues evaluating a conditional update, the operator will set the `Unknown` status on the `Recommended` condition.
The operator will grow a new `warning`-level `CannotEvaluateConditionalUpdates` alert that fires if `lastTransitionTime` for any `Recommended=Unknown` condition is over an hour old.

Any `conditionalUpdates` with `Recommended=True` will have its release inserted into `availableUpdates`.

Both `availableUpdates` and `conditionalUpdates` should be sorted in decreasing SemVer order for stability, to avoid unnecessary status churn.

### Update client support for the enhanced schema

[The web-console][web-console] and [`oc`][oc] both consume ClusterVersion to present clients with a list of available updates.
With this enhancement, they will both be extended to consume [`conditionalUpdates`](#enhanced-clusterversion-representation).
When listing recommended updates, clients will list the contents of `availableUpdates`.
When listing all supported updates, clients will additionally include entries from `conditionalUpdates` with `Recommended!=True`, and include the `reason` and `message` from the `Recommended` condition alongside the supported-but-not-recommended updates.
Clients may optionally provision for reporting additional condition types, in case new types are added in the future.

### User Stories

#### Bugs which impact a subset of clusters

As described in [the *Motivation* section](#motivation), enabling things like "we'd like to block this edge for clusters born in 4.1 with the `None` platform".

Or, to use a more recent example, 4.7.4 was exposed to a number of issues:

* vSphere hostnames changing:

    ```yaml
    to: 4.7.4
    from: .*
    url: https://bugzilla.redhat.com/show_bug.cgi?id=1942207#c3
    clusters:
      platform: VSphere
    ```

* vSphere HW 14 vs. 4.7's kernel leading to cross-node networking issues:

    ```yaml
    to: 4.7.4
    from: 4\.6\..*
    url: https://bugzilla.redhat.com/show_bug.cgi?id=1935539#c21
    clusters:
      platform: VSphere  # FIXME: Or PromQL, to cover the None-but-actually-vSphere underneath case?
    ```

* Authentication operator leaking connections:

    ```yaml
    to: 4.7.4
    from: 4\.6\..*
    url: https://access.redhat.com/errata/RHBA-2021:2631
    clusters:
      platform: VSphere  # FIXME: Or PromQL, to cover the None-but-actually-vSphere underneath case?
    ```

### Risks and Mitigations

#### Stranding supported clusters

As described [in the documentation][support-documentation], supported updates are still supported even if incoming edges are blocked, and Red Hat will eventually provide supported update paths from any supported release to the latest supported release in its z-stream.
There is a risk, with the dynamic, per-cluster graph, that targeted edge blocking removes all outgoing update recommendations for some clusters on supported releases.
This risk can be mitigated in at least two ways:

* For the fraction of customer clusters that do not [opt-out of submitting Insights/Telemetry][uploaded-telemetry-opt-out], we can monitor [the existing `cluster_version_available_updates`][uploaded-telemetry-cluster_version_available_updates] to check for clusters running older versions which are still reporting no available, recommended updates.

* We can process the graph with tooling that removes all `conditionalEdges` and look for any supported versions without exit paths.

#### Malicious conditions

An attacker who compromises a cluster's [`upstream`][api-upstream] update service can already to some fairly disruptive things, like recommend updates from 4.1.0 straight to 4.8.0.
But at the moment, the cluster administrator (or whoever is writing to [ClusterVersion's `spec.desiredUpdate`][api-desiredUpdate]) is still in the loop deciding whether or not to accept the recommendation.

With this enhancement, the cluster-version operator will begin performing more in-cluster actions automatically, such as evaluating PromQL recommended by the upstream update service.
If the Prometheus implementation is not sufficiently hardened, malicious PromQL might expose the cluster to the attacker, with the simplest attacks being computationally intensive queries which cost CPU and memory resources that the administrator would rather be spending on more important tasks.
FIXME: notes from monitoring folks about hardening the PromQL engine to reduce these risks.

## Design Details

### Open Questions

#### Metadata to include when blocking a conditional request

A URI seems like the floor here.
But do we want other information as well?
For example, a reason and message so we can say:

```Text
Reason: InvalidRouteHostSupport
Message: Updating to 4.7.12 is supported, but not recommended, because your cluster contains routes with invalid spec.host.  For more details, see https://bugzilla.redhat.com/show_bug.cgi?id=1957584#c19
```

The benefit would be that folks would have some context about what they'd see if the clicked through to the detail URI.
The downside would be that we'd need to boil the issue down to a slug and sentence or two, and that might lead to some tension about how much detail to include.

#### Non-PromQL filters

Teaching the CVO to query the local Thanos/Prometheus shouldn't be too bad (openshift/origin already does this).
But to simplify things out of the gate, we may want to use `platform` or something that has less granularity, but simpler execution (the CVO would pull the Infrastructure resource and compare with the configured `platform` to decide if the edge was recommended).
Worth doing this in 1.1.0, and extending to PromQL in a backwards-compatible 1.2.0?

#### Reporting recommendation freshness

Currently [`availableUpdates`][api-availableUpdates] does not have a way to declare the freshness of its contents (e.g. "based on data retrieved from the upstream update service at `$TIMESTAMP`").
We do set the `RetrievedUpdates` condition and eventually alert if there are problems retrieving updates, and the expectation is that if we aren't complaining about being stale, we're fresh enough.
We could take the same approach with `conditionalUpdates`, but now that we also have "based on evaluations of the in-cluster state at `$TIMESTAMP`" in the mix, we may want to proactively declare the time.
On the other hand, continually bumping something that's similar to the node's `lastHeartbeatTime` is a bunch of busywork for both the cluster-version operator and the API-server.
Is the added transparency worth it?

### Test Plan

[The graph-data repository][graph-data] should grow a presubmit test to enforce as much of the new schema as is practical.
Validating any PromQL beyond "it's a string" is probably more trouble than its worth, but we should keep an eye on Telemetry to see if any deployed cluster-version operators are reporting trouble with condition evaluation.

Extending existing mocks and stage testing with data using the new schema should be sufficient for [update service support](#update-service-support-for-the-enhanced-schema).

Adding unit tests with data from a mock Cincinnati update service should be sufficient for [cluster-version operator support](#cluster-version-operator-support-for-the-enhanced-schema).

Ad-hoc testing when landing new features should be sufficient for `oc` and the web-console, although if they have existing frameworks for comparing output with mock cluster resources, that would be great too.

### Graduation Criteria

This will be released directly to GA.

#### Dev Preview -> Tech Preview

This will be released directly to GA.

#### Tech Preview -> GA

This will be released directly to GA.

#### Removing a deprecated feature

This enhancement does not remove or deprecate any features.

### Upgrade / Downgrade Strategy

The graph-data schema is already versioned.

We have [an open RFE][OTA-123] to version the Cincinnati API, but even without that, adding new optional properties (`conditionalEdges`) for new features (edges which would have previously been completely blocked) is a backwards-compatible change.

### Version Skew Strategy

Newer update services consuming older graph-data will know that they can use their 1.1.0 parser on 1.0.0 graph-data without needing to make changes.

Older update services consuming newer graph-data will know that they are missing some features unique to 1.1.0, but that they will still get something reasonable out of the data by using their 1.0.0 parser (they'll just consider all conditional edges to be complete blockers).

Newer clients talking to older update services will not receive any `conditionalEdges`, but they will understand all the data that the update service sends to them.
Older clients talking to newer update services will not notice `conditionalEdges`, so those edges will continue to be unconditionally blocked for those clients.

Newer clients consuming older ClusterVersion will not receive any `conditionalUpdates`, but they will understand all the data included in the ClusterVersion object (e.g. `availableUpdates`).
Older clients consuming newer ClusterVersion will not notice `conditionalUpdaets`, so those edges will continue to be unconditionally blocked for those clients.

## Implementation History

Major milestones in the life cycle of a proposal should be tracked in `Implementation
History.

## Drawbacks

Dynamic edge status that is dependent on cluster state makes [the graph-data repository][graph-data] a less authoritative view of the graph served to a given client at a given time, as discussed in [the *risks and mititgations* section](#risks-and-mitigations).
This is mitigated by ClusterVersion's [`status.history[].overrides'](#enhanced-clusterversion-representation), which records any cluster-version operator objections which the cluster administrator chose to override.
It is possible that cluster administrators would chose to clear that data, but it seems unlikely that they would invest significant effort in trying to cover their tracks when [the edges are supported regardless of whether they were recommended][openshift-docs-32091].

## Alternatives

### Additional data sources

The update service could switch on data scraped from Insights tarballs or other sources instead of Prometheus.
And we could extend `clusters` in future work to allow for that.
With this initial enhancement, I focused on Prometheus because it already exposes an API and provides access to lots of in-cluster data via a single [PromQL][] query string.

#### Update-service-side filtering

Instead of filtering cluster-side in the cluster-version operator, we could filter edges on the update-service side by querying [uploaded Telemetry][uploaded-telemetry] or [client-provided query parameters][cincinnati-for-openshift-request].
However, there is more data available in-cluster beyond what is uploaded as Telemetry.
And because we are [supporting edges which we do not recommend][openshift-docs-32091], we'd need to pass the reasons for not recommending those edges out to clusters anyway.
Passing enough information to make the decision completely on the cluster side is not that much more work.

### Failing open

Whether a conditional edge should be recommended for a given cluster depends on intent flowin from the graph-data maintainers, through update services, to the cluster version operator, and then being evaluated in-cluster.
That flow can break down at any point; for example, the update service may only understand graph-data schema 1.0.0, and not understand 1.1.0.
Or the cluster-version operator may have trouble connecting to the in-cluster Thanos/Prometheues.
In those situations, this enhancement proposal recommends blocking the edge.

An alternative approach would have been failing open, where "failed to evaluate the graph-data maintainer intentions" would result in recommending edges.
That would reduce the risk of leaving a cluster stuck without any recommended updates.
But evalution failures should trigger alerts, so the appropriate administrators can resolve the issue, and delaying an update until we can make a clear determination is safer than updating while we are unable to make a clear determination.

As a final safety valve for situations where recovering evaluation capability would take too long, confident cluster administrators can force through the update guard.

### Query coverage

[The `promql` proposal](#enhanced-graph-data-schema-for-blocking-edges) specifies a single query that allows the update service to distinguish clusters where the edge is recommended (the query returns 0) from clusters where the edge is not recommended (the query returns 1).
This allows the cluster-version operator to distinquish between the three states of `Recommended=True` (0), `Recommended=False` (1), or `Recommended=Unknonwn` (no result, for example because the query asked for metrics which the local Prometheus was failing to scrape).

[api-availableUpdates]: https://github.com/openshift/api/blob/33d847a4a36d200270e2e88a68dca23db8beafda/config/v1/types_cluster_version.go#L123-L130
[api-cluster-version-status]: https://github.com/openshift/api/blob/33d847a4a36d200270e2e88a68dca23db8beafda/config/v1/types_cluster_version.go#L146-L185
[api-desiredUpdate]: https://github.com/openshift/api/blob/33d847a4a36d200270e2e88a68dca23db8beafda/config/v1/types_cluster_version.go#L40-L54
[api-history]: https://github.com/openshift/api/blob/33d847a4a36d200270e2e88a68dca23db8beafda/config/v1/types_cluster_version.go#L146-L185
[api-upstream]: https://github.com/openshift/api/blob/33d847a4a36d200270e2e88a68dca23db8beafda/config/v1/types_cluster_version.go#L56-L60
[block-edges]: https://github.com/openshift/cincinnati-graph-data/tree/29e2d0bc2bf1dbdbe07d0d7dd91ee97e11d62f28#block-edges
[blocking-4.5.3]: https://github.com/openshift/cincinnati-graph-data/commit/8e965b65e2974d0628ea775c96694f797cd02b1e#diff-72977867226ea437c178e5a90d5d7ba8
[cincinnati]: https://github.com/openshift/cincinnati
[cincinnati-api]: https://github.com/openshift/cincinnati/blob/master/docs/design/cincinnati.md
[cincinnati-for-openshift-design]: https://github.com/openshift/cincinnati/blob/master/docs/design/openshift.md
[cincinnati-for-openshift-request]: https://github.com/openshift/cincinnati/blob/master/docs/design/openshift.md#request
[cincinnati-spec]: https://github.com/openshift/cincinnati/blob/master/docs/design/cincinnati.md
[cincinnati-graph-api]: https://github.com/openshift/cincinnati/blob/master/docs/design/cincinnati.md#graph-api
[graph-data]: https://github.com/openshift/cincinnati-graph-data
[graph-data-schema-version]: https://github.com/openshift/cincinnati-graph-data/tree/29e2d0bc2bf1dbdbe07d0d7dd91ee97e11d62f28#schema-version
[json-array]: https://datatracker.ietf.org/doc/html/rfc8259#section-5
[json-object]: https://datatracker.ietf.org/doc/html/rfc8259#section-4
[json-string]: https://datatracker.ietf.org/doc/html/rfc8259#section-7
[oc]: https://github.com/openshift/oc
[openshift-docs-32091]: https://github.com/openshift/openshift-docs/pull/32091
[OTA-123]: https://issues.redhat.com/browse/OTA-123
[PromQL]: https://prometheus.io/docs/prometheus/latest/querying/basics/
[PromQL-or]: https://prometheus.io/docs/prometheus/latest/querying/operators/#logical-set-binary-operators
[rhbz-1838007]: https://bugzilla.redhat.com/show_bug.cgi?id=1838007
[rhbz-1858026-impact-statement]: https://bugzilla.redhat.com/show_bug.cgi?id=1858026#c28
[rhbz-1858026-impact-statement-request]: https://bugzilla.redhat.com/show_bug.cgi?id=1858026#c26
[rhbz-1957584-impact-statement-request]: https://bugzilla.redhat.com/show_bug.cgi?id=1957584#c19
[support-documentation]: https://docs.openshift.com/container-platform/4.7/updating/updating-cluster-between-minor.html#upgrade-version-paths
[uploaded-telemetry]: https://docs.openshift.com/container-platform/4.7/support/remote_health_monitoring/showing-data-collected-by-remote-health-monitoring.html#showing-data-collected-from-the-cluster_showing-data-collected-by-remote-health-monitoring
[uploaded-telemetry-opt-out]: https://docs.openshift.com/container-platform/4.7/support/remote_health_monitoring/opting-out-of-remote-health-reporting.html
[uploaded-telemetry-cluster_version_available_updates]: https://github.com/openshift/cluster-monitoring-operator/blame/e104fcc9a5c2274646ee3ac50db2cfb7905004e4/Documentation/data-collection.md#L43-L47
[web-console]: https://github.com/openshift/console
