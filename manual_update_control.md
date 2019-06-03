# Manual Update Control Spec (Proposal)
## Background
Today, Tilt updates on each save. This is frustrating:
* builds are often broken, leading to alert fatigue
* deploying a half-done version can break things worse, e.g. putting the database into an inconsistent state
* Doing a git checkout is scary, because it might freak out Tilt
* We should allow users to control when Tilt updates.

## Goals:
* User can start an update (or enqueue one at the front of the queue) for any service by performing an interaction in the UI.
* User can choose to put Tilt resources into a manual update mode.
* User can see whether there are pending file changes by default. (If we don't do this, a user could have undeployed changes and be surprised, breaking our brand promise)
* User can configure this in the Tiltfile, not the command-line, as per Tiltfile design principles
* User can set this for all resources, or configure per-resource. (This matters for projects where some services are in a language like js that expects hot reload and other are in a service like java that has more errors builds)
* User can also "Pause" Tilt in the UI. When Tilt is Paused, it won't start new updates automatically. User can manually initiate an update while Tilt is paused.
* For services that use LiveUpdate, the user can start either a full build update or a live update.
## Non-/Future Goals
### stop updates once they've started
(This would require more engine changes; we should do it at some point but it's not clear if users have builds that take long enough for this to be a big pain point)
### Approve risky changes
Some users (e.g. AdamB) have complained when we force-update otherwise immutable k8s objects (specifically StatefulSets). It'd be great if we could warn a user that we're trying to do a dangerous/forced update, and let them approve or ignore.
## Tiltfile Changes
The new function `update_mode()` sets the default update mode for resources. It accepts one argument, which must be either of the (new) constants `UPDATE_AUTO` or `UPDATE_MANUAL`, which are singleton values that are in global scope for the Tiltfile.

`k8s_resource` and `dc_resource` take an additional optional argument `update_mode` (default value: `UPDATE_AUTO`), which must be one of the above constants.

## UI Changes
* An update control to start an update of a resource (including Tiltfile). (If there is a current update running, it will put it the selected resource at the front of the queue).
* A forceful update control will do the same, unless the selected resource is using Live Update, in which case it will do an image update, not a live update.
* A pause control to toggle Pause. While Pause is on, Tilt will not queue builds automatically. At the moment the user turns off Pause, all resources with pending file edits will be enqueued.
* For a resource, you'll be able to see:
  * If it's dirty. E.g., "*" if it's dirty. But also some indication that updates on each new update.. (e.g. two symbols and it switches between the two each time you save). This way the user knows Tilt is seeing the edits.
  * Info about a relevant update. One resource could have both a current update and a pending update, e.g. In priority order, we'll show you:
    * Current update: the visual indicator that it's building
    * Pending Update:
      * If global pause, a paused+dirty indicator
      * If in queue, a queued+dirty indicator (possibly with a highlight for the next up)
      * If manual, an indication that it's dirty and waiting for user input

## Questions left to implementation
These are questions that we consider now, but don't think we can answer well and will leave to implementation.

### Call to Action for Paused/Manual resources
Paused and Manual resources with pending changes would be more helpful if the UI included a hint on how to run them. How can we fit this in a way that fits in the available space, and that's more useful than distracting?

## Alternatives Considered

### Only allow one global update mode
This is reasonable, and should be the first part we cut if this takes longer than expected.

### Stop in-progress updates
This is more work, and it's not clear if it adds much value. We can do it later.

### Allow pausing a single resource in the UI
We could allow pause to apply to one resource, not all of them. This would require more care to design, and it's not clear if it's that much better than pausing all builds, so let's not do it until we have signal it's important.

Also, if a user wants that frequently, they may want to change it in the Tiltfile. Or it's a case we don't understand yet.
