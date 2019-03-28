# Problems

1. The way Tilt produces resources is confusing. It does something like iterate over images that are found in k8s yaml, joins those with any builds that have been defined with the same names, and then creates resources with names taken from the last segments of the image names. I'm not entirely sure that's accurate, and I'm not sure how closely it'd match another Windmill engineer's description. There's also [this GH issue](https://github.com/windmilleng/tilt/issues/1106).
2. Because images are the key to Tilt resources, we run into problems when multiple K8S entities use the same image (Sam Hagan and Dex have both reported this). We end up throwing a few deployments together into a single Tilt resource, making for confusing and/or incorrect pod status and logs.
3. Keying k8s_resource by image name feels weird, especially in the case of multiple images in one entity.

# Solution

1. Define a "workload" as a thing that gets turned into a Tilt resource. Practically, this is anything that we want to have its own runtime status and/or runtime log. For k8s, this is, e.g., a Deployment, a Job, or a CRD that's been declared via k8s_kind.
2. Rewrite Tiltfile assembly logic to create Resources from workloads in the k8s input and then go looking for their built images.
3. Introduce the notion of k8s entity identifiers, of the form `(<name>(:<kind>(:<namespace>(:<group)?)?)?`.
    1. A k8s entity identifier _i_ _matches_ a k8s entity _e_ if all specified components of the _i_ match the corresponding fields of _e_.
    2. A k8s entity identifier _i_ _uniquely matches_ a k8s entity _e_ if of all defined entities, _i_ matches _e_ and only _e_.
4. Default resource names for workloads will be the shortest k8s entity identifier that uniquely matches those workloads among workloads.
    1. e.g., if our only workloads are deployments named `foo` and `bar`, our resources will be named `foo` and `bar`.
    2. If our only workloads are a deployment named foo and a cronjob named foo, our resources will be named `foo:deployment` and `foo:cronjob`.
    3. If we have both a deployment and a service named `foo`, the resource will be named `foo`, because for default naming purposes, we only care about uniquely matching among workloads, and a service is not a workload.
5. Change k8s_resource:
    1. No longer takes an image.
    2. Identifies the workload via an entity identifier. If the entity identifier does not uniquely match an entity given to `k8s_yaml`, that's a Tiltfile load error.

    ```
    def k8s_resource(name: str, entity: str, new_name: str, ...) # other existing options not included here

    Exactly one of `name` and `entity` must be specified. If both are specified, Tiltfile loading fails.

    Args:
      name: identifies which resource's options to set. Must equal the name of a resource (whether from default name or from `workload_to_resource_function`, described below)
      entity: identifies which workload's resource's options to set. Must uniquely identify a workload.
      new_name: renames the resource identified by `name` or `entity`
    ```

    e.g., if we have both a deployment and a cronjob named `vigoda`:
    ```
    # name the deployment's resource "vigoda" and the cronjob's "cron"
    k8s_resource(entity='vigoda:deployment', new_name='vigoda')
    k8s_resource(entity='vigoda:cronjob', new_name='cron')

    # same as above
    k8s_resource(entity='vigoda:deployment:default', new_name='vigoda')
    k8s_resource(entity='vigoda:cronjob', new_name='cron')

    # same as above, but using name instead of entity
    k8s_resource(name='vigoda:deployment', new_name'vigoda')
    k8s_resource(name='vigoda:cronjob', new_name='cron')

    # error, since "vigoda" does not uniquely match a workload entity
    k8s_resource(entity='vigoda', port_forwards=10000)

    # error, since there's no resource named "vigoda" (only "vigoda:deployment" and "vigoda:cronjob")
    k8s_resource(name='vigoda', new_name='foo')

    # error, since there's already an entity named "vigoda:cronjob"
    k8s_resource(entity='vigoda:deployment', new_name='vigoda:cronjob')
    ```
6. For servantes, this results in resources named, e.g., "matt-vigoda". We don't want to have to duplicate naming logic to call k8s_resource. Additionally, we sometimes want to group multiple k8s entities into one resource (e.g., a Deployment with its Service). To address these, we'll add a new directive, `workload_to_resource_function` (suggestions welcome) that takes a function to translate entities into resource names, e.g.:

    ```
    def resource_name(name, kind, namespace, group):
    	return name.replace("matt-", "")

    workload_to_resource_function(resource_name)

    # now, e.g., "matt-vigoda"'s auto-assigned resource name will be "vigoda"
    ```

    1. If this results in two workloads having the same resource name, Tiltfile loading fails.
    2. This does not currently interact with uniquifying at all. We assume the user will generate unique names and do not try to add kind/namespace/group if there are conflicts. We could change `workload_to_resource_function` to take a func that returns name/kind/namespace/group and then apply normal uniquifying to the result, but that seems needlessly complicated for now, and not hard to add later.

7. If multiple workloads end up grouped to the same resource, that's a Tiltfile load error.
    1. This is sufficiently complicated to support and of sufficiently questionable value that it's not worth sorting out at the moment.

# Open questions

1. Backwards compatibility
2. Which components of entity identifiers are case sensitive? (not really relevant to approval, but needs to be figured out).
    1. From quick prodding at kubectl, it looks like only `name` is case-sensitive, and we should make everything else lower-case / case-insensitive.
