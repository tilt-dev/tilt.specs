# TiltRun
TiltRun the spec and status for the Tilt process.

The spec includes details about the invocation of Tilt (which Tiltfile? what mode?) and the status includes both meta information about the Tilt process (PID, start time) as well as high-level information about the resources its managing.

As it stands today, this is a singleton for a Tilt process but is also modeled to consider a future where a Tilt daemon manages multiple `TiltRun`s, which are created either via Tilt CLI or direct API interaction.

**Use Cases**
* [x] Provide debug information about what `tilt ci` is waiting on [#4298](https://github.com/tilt-dev/tilt/issues/4298)
* [x] Expose process info for Tilt server [#4283](https://github.com/tilt-dev/tilt/issues/4283)
* [ ] Allow customizing CI exit conditions [#4194](https://github.com/tilt-dev/tilt/issues/4194)

## Spec
The spec is actually not particularly interesting at the moment - most of the complexity lies in the status.

Future considerations for inclusion here would be any other useful configuration around the engine itself, particularly if useful to help debug why things seem "stuck", e.g. max parallelism.

```go
type TiltRunSpec struct {
	// TiltfilePath is the path to the Tiltfile for the run. It cannot be empty.
	TiltfilePath string `json:"tiltfilePath"`
	// ExitCondition defines the criteria for Tilt to exit.
	ExitCondition ExitCondition `json:"exitCondition"`
}

type ExitCondition string

const (
	// ExitConditionManual cedes control to the user and will not exit based on resource status.
	//
	// This is used by `tilt up`.
	ExitConditionManual ExitCondition = "manual"
	// ExitConditionCI terminates upon the first encountered build or runtime failure or after all resources have been
	// started successfully.
	//
	// This is used by `tilt ci`.
	ExitConditionCI ExitCondition = "ci"
)
```

## Status
The status object includes details about the Tilt server process (e.g. PID and start time), high-level status about the resources it's managing, and summary about its overall execution state to power `tilt ci`.
```go
// TiltRunStatus defines the observed state of TiltRun
type TiltRunStatus struct {
	// PID is the process identifier for this instance of Tilt.
	PID int64 `json:"pid"`
	// StartTime is when the Tilt engine was first started.
	StartTime metav1.MicroTime `json:"startTime"`
	// Targets are normalized representations of the servers/jobs managed by this TiltRun.
	//
	// A resource from a Tiltfile might produce one or more targets. A target can also be shared across
	// multiple resources (e.g. an image referenced by multiple K8s pods).
	Targets []Target `json:"resources"`

	// Done indicates whether this TiltRun has completed its work and is ready to exit.
	Done bool `json:"done"`
	// Error is a non-empty string when the TiltRun is Done but encountered a failure as defined by the ExitCondition
	// from the TiltRunSpec.
	Error string `json:"error,omitempty"`
}
```

### Target
Target is unfortunately a bit of an overloaded word, but it does map very close to what the Tilt engine refers to as targets internally, so it hopefully doesn't introduce a lot of confusion.

A `Tiltfile` resource can produce one or more targets. For example, a `local_resource` with no `update_cmd` but a `serve_cmd` would produce a single target, while a K8s resource could have a "build" target (`kubectl apply ...`) and "runtime" target (the actual pod execution).

Additionally, just like within the engine, a target can be associated with one or more resources. The most common (and only as today?) use case for this is a `docker_build` image that's referenced by multiple K8s pods: internally, the Tilt engine maintains a single target to avoid redundant image builds.

Currently, the engine internals try to aggregate multiple internal targets to _exactly_ two exposed states: build and runtime. As a result, some purely "internal" targets (such as image builds) might not be immediately **explicitly** visible via the API, but will be implicilty considered within their wrapping resource(s)' target states.

The forced two states (build + runtime) is also problematic because not all resources truly have both and so we often force those semantics. For example, the `Tiltfile` itself only "builds" but doesn't have a concept of runtime. The aforementioned `local_resource` can also cover all combinations: `update_cmd` but no `serve_cmd` is similar to `Tiltfile` (build-only), but you can also have a `serve_cmd` with NO `update_cmd`, which is runtime-only. It can also have both, of course.

Often the use of the world "build" itself is a misnomer or confusing, such as during a K8s deploy (`kubectl apply`), which isn't what most users would traditionally consider a build.

Lastly, while "runtime" is generally considered to be something server-like that runs indefinitely, K8s jobs are a notable exception to this rule and so require special-cased logic.

As a result, the API concept of "target" is intended to provide a minimal, normalized representation that is adequate for both use cases.

```go
// Target is a server or job whose execution is managed as part of this TiltRun.
type Target struct {
	// Name is the name of the target; this is auto-generated from Tiltfile resources.
	Name      string   `json:"name"`
	// Type is the execution profile for this resource.
	//
	// Job targets run to completion (e.g. a build script or database migration script).
	// Server targets run indefinitely (e.g. an HTTP server).
	Type TargetType `json:"type"`
	// Resources are one or more Tiltfile resources that this target is associated with.
	Resources []string `json:"resources,omitempty"`
	// State provides information about the current status of the target.
	State TargetState `json:"runtime,omitempty"`
}
```

#### TargetType
To address different semantics between targets that run indefinitely and those that run to completion, each target has a type associated that can be useful when interpreting its state.

Specifically, for the `tilt ci` use case, it will wait for all "job" targets to get to `Terminated`, while it will only wait for "server" targets to get to `Active`.

Thankfully, beyond that, the unified data model actually eliminates a "RuntimeStatus"-type enum, so there's less of a need for conditional logic in the general case.

```go
// TargetType describes a high-level categorization about the expected execution behavior for the target.
type TargetType string

const (
	// TargetTypeJob is a target that is expected to run to completion.
	TargetTypeJob TargetType = "job"
	// TargetTypeServer is a target that runs indefinitely.
	TargetTypeServer TargetType = "server"
)
```

#### TargetState
The state of the target is modeled after numerous K8s APIs as well as the new apiserver-based `Cmd` model within Tilt.

There are 3 possible states: `Waiting`, `Active`, and `Terminated`; the relevant one is populated with  details for the eponymous state and the others are nil. If all of them are nil, the target is currently inactive. For example, a resource with `auto_init=False` set will not have any populated states until the user triggers the resource (and thus its respective targets) to start.

```go
// TargetState describes the current execution status for a target.
//
// Either EXACTLY one of Waiting, Active, or Terminated will be populated or NONE of them will be.
// In the event that all states are null, the target is currently inactive or disabled and should not be expected to execute.
type TargetState struct {
	// Waiting being non-nil indicates that the next execution of the target has been queued but not yet started.
	Waiting    *TargetStateWaiting    `json:"pending,omitempty"`
	// Active being non-nil indicates that the target is currently executing.
	Active     *TargetStateActive     `json:"active,omitempty"`
	// Terminated being non-nil indicates that the target finished execution either normally or due to failure.
	Terminated *TargetStateTerminated `json:"terminated,omitempty"`
}
```

##### TargetStateWaiting
```go
// TargetStateWaiting is a target that has been enqueued for execution but has not yet started.
type TargetStateWaiting struct {
    // StartTime is the time at which the target entered the waiting state.
    StartTime time.Time `json:"startTime"`
    // Reason is a brief description of why the target has not yet started execution.
    Reason string `json:"reason"`
}
```

**`Reason` (FUTURE)**
The most useful thing we can expose here is _why_ the target is waiting - that is, why it has not yet started execution (e.g. waiting for a dependency, parallelism limits).

This might not be possible to determine consistently currently, so there might be cases with a generic/unknown reason in the interim.

This is also not _critical_ to initial use cases (although potentially useful for debugging a hung `tilt ci` run), so we might omit it for the moment.

##### TargetStateActive
```go
// TargetStateActive is a target that is currently running but has not yet finished.
type TargetStateActive struct {
	// StartTime is when execution began.
	StartTime metav1.MicroTime `json:"startTime"`
}
```

**`TriggerReason` (FUTURE)**
* What caused this to run (e.g. watched file change)?
  * We don't currently propagate this info through the pipeline fully enough to consistently/accurately have it at the point we need it; it's also not critical to initial use cases
  * It also might make sense for specific API types to provide this instead so they can include even more details (e.g. the actual list of files that changed)


##### TargetStateTerminated
```go
// TargetStateTerminated is a target that finished running, either because it completed successfully or
// encountered an error.
type TargetStateTerminated struct {
	// StartTime is when the target began executing.
	StartTime metav1.MicroTime `json:"startTime"`
	// FinishTime is when the target stopped executing.
	FinishTime metav1.MicroTime `json:"finishTime"`
	// Error is a non-empty string if the target encountered a failure during execution that caused it to stop.
	//
	// For targets of type TargetTypeServer, this is always populated, as the target is expected to run indefinitely.
	Error string `json:"error,omitempty"`
}
```
