# KEP-NNNN: Well-Known Label to Delegate EndpointSlice Control to Custom Controller

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories (Optional)](#user-stories-optional)
    - [Story 1](#story-1)
    - [Story 2](#story-2)
  - [Notes/Constraints/Caveats (Optional)](#notesconstraintscaveats-optional)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Test Plan](#test-plan)
      - [Prerequisite testing updates](#prerequisite-testing-updates)
      - [Unit tests](#unit-tests)
      - [Integration tests](#integration-tests)
      - [e2e tests](#e2e-tests)
  - [Graduation Criteria](#graduation-criteria)
  - [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy)
  - [Version Skew Strategy](#version-skew-strategy)
- [Production Readiness Review Questionnaire](#production-readiness-review-questionnaire)
  - [Feature Enablement and Rollback](#feature-enablement-and-rollback)
  - [Rollout, Upgrade and Rollback Planning](#rollout-upgrade-and-rollback-planning)
  - [Monitoring Requirements](#monitoring-requirements)
  - [Dependencies](#dependencies)
  - [Scalability](#scalability)
  - [Troubleshooting](#troubleshooting)
- [Implementation History](#implementation-history)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [Infrastructure Needed (Optional)](#infrastructure-needed-optional)
<!-- /toc -->

## Release Signoff Checklist

Items marked with (R) are required *prior to targeting to a milestone / release*.

- [ ] (R) Enhancement issue in release milestone, which links to KEP dir in [kubernetes/enhancements] (not the initial KEP PR)
- [ ] (R) KEP approvers have approved the KEP status as `implementable`
- [ ] (R) Design details are appropriately documented
- [ ] (R) Test plan is in place, giving consideration to SIG Architecture and SIG Testing input (including test refactors)
  - [ ] e2e Tests for all Beta API Operations (endpoints)
  - [ ] (R) Ensure GA e2e tests meet requirements for [Conformance Tests](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/conformance-tests.md) 
  - [ ] (R) Minimum Two Week Window for GA e2e tests to prove flake free
- [ ] (R) Graduation criteria is in place
  - [ ] (R) [all GA Endpoints](https://github.com/kubernetes/community/pull/1806) must be hit by [Conformance Tests](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/conformance-tests.md) 
- [ ] (R) Production readiness review completed
- [ ] (R) Production readiness review approved
- [ ] "Implementation History" section is up-to-date for milestone
- [ ] User-facing documentation has been created in [kubernetes/website], for publication to [kubernetes.io]
- [ ] Supporting documentation—e.g., additional design documents, links to mailing list discussions/SIG meetings, relevant PRs/issues, release notes

[kubernetes.io]: https://kubernetes.io/
[kubernetes/enhancements]: https://git.k8s.io/enhancements
[kubernetes/kubernetes]: https://git.k8s.io/kubernetes
[kubernetes/website]: https://git.k8s.io/website

## Summary

This proposal adds a new well-known label `service.kubernetes.io/endpoint-controller-name` to Kubernetes Services. This label disables the default Kubernetes EndpointSlice controller and delegates the control of EndpointSlices to a custom EndpointSlice controller.

Additionally, this KEP aims to give more flexibility for the EndpointSlice Reconciler package to allow users to select the way the pods should be converted into endpoints.

## Motivation

As of now, a service can be delegated to a custom service proxy if the label `service.kubernetes.io/service-proxy-name` is set. Introduced in [KEP-2447](https://github.com/kubernetes/enhancements/issues/2447), this allows custom service proxies to implement service in different ways to address different purpose / use-cases. However, the EndpointSlices attached to this service will still be reconciled in the same way as any other services. Addressing more purpose / use-cases, for example, different pod IP addresses, is therefore not natively possible.

Delegating EndpointSlice control would allow custom controllers to define their own criterias for pod availability, selecting different pod IPs than the pod.status.PodIPs and more. As reference implementation, and since [KEP-3685](https://github.com/kubernetes/enhancements/issues/3685), the reconciler logic used by Kubernetes can be reused by custom EndpointSlice controllers. 

### Goals

* Disable the EndpointSlice controller for services that hold the label `service.kubernetes.io/endpoint-controller-name`.
* Expose the convertion of Pods to an Endpoints from the Kubernetes EndpointSlice Reconciler.

### Non-Goals

* Change / Replace / Deprecate the existing behavior of the Kubernetes EndpointSlice controller.
* Implement new EndpointSlice controllers as part of Kubernetes.
* Modify the Service / EndpointSlice Specs.

## Proposal

#### Well-Known Label

`service.kubernetes.io/endpoint-controller-name` will be added as a well-known label applying on the Service object. When set on a service, the Endpoint, EndpointSlice, and EndpointSliceMirroring for that service will be disabled, thus, Endpoints and EndpointSlices for this service will not be created by the Kubernetes Controller Manager. 

#### Flexibility on the EndpointSlice Reconciler Package

The `Reconciler` will hold a reference to a function implementing the function type described below to allow users to implement their own version of PodToEndpoint. 

```golang
type PodToEndpoint func(pod *v1.Pod, node *v1.Node, service *v1.Service, addressType discovery.AddressType) discovery.Endpoint
```

The current `podToEndpoint`, implemented here: [endpointslice/utils.go](https://github.com/kubernetes/endpointslice/blob/v0.30.2/utils.go#L37), could be made public, so the Kubernetes EndpointSlice would use it and continue to work with the same behavior as today.

Currently used here: [endpointslice/reconciler.go](https://github.com/kubernetes/endpointslice/blob/v0.30.2/reconciler.go#L208), this function converts the pod to an endpoint by setting the TargetRef, pod readiness, PodIPs...


### User Stories (Optional)

#### Story 1

As a Cloud Native Network Function (CNF) vendor, some of my services are handled by custom service-proxies over secondary networks provided by, for example, [Multus](https://github.com/k8snetworkplumbingwg/multus-cni). IPs configured in the service and registered by the EndpointSlice controller must be only the secondary IPs provided by the secondary network provider (e.g. [Multus](https://github.com/k8snetworkplumbingwg/multus-cni), also see [KEP-3698](https://github.com/kubernetes/enhancements/issues/3698)).

Therefore, it must be possible to disable the default Kubernetes EndpointSlice Controller for certain services and re-use the EndpointSlice controller implementation with custom functions converting pods to endpoints.

### Notes/Constraints/Caveats (Optional)

N/A

### Risks and Mitigations

The existing behavior will be kept by default, and the Kubernetes EndpointSlice Controller will be disabled only when the service contains this new label. This ensures services without the label to continue to be managed as usual.

This will have no effect on other EndpointSlice controller implementations since they will not be influenced by the presence of this label.

## Design Details

### Test Plan

[x] I/we understand the owners of the involved components may require updates to
existing tests to make this code solid enough prior to committing the changes necessary
to implement this enhancement.

##### Prerequisite testing updates

##### Unit tests

/

##### Integration tests

/

##### e2e tests

- Usage of `service.kubernetes.io/endpoint-controller-name` on services
    * A service is created with the label and the service has then no endpoints.
    * A service is created without the label, the service has endpoints. Then service is updated with the label and the service has no longer any endpoints.

### Graduation Criteria

N/A

### Upgrade / Downgrade Strategy

N/A

### Version Skew Strategy

N/A

## Production Readiness Review Questionnaire

### Feature Enablement and Rollback

###### How can this feature be enabled / disabled in a live cluster?

- [ ] Feature gate (also fill in values in `kep.yaml`)
  - Feature gate name:
  - Components depending on the feature gate:
- [ ] Other
  - Describe the mechanism: setting the label `service.kubernetes.io/endpoint-controller-name` disables the Kubernetes endpointslice controller.
  - Will enabling / disabling the feature require downtime of the control
    plane? No
  - Will enabling / disabling the feature require downtime or reprovisioning
    of a node? No

###### Does enabling the feature change any default behavior?

N/A

###### Can the feature be disabled once it has been enabled (i.e. can we roll back the enablement)?

N/A

###### What happens if we reenable the feature if it was previously rolled back?

N/A

###### Are there any tests for feature enablement/disablement?

N/A

### Rollout, Upgrade and Rollback Planning

###### How can a rollout or rollback fail? Can it impact already running workloads?

N/A

###### What specific metrics should inform a rollback?

N/A

###### Were upgrade and rollback tested? Was the upgrade->downgrade->upgrade path tested?

N/A

###### Is the rollout accompanied by any deprecations and/or removals of features, APIs, fields of API types, flags, etc.?

N/A

### Monitoring Requirements

###### How can an operator determine if the feature is in use by workloads?

N/A

###### How can someone using this feature know that it is working for their instance?

- [ ] Events
  - Event Reason: 
- [x] API .status
  - Condition name: 
  - Other field: When the `service.kubernetes.io/endpoint-controller-name` label is set on a service, no Endpointslice and no Endpoint will be created but the Kubernetes Controller Manager.
- [ ] Other (treat as last resort)
  - Details:

###### What are the reasonable SLOs (Service Level Objectives) for the enhancement?

N/A

###### What are the SLIs (Service Level Indicators) an operator can use to determine the health of the service?

- [ ] Metrics
  - Metric name:
  - [Optional] Aggregation method:
  - Components exposing the metric:
- [ ] Other (treat as last resort)
  - Details:

###### Are there any missing metrics that would be useful to have to improve observability of this feature?

No

### Dependencies

###### Does this feature depend on any specific services running in the cluster?

No

### Scalability

###### Will enabling / using this feature result in any new API calls?

No

###### Will enabling / using this feature result in introducing new API types?

No

###### Will enabling / using this feature result in any new calls to the cloud provider?

No

###### Will enabling / using this feature result in increasing size or count of the existing API objects?

No

###### Will enabling / using this feature result in increasing time taken by any operations covered by existing SLIs/SLOs?

No

###### Will enabling / using this feature result in non-negligible increase of resource usage (CPU, RAM, disk, IO, ...) in any components?

No

###### Can enabling / using this feature result in resource exhaustion of some node resources (PIDs, sockets, inodes, etc.)?

No

### Troubleshooting

###### How does this feature react if the API server and/or etcd is unavailable?

N/A

###### What are other known failure modes?

N/A

###### What steps should be taken if SLOs are not being met to determine the problem?

N/A

## Implementation History

- Initial proposal: 2024-06-01

## Drawbacks

TBD

## Alternatives

### Use Annotation as Selector

Services without selectors will not get any EndpointSlice objects. Therefore, selecting pods can be done in different ways, for example, via annotation. An annotation will be used in the service to select which pods will be used as backend for this service. For example, [nokia/danm](https://github.com/nokia/danm) uses `danm.k8s.io/selector` (e.g. [DANM service declaration](https://github.com/nokia/danm/blob/v4.3.0/example/svcwatcher_demo/services/internal_lb_svc.yaml#L7)), and [projectcalico/vpp-dataplane](https://github.com/projectcalico/vpp-dataplane) uses `extensions.projectcalico.org/selector` (e.g. [Calico-VPP Multinet services](https://github.com/projectcalico/vpp-dataplane/blob/v3.25.1/docs/multinet.md#multinet-services)). To simplify the user experience, a mutating webhook could read the selector, add them to the annotation and clear them from the specs when the type of service is detected.

The custom EndpointSlice Controller will then read the annotation to select the pods targeted by the service.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: service
  annotations:
    selector: "app=a"
spec: {}
```

### Use Dummy Selector

The set of Pods targeted by a Service is determined by a selector, the labels in the selector must be included as part of the pod labels. If a dummy selector is added to the service, Kubernetes will not select any pod, the endpointslices created by Kubernetes will then be empty. To simplify the user experience, a mutating webhook could add the dummy selector when the type of service is detected.

The custom EndpointSlice Controller could read the service.spec.selector and ignore the dummy label to select pods targeted by the service.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: service
spec:
  selector:
    app: a
    dummy-selector: "true"
```

## Infrastructure Needed (Optional)

N/A
