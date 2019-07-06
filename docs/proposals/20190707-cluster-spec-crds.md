---
title: Cluster Provider Spec and Status as CRDs 
authors:
  - "@pablochacin"
reviewers:
  - "@ncdc"
  - "@vincepri"
creation-date: 2019-07-07
last-updated: 2019-07-07
status: provisional
see-also:
  - "/docs/proposals/20190610-machine-states-preboot-bootstrapping.md"
---

# Cluster Spec & Status CRDs


## Table of Contents

A table of contents is helpful for quickly jumping to sections of a proposal and for highlighting any additional information provided beyond the standard proposal template.
[Tools for generating][] a table of contents from markdown are available.

- [Title](#title)
  - [Table of Contents](#table-of-contents)
  - [Summary](#summary)
  - [Motivation](#motivation)
    - [Goals](#goals)
    - [Non-Goals](#non-goals)
  - [Proposal](#proposal)
    - [User Stories [optional]](#user-stories-optional)
      - [Story 1](#story-1)
      - [Story 2](#story-2)
    - [Implementation Details/Notes/Constraints [optional]](#implementation-detailsnotesconstraints-optional)
    - [Risks and Mitigations](#risks-and-mitigations)
  - [Design Details](#design-details)
    - [Test Plan](#test-plan)
    - [Graduation Criteria](#graduation-criteria)
      - [Examples](#examples)
        - [Alpha -> Beta Graduation](#alpha---beta-graduation)
        - [Beta -> GA Graduation](#beta---ga-graduation)
        - [Removing a deprecated flag](#removing-a-deprecated-flag)
    - [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy)
    - [Version Skew Strategy](#version-skew-strategy)
  - [Implementation History](#implementation-history)
  - [Drawbacks [optional]](#drawbacks-optional)
  - [Alternatives [optional]](#alternatives-optional)
  - [Infrastructure Needed [optional]](#infrastructure-needed-optional)

[Tools for generating]: https://github.com/ekalinin/github-markdown-toc

## Summary

In Cluster API (CAPI) v1alpha1 Cluster object contains the `ClusterSpec` and `ClusterStatus` objects. Both objects contains in turn provider-specific fields, `ProviderSpec` and `ProviderStatus` embedded as opaque [RawExtensions](https://godoc.org/k8s.io/apimachinery/pkg/runtime#RawExtension). 

This proposal outlines replacing these Provider embedded opaque fields by references to objects managed by the providers.

## Motivation

Embedding opaque provider-specific information has some disadvantages : 
- Using an embedded provider spec as a RawExtensio means its contant is not validated as part of the Cluster validation
- Embeddeing the provider spec dificults implementing the logic of the provider as a controller watching for changes in the specs.
- Embedding the provider status as part of the Cluster requires that either the generic cluster controller be responsible for pulling this state from the provider (e.g. by invoking a provider actuator) or the provider updating this field, incurring in overlapping responsabilities. 

### Goals

1. To use Kubernetes Controllers to manage the lifecycle of provider Cluster spec.
1. To validate provider-specific content as early as possible (create, update).
1. Allow cluster providers to expose the status via a status subresource

### Non-Goals/Future Work
1. To modify the generic Cluster object. Further work spliting the cluster spec into more granular elements, such as infrastructure and control plane specs (in the same line of the Machine object) should be addressed in future proposals
1. To replace the provider cluster status by alternative representations, such a a list of conditions.
1. To propose cluster state lifecycle hooks. This must be part of a future proposal that builds on these proposed changes
1. Impose the operator pattern as the implementation model for provider specific cluster creation logic. Even when this is facilitated by this proposal, the actuator model can still be used. However, the actuator should have to access the provider spec object.


## Proposal

### Data Model changes

```go
type ProviderSpec struct
```
- **To remove**
    - **Value** [optional] _Superseded by SpecRef_
        - Type: `*runtime.RawExtension`
        - Description: Value is an inlined, serialized representation of the resource configuration. It is recommended that providers maintain their own versioned API types that should be serialized/deserialized from this field, akin to component config.

- **To add**
    - **SpecRef** [optional]
        - Type: `*corev1.ObjectReference`
        - Description: SpecRef is a reference to a cluster provider-specific resource that holds the details for provisioning a cluster in said provider. This reference is optional and allows the provider to implement its own cluster specification details.

```go
type ClusterStatus struct
```
- **To remove**
    - **ProviderStatus* [optional]
        - Type: `*runtime.RawExtension`
        - Description:  Provider-specific status. It is recommended that providers maintain their own versioned API types that should be serialized/deserialized from this field.

- **To add**
    - **StatusRef** [optional]
        - Type: `*corev1.ObjectReference`
        - Description: SpecRef is a reference to a cluster provider-specific resource that holds the status of the cluster in said provider. This reference is optional and allows the provider to implement its own cluster status details.

### Cluster Provisioning sequence

Moving the provider spec to a separate object makes more evident the separation of responsabilities between the generic Cluster controller and the provider's cluster controller, but also brings the need for coordination as the provider controller will likely require information from the Cluster objet during the cluster provisioning process.



```
    O                 +------------+                +------------+               +------------+
  --|--               |     API    |                |   Cluster  |               |  Provider  |
   / \                |   Server   |                | Controller |               | Controller |
    |                 +------------+                +------------+               +------------+
    | Create Cluster,        |                             |                             |
    |     Provder Spec       |                             |                             |
    |----------------------->|   New Cluster               |                             |
    |                        |---------------------------> |                             |
    |                        |   Get Provider Spec        +-+                            |
    |                        |<---------------------------| |                            |
    |                        |  +----------------------------------------+               |    
    |                        |  | IF Not seen before      | |            |               |    
    |                        |  |                         | |--+         |               |    
    |                        |  |                         | |  | Watch   |               |    
    |                        |  |                         | |  | Provider|               |    
    |                        |  |                         | |  | Spec    |               |    
    |                        |  |                         | |<-+         |               |    
    |                        |  +----------------------------------------+               |
    |                        |                            | |                            |
    |                        |  +----------------------------------------+               |    
    |                        |  | IF Provider Spec has no | |            |               |    
    |                        |  |    Owner                | |            |               |    
    |                        |  |                         | |            |               |    
    |                        |  | Set owner ref to Cluster| |            |               |    
    |                        |<-+-------------------------| |            |               |    
    |                        |  |                         | |            |               |    
    |                        |  +----------------------------------------+               |
    |                        |                            +-+                            |
    |                        |  New Provider Spec          |                             |
    |                        |-----------------------------+---------------------------->|
    |                        |                             |                            +-+
    |                        |                             |  +----------------------------------------+
    |                        |                             |  | IF Provider Spec has    | |            |
    |                        |                             |  |    Owner Ref            | |            |
    |                        |  Get cluster                |  |                         | |            |
    |                        |<----------------------------+--+-------------------------| |            |
    |                        |                             |  |                         | |--+         |
    |                        |                             |  |                         | |  |Provision|
    |                        |                             |  |                         | |  |         |
    |                        |                             |  |                         | |<-+         |
    |                        |                             |  +----------------------------------------+
```
Fig. When a Cluster object is created, the cluster controller will retrieve the Provider Spec object. If the object has not seen before, it will start watching it. Also, if the object's owner is not set, it wil set to the Cluster objec. When a Provider Spec object is created, the provider controller will check the provider spec object reference. If set, it will retrieve the cluster object.

### User Stories [optional]

#### As an infrastructure provider author, I would like to build a controller to manage provisioning clusters without being restricted to a CRUD API.

#### As an infrastructure provider author, I would like to take advantage of the kubernetes API to provide validation for provider-specific data needed to provision a machine.

### Implementation Details/Notes/Constraints [optional]

### Role of Cluster Controller
The Cluster Controller should be the only controller having write permissions to the Cluster objects. Its main responsability is to 
- Manage cluster finalizers
- Set provider spec object's OwnerReferences to Cluster
- Update Status fields from provider custom resources.

#### Cluster Controller dynamic watchers

As the provider spec data is no longer inlined in the Cluster object, the Cluster Controller needs to watch for updates to provider specific resources so it can detect events such as deletions. To achieve this, it can use controller-runtime’s `source.Informer` type and client-go’s `dynamicinformer.DynamicSharedInformerFactory`.

When the Cluster Controller reconciles a Cluster and detects a reference to a provider-specific resource, it:
- Gets a dynamic informer for the provider-specific resource.
- Starts watching this resource using the dynamic informer and a `handler.EnqueueRequestForOwner` configured with Cluster as the OwnerType (to ensure it only watches provider objects related to a CAPI cluster)

### Risks and Mitigations

## Design Details

### Test Plan

TODO

### Graduation Criteria

TODO

### Upgrade / Downgrade Strategy

TODO

### Version Skew Strategy

TODO

## Implementation History

- [x] 03/20/2019 [Issue Opened](https://github.com/kubernetes-sigs/cluster-api/issues/833) proposing the externalization of embeded provider objects as CRDs
- [x] 06/19/2009 [Discussed](https://github.com/kubernetes-sigs/cluster-api/issues/833#issuecomment-501380522) the inclusion in the scope of v1alpha2.
- [ ] Presentation of proposal to the community
- [ ] Feedback

## Drawbacks [optional]

TODO

## Alternatives [optional]

Similar to the `Drawbacks` section the `Alternatives` section is used to highlight and record other possible approaches to delivering the value proposed by a proposal.
