# Enable/Disable Resource API Spec

## Background

Tilt currently has a system for changing the arguments to `tilt up` at
runtime. We usually call it `tilt args`.

https://docs.tilt.dev/tiltfile_config.html

Most teams that use this system are using it for enabling and disabling sets of
services, as demonstrated in this example:

https://docs.tilt.dev/tiltfile_config.html#run-a-defined-set-of-services

This helps save CPU when you only need to run a subset of your services.

A number of teams have built wrappers around `tilt args`. This has helped us to
learn a lot about the ergonomics that people want, and the pains of the current
system. We have mocks and a pretty good idea of how we want this feature to behave.

## Primary Goal

Suggest a few data models for the Spec and Status of disabled services.

Discuss their pros and cons.

Make a decision on which data model we want to try implementing for v1 of this feature.

## Secondary Goals

### API Consistency

We see the future of Tilt extensibility as the API Server and Kubernetes
operator pattern.

What that means: there's an HTTP server that allows you can read, watch, and
write data models. Long-term, to add a new feature to Tilt, you will add a new
resource/controller pair.

Users should be able to disable services from either the CLI or the
Tiltfile. Implementers should not need to implement custom API endpoints.

Future resource types should be able to support whatever enable/disable model is
described here.

### Decouple Enable/Disable Services from Tiltfile Execution

The current `tilt args` system is tightly integrated with Tiltfile execution. If
you change the args, Tilt needs to re-execute the Tiltfile to even determine
what changed.

This quickly leads to circular dependencies (e.g., code at the beginning of the Tiltfile
that wants to know what services will be enabled at the end of the Tiltfile).

### No New Tiltfile State

Per above, we want the Tiltfile API to converge towards Kubernetes API conventions.

We do NOT think that adding special hooks and extension points to the Tiltfile
execution model is the right answer.  We don't want special Tiltfile data flow or overrides.

We may have tiltfile functions that are auto-generated from API objects. We may
also change existing Tiltfile functions to add new fields to API objects that
they assemble.

## Non-Goals

### User Interfaces

The user interface for disabling a service is beyond the scope of this doc.

### Detailed API Specifications

Although this doc contains some Go structs, please don't interpret those Go
structs as detailed API specifications!

They're mainly intended to be illustative.

For this doc, we're more interested in broad categories of solutions rather than
exact field names. We expect to revise field names as we implement the feature.

## Possible Approaches

The approaches here aren't necessarily mutually exclusive.

### DisableOnSpec / DisableOnStatus

If an API object can be disabled, then that API object should have two new
fields for processing "disable" triggers: `spec.disableOn` and
`status.disableOn`.

Here's what that might look like:

```go
type KubernetesApplySpec struct {
  ...
  
  DisableOn *DisableOnSpec
}

type KubernetesApplyStatus struct {
  ...
  
  DisableOn *DisableOnStatus
}

type DisableOnSpec struct {
  UIButtons []*UIButton
}

type DisableOnStatus struct {
  Disabled bool
  LastUpdateTime metav1.Time
  Reason string
}
```

To disable a resource, the UI POSTs a status update to the UIButton. The
KubernetesApply controller consumes that status update and toggles the
enabled/disabled state.

The UIResource controller aggregates all the DisableOnStatus fields for a resource,
and presents that state to the UI.

```go
type UIResourceStatus struct {
  Enabled UIEnableState
}

type (
  UIEnableStateOn UIEnableState = iota
  UIEnableStatePartial
  UIEnableStateOff
)
```

In this model, there's NO source of truth in the model of whether a resource is
enabled or disabled.  It's only triggered with a toggle.  The source of truth is
internal to the controller. But interfaces can still get the "result" of being
disabled from the UIResource aggregator.

The individual controllers get to decide how to process those toggles. If you
click the button twice before the controller responds, the controller gets to
decide if that counts as one toggle or two toggles.

NB: This might be a good thing! For resources where it takes a long time to
start or stop a service, you probably WANT the controller to make smart
decisions about "queued up" toggles.

A slight variant on this idea: `DisableStatus` doesn't explicitly model whether
the resource is disabled. Instead, we just see the resource has been terminated
with a reason "Disabled". Different types of resource might model this in
different ways (for example, Cmd might have it be a propery of its
CmdStateTerminated and CmdStatePending fields.)

### DisableSource

If an API object can be disabled, then that API object should have three new
fields: `Disabled`, `DisableSource`, and `DisableStatus`. `Disabled` and
`DisableSource` points at the source of truth data model for whether the
resource is disabled.

Here's what that might look like:

```go
type KubernetesApplySpec struct {
  ...
  
  Disabled *bool
  DisableSource *DisableSource
}

type KubernetesApplyStatus struct {
  ...
  
  DisableStatus *DisableStatus
}

type DisableSource struct {
  ConfigMap *ConfigMapDisableSource
}

type ConfigMapDisableSource struct {
  // The name of the ConfigMap
  Name string
    
  // The key where the enable/disable state is stored.
  Key string
  
  // Specify whether the ConfigMap must be defined.
  Optional *bool
}

type DisableStatus struct {
  Disabled bool
  LastUpdateTime metav1.Time
  Reason string
}
```

