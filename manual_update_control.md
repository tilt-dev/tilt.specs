# Manual Update Control Spec (Proposal)
## Background
Today, Tilt updates on each save. This is frustrating:
* builds are often broken, leading to alert fatigue
* deploying a half-done version can break things worse, e.g. putting the database into an inconsistent state
* Doing a git checkout is scary, because it might freak out Tilt
* We should allow users to control when Tilt updates.
Goals
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
## Tiltfile Changes
The new function `update_mode()` sets the default update mode for resources. It accepts one argument, which must be either of the (new) constants `UPDATE_AUTO` or `UPDATE_MANUAL`, which are singleton values that are in global scope for the Tiltfile.

`k8s_resource` and `dc_resource` take an additional optional argument `update_mode` (default value: `UPDATE_AUTO`), which must be one of the above constants.

## UI Changes
* Pressing 'u' will start an update of the selected resource (including Tiltfile). (If there is a current update running, it will put it the selected resource at the front of the queue).
* Pressing shift+'u' will do the same, unless the selected resource is using Live Update, in which case it will do an image update, not a live update.
* Pressing spacebar will toggle Pause. While Pause is on, Tilt will not queue builds automatically. At the moment the user turns off Pause, all resources with pending file edits will be enqueued.
* The "Build" column will be renamed to "Update".
* The Update column for a Resource will show:
  * If it's dirty. E.g., "*" if it's dirty. But also some indication that updates on each new update.. (e.g. two symbols and it switches between the two each time you save). This way the user knows Tilt is seeing the edits.
  * Info about some update, in priority order:
    * Current update: "Updating (1.5s)"
    * Pending Update:
      * If global pause: "Paused"
      * If in queue: "Queue: N" (where N is the position in the queue)
      * If manual: "Manual"
    *Previous Update

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
