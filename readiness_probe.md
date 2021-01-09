# Local Resource Readiness Probe

## Background

Kubernetes has a notion of "readiness" - a test that the server is up and ready
to receive traffic.

Tilt uses readiness to determine start order of Kubernetes services and to
display server health. But there is no equivalent concept for local
services. Local services are a popular use-case for teams migrating to
Kuberentes, because it allows them to mix local services and Kubernetes
services.

## Goals

- Let users control the start order of local resource servers
- Let users control the startup display of local resource servers
- Standardize on Kubernetes' notion of readiness and probes

## Non-Goals

- We do not want to invent a new Tilt-proprietary notion of readiness or
  dependencies. Much bigger teams have gotten stuck on this problem. (see
  https://github.com/docker/compose/issues/4305,
  https://github.com/docker/compose/issues/374).

- Don't spend too much time thinking about what the Tiltfile API for this should
  look like. We need a bigger rethink of how to autogenerate Tiltfile APIs for
  simple struct specs, to make Tilt easier to extend. It is OK to take shortcuts
  on this for now and we can revisit it later.

## Detailed API

### Data Model

The `LocalTarget` spec should have one new field:

```
type LocalTarget struct {
  ...

  # Specifies how to probe the process for readiness
  # https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.20/#probe-v1-core
  Probe *v1.Probe
}
```

The `LocalRuntimeState` status object should also have one new field:

```
type LocalRuntimeState struct {
  ...
  Ready boolean
}
```

These fields are deliberatey chosen to replace the similar fields on
ContainerSpec and ContainerStatus in Kubernetes.

### Tiltfile API

We'll should add a new argument to local_resource that's only applicable when
coupled with a serve command:

```py
local_resource(
  'storybook',
  serve_cmd='yarn run storybook',
  readiness_probe=probe(http_get=http_get_action(port=9009)))
```

Given our renewed focus on extensibility and making Tilt more pluggable (see:
https://github.com/tilt-dev/company-private/blob/master/company-goals/vision.md),
we should be thinking through how to make the Tiltfile and Kubernetes API spec
interoperate. That might mean a system that autogenerates the function
specification from Kubernetes API specs. 

But for now, it's OK to punt on this. That might mean letting people specify
this as dictionaries or YAML for now, and revisiting this in a future
project. Or it might mean implementing a limited API for the Tiltfile function
by hand, and filling an automated implementation later.

### Engine Semantics

If no probe is specified, then the Tilt engine should mark a LocalRuntimeState as Ready=true when
the PID of the process is created.

If a probe is specified, the LocalController should start a process in the background that runs
the Kubernetes probe. Any logs from this probe should be displayed in the runtime log of the local_resource.

When the probe completes successfully, it should mark LocalRuntimeState's Ready field to true.

As much as possible, LocalController should use the Kubernetes probe
implementation. To avoid depending on kubernetes/kubernetes, it is OK to fork
this code into Tilt under a third_party or appropriately marked directory (for
licensing reasons).

Kuberentes probe implementation - https://github.com/kubernetes/kubernetes/tree/master/pkg/probe

Tilt's implementation of RuntimeStatus should be changed to look at the Ready
field of LocalRuntimeState, and return Pending if Ready is false.

## Related Documents

Docker Compose v2 Healthcheck - https://docs.docker.com/compose/compose-file/compose-file-v2/#healthcheck

Docker Compose removes Healthcheck - https://github.com/docker/compose/issues/374

Kubernetes probe conceptual doc - https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#when-should-you-use-a-readiness-probe

Kuberentes probe API doc - https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.20/#probe-v1-core
