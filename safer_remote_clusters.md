# Safer Remote Clusters (proposal)

### Problem

Users often change their kube context, forget about it, `tilt up`, and end up
deploying to prod.

### Solutions

The following are not mutually exclusive!

#### Point users at a pure Starlark Solution

a la [what dmiller came up with](https://gist.github.com/jazzdan/d30ef73482920beb3ac9fa2fcb2c3146)

Pros:
* cheap
* flexible

Cons:
* not very discoverable
* unsafe by default

#### Require intervention when deploying over a non-Tilt-deployed entity

Pros:
* would be nice to have regardless of anything else we do

Cons:
* not clear we have a good existing UI pattern for such an intervention
* doesn't protect users from deploying non-prod resources to prod (e.g., I've `tilt up`'d both integration tests and servantes in prod this week), though that's obviously a much less dire problem than killing production

#### Require remote k8s clusters to be explicitly whitelisted

Add an `allow_k8s_contexts` Tiltfile directive. Generate a Tiltfile error unless one of:
1. the k8s context api's host is localhost
2. the k8s context's name is in a whitelist of known-local context names
3. the k8s context's name is passed to `allow_k8s_contexts`

Pros:
* pretty simple UI

Cons:
* gets kind of weird in the staging case - sometimes I want Tilt to be able to deploy there, and sometimes I don't. I'd have to edit the Tiltfile every time. This might be unavoidable.

### Proposal

Implement the last option (`allowed_remote_k8s_clusters`). Primarily because:
1. The UI for the second option is unclear.
2. I'd personally like protection from accidentally deploying servantes / integration tests to GKE.
