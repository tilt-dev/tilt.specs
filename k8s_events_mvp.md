# K8S Events

### Problem

Users regularly have configuration / cluster issues that show up as "events" in their k8s objects, but are not surfaced in any way by Tilt. This wastes users time and is a common contradiction to our pitches along the lines of "your system knows what the error is, and we dig it out for you" or "stop playing 20 questions with kubectl"

### Solution

For now, we'll simply watch events and log them if their type is not "Normal" ((current possible types are "Normal" and "Warning")[https://github.com/windmilleng/tilt/blob/38b55952ab972efd330a1adaac710540531db288/vendor/k8s.io/api/core/v1/types.go#L4667]).

In the future, we will probably want to draw attention to them in the UI, as we do for errors and restarts. For now, we're just taking the first step down that path and logging things.

This spec is focused on the MVP of getting k8s events into logs.

### Implementation Details

Here's a sample event on a Tilt deployment of servantes' fe service:
```
(*v1.Event)(0xc000cdf400)(&Event{ObjectMeta:k8s_io_apimachinery_pkg_apis_meta_v1.ObjectMeta{Name:matt-fe-649447bf8.159eedef32f08d04,GenerateName:,Namespace:default,SelfLink:/api/v1/namespaces/default/events/matt-fe-649447bf8.159eedef32f08d04,UID:6da2ea3d-773d-11e9-8112-025000000001,ResourceVersion:929420,Generation:0,CreationTimestamp:2019-05-15 14:15:32 -0400 EDT,DeletionTimestamp:<nil>,DeletionGracePeriodSeconds:nil,Labels:map[string]string{},Annotations:map[string]string{},OwnerReferences:[],Finalizers:[],ClusterName:,Initializers:nil,ManagedFields:[],},InvolvedObject:ObjectReference{Kind:ReplicaSet,Namespace:default,Name:matt-fe-649447bf8,UID:c9bcc82c-773c-11e9-8112-025000000001,APIVersion:extensions,ResourceVersion:929414,FieldPath:,},Reason:SuccessfulDelete,Message:Deleted pod: matt-fe-649447bf8-77hgq,Source:EventSource{Component:replicaset-controller,Host:,},FirstTimestamp:2019-05-15 14:15:32 -0400 EDT,LastTimestamp:2019-05-15 14:15:32 -0400 EDT,Count:1,Type:Normal,EventTime:0001-01-01 00:00:00 +0000 UTC,Series:nil,Action:,Related:nil,ReportingController:,ReportingInstance:,})
```

#### Mapping an event to a manifest name

The chief complexity here is that when we get an event, we want to know which manifest's log it should appear in.
We inject into all deployed objects a `tilt-manifest` label that tells us its manifest. This does not show up in the event. Thus, we need to get from the event to its associated k8s object, so we can check its labels. There are two fields we could use for this: `UID` and `ObjectReference`. Tilt does not currently store the appropriate information to determine the manifest from either of these values.

We've discussed, for other purposes, changing our `kubectl apply` to use `-ojson` to get UIDs back for the objects we create. Unfortunately, that doesn't suffice for this case. The event is on the ReplicaSet created by the Deployment. Running `kubectl apply -ojson` gives us the Deployment's UID, but not the ReplicaSet's.

I think we want something like a `map[model.UID]model.ManifestName` or `map[v1.ObjectReference]model.ManifestName` that we could use to determine to which manifest we should log an Event.

##### Option 1 - watch all events and keep the map up-to-date
Ideally, we could just watch all objects and update the map as they're added/deleted, but that doesn't seem to be possible via the k8s api (e.g., [1](https://github.com/kubernetes/kubernetes/pull/57792), [2](https://github.com/kubernetes/kubernetes/issues/1685)).

We could set up individual watches for all object types to get the same effect. Maybe we'll want to do that eventually, but it feels dirty. This means: 1) getting a list of the potential object types (and, really, we should also set up a watcher on the defined object types to ensure we pick up any new CRDs), 2) looping over those and creating a watcher on each.

##### Option 2 - lazily populate the map
Instead, we could just treat the map as a cache, and perform a lookup every time we get an unknown UID/ObjectReference. This effectively means we're making a query for every k8s object in the cluster that has an event (whether that object is Tilt-related or not). It's possible in the majority of cases this is fine. We could also potentially batch this later if needed. This also adds latency to logging events (I haven't tested, but I'd expect it to be less than a second, so not the end of the world).

#### UID vs ObjectReference
The event object has two fields we could use to figure out which manifest it belongs to: UID and ObjectReference.

UID seems more straightforward to deal with, but I have been unable to find an API that lets one go from UID -> k8s object.

If I cannot locate one, then it doesn't seem too much worse to instead look up the object by ObjectReference (i.e., kind/namespace/name/apiversion).

#### CRDs
Pods created by CRDs do not have tilt-manifest labels, which complicates mapping their events back to a manifest. We could potentially follow the owner UID chain up until we hit an object that does have a tilt-manifest label. If this turns out to be cheap, we can do it in this first pass. If not, then it seems worth just punting on supporting CRDs as part of the first pass of this feature.

Another possible avenue is checking extra_pod_labels, but that seems much more complicated and not worth considering for this first pass.

#### Or only put them in the global log?
It seems like the vast majority of the complexity of this feature is tying events back to their Tilt manifests. If we skipped that, maybe we'd save 90% of the effort and still get 80% of the benefit.

Unfortunately, I think we still basically need to do all of that work in order to distinguish Tilt-related events from non-Tilt-related events. AFAICT, the only API that will give us Tilt-related events gives us all events, and then we're basically back to the same problem of wanting to get to the k8s object to tell if it has a tilt run id.
