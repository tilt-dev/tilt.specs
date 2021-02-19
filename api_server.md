# Tilt API Server


## Data Model

### File Watch
```go
type FileWatchSpec struct {
    Glob string
    Ignores []string
}

type FileWatchStatus struct {
    LastEventTimestamp time.Time
}
```
**Option 1 - `ownerRef` + Events**

`FileWatchController` emits events whenever there's a change; controllers that created these are responsible for watching + reconciling based on them

**Option 2 - Label/Annotation Selector**

`FileWatchSpec` could include a selector and as part of its reconcile find all entities that match and update a label/annotation on them, e.g. `tilt.dev/file-watch-timestamp: 2021-02-17T10:31:00.000-0400`.

(This is similar to how `kubectl rollout restart` works.)

### Local Resource
```go
// should we split into build + serve?
//   many local resources are "build-only" (e.g. seed DB)
//   some are also serve-only (e.g. "go run ...")
// if split -> how do dependencies communicate?
type LocalResourceSpec {
    DisplayName string
    TriggerMode TriggerMode

    BuildTemplate LocalProcessSpec
    ServeTemplate LocalProcessSpec

    AllowParallel bool

    Links []string
}

type LocalResourceStatus {
    Build *struct {
        Error error
        StartTime time.Time
        FinishTime time.Time
        Reason BuildReason
    }
    Serve RuntimeStatus
}
```
1. `LocalResourceController` creates `LocalProcess` based on `BuildTemplate`
2. Watches for events on owned `LocalProcess` entities
   * **Build Process Events**
     * Updates `Build` status
     * If success, creates `LocalProcess` based on `ServeTemplate`
   * **Serve Process Events**
     * Updates `Serve` status
3. Watches for changes to `LocalResource`; if changed, delete any build/serve entities + re-create (?)

### Local Process
```go
type LocalProcessSpec struct {
    Argv []string
    Dir string
    Env map[string]string

    // make this more like K8s pod restart policy?
    Daemon bool

    ReadinessProbe *k8s.Probe
}

type LocalProcessStatus struct {
    PID int64
    StartTime time.Time
    FinishTime time.Time
    ExitCode int32
    Ready bool
}
```
* Launch process with monitor that emits eventes based on process lifecycle
* Watch for process lifecycle events + update status
* If `ReadinessProbe` defined, trigger monitor that emits events + watch for probe events to update `Ready`
* If `Daemon` and process exits, auto-restart

### K8s Resource
```go
type K8sResourceSpec {
    DisplayName string
    YAML string
    PortForwards []PortForward
    ObjectRefs []k8s.ObjectReference

    PodReadinessMode PodReadinessMode

    Links []string
}

type K8sResourceStatus {
    PodAncestorUID k8s.UID

    Pods map[k8s.PodID]*k8s.Pod
    LoadBalancers map[k8s.ServiceName]*url.URL

    DeployedUIDs []k8s.UID
    DeployedPodTemplateSpecHashes []PodTemplateSpecHash

    // this is what status looks like in Tilt today
    // I think there's probably a lot of flawed assumptions baked in
    LastReadyOrSucceededTime time.Time
    HasEverDeployedSuccessfully bool
}
```

### Images
How do images trigger dependent builds and communicate necessary info to them (e.g. ref)?

#### Docker
```go
type DockerImageSpec {
    Refs struct {
        Config string
        Local string
        Cluster string
    }

    Dockerfile string
    ...
}

type DockerImageStatus {
    Error error
    Digest string
    Tag string
}
```

#### Custom
```go
type CustomImageSpec {
    CommandTemplate LocalProcessSpec
    Tag string
    DisablePush bool
    SkipsLocalDocker bool
}

type CustomImageStatus {
    Error error
    Digest string
    Tag string
}
```