The DisableStatus for a set of objects would be aggregated into UIResource like
in the previous proposal.

There's some prior art here: I've borrowed this pattern from Kubernetes, where
you can specify the environment variables for a container with `Env` and `EnvFrom`.

https://pkg.go.dev/k8s.io/api/core/v1#Container

You could imagine starting out by just implementing one of the fields
(either `Disabled` or `DisableSource` would be fine).

For the initial implementation, I would have the Tiltfile create ConfigMaps for
each UIResource, and wire up the each object's DisableSource to the correct
ConfigMap.

The downside of this approach is that, if you want the UI to be able to disable a resource,
what API does it call and how does it know how to call it? Does it follow DisableSource
to the ConfigMap, then update the ConfigMap? Or does it support a few common conventions?

I could imagine a lot of complexity on the UI side to figure out which property
to update for each type of object.

### Resource Selector

We add a brand new type of object to our system: `Resource`. A `Resource` groups
together a set of objects by label. The closest analogue in the Kubernetes ecosystem
is a Service or an Ingress.

```go
type ResourceSpec struct {
  Selectors []ResourceSelector
  
  Disabled bool
}

// Selects a group of resources
type ResourceSelector struct {
  Group string
  Version string
  Kind string
  
  // +optional
  Labels map[string]string
  
  // +optional
  Name string
}
```

There's a direct 1:1 correspondence between a `Resource` model and the
`UIResource` that exists today.

A variant on this proposal is that we merge `Resource` and `UIResource` into a
single object where the Spec has the selector and disable state, and the Status
has all the aggregate status of those objects.

When the UI wants to disable a Resource, it updates the spec of the Resource
object.

If the Resource is disabled, the Resource controller watches for objects that
match its selector, and immediately deletes them whenever they're created.

One interesting property of this proposal is individual API object controllers
don't need to implement disable functionality. If they handle deletes properly,
then a delete is equivalent to a disable.

If an object isn't targeted by a resource selector, then there's no way to
disable it from the UI (unless we allow the UI to delete every object type.)

The `Resource` object fills in a piece of the API that's currently missing: the
`Manifest`, or a mechanism for modelling a group of API objects. We don't really
have that right now in the API server. But I'm not sure if this is the right time
to introduce it.

NB: We currently don't have much experience with the Kubernetes tooling for
dynamic clients (e.g., where we specify a Kind by string, and we dynamically
look up the HTTP APIs to call). So there might be pitfalls here that I'm not
aware of.

### Build Graph

We add a brand new type of object to our system: `BuildGraph`. A `BuildGraph`
coordinates the ordering between a set of objects.

The closest analogue in the Kubernetes ecosystem is an Argo Workflow. And, as
you would expect, an Argo Workflow has a notion of `suspend`.

https://argoproj.github.io/argo-workflows/examples/#suspending

where you can specify how to suspend a set of tasks.

The `BuildGraph` object would replace the current Build Controller in Tilt,
which is the legacy subscriber that kicks off image builds and Kubernetes
deploys.

Here's what this might look like:

```go
type BuildGraphSpec struct {
  Tasks []BuildGraphTask
  
  Disabled bool
}


type BuildGraphTask struct {
  Group string
  Version string
  Kind string
  Name string
}
```

In order for this to work, we would need some protocol for a BuildGraph
controller to communicate to controllers that it's taking over the build
sequence. And that they should only start building when the BuildGraph tells
them it's OK to do so.

To start, this could simply be a "paused" annotation. Each API controller would
have to know about and support this annotation. They would each know that they
have to yield some control to the BuildGraph.

A nice property of a BuildGraph is that it gives us a clear place to implement
resource dependencies down the line. And it allows future experiments with different
build graph implementations and resource dependency implementations.

The downside of this approach is that it's VERY different from existing objects in the API server.
We simply don't have objects that pre-empt other objects' controllers. 

## Overall Thoughts

I think any of the above proposals could work. 

I like the DisableOn trigger the most.

If you model "Disabled" as explicit state, people are going to want different
ways to override that state (e.g., if you can set the Disabled state from two
different ConfigMaps, which one wins?). You start needing booleans in YAML to
express how those state variables combine. And they're going to have different
opinions about how long that state persists (e.g., if I reload the Tiltfile,
does that reset the disabled state in the YAML?).

If the primary operation is not "is it disabled?" but rather "how do I toggle
whether it's disabled?", that dodges those problems. It's probably a better
thing for our API to focus on.

## Miscellaneous Questions

### Disable States

All the above examples assume that a single object being disabled is a binary
state.

I could easily imagine four states:

- Disabled
- Enabled
- Stopping (was Enabled, transitioning to Disabled)
- Starting (was Disabled, transitioning to Enabled)

It's also worth pointing out the general-purpose Kubernetes convention to avoid
`bool` fields. 

https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md#primitive-types


