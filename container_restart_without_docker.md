# `container_restart` without Docker

## Problem

`live_update`'s [`container_restart` feature](https://docs.tilt.dev/api.html#api.container_restart) is currently implemented by calling
a Docker API to restart the container. Thus, it only works for users
who are using the Docker runtime (and not, e.g., cri.o or containerd).
These other runtimes do not have any equivalent API, so it's not just
a matter of simply calling the right API for the runtime.

## Solution

Allow `container_restart`'s functionality without relying on the Docker API to restart the process.

In the absence of an API, we're pretty much stuck with wrapping the entrypoint
with a script that allows us to hide the process restarts from the container
(roughly: wrap the process in a `while true` loop, so that we can kill
it and have it restart).

To accomplish this, we need these things:

1. Implement a wrapper script that handles restarting the process (already implemented [here](https://github.com/windmilleng/rerun-process-wrapper)).
2. Have Tilt put that wrapper script in any `live_update`-enabled images it
   builds.
3. Have Tilt change the entrypoint for any `live_update`-enabled images to call
   the wrapper script instead.

For the sake of this document, we'll assume (1) and (2) are straightforward and
not worth discussing.

We do not currently have a good solution for (3). Setting the container's
entrypoint to a new value is easy (and, in fact, already implemented in Tilt!).
The problem is that the new value is a function of the old value, and getting
the old value is hard.

Obvious options:
1. Calculate the entry point.
2. Run the image and observe the entry point.
3. Require the user to tell us the entry point, e.g., instead of `container_restart()`, `container_restart('/go/bin/my_app')`

#### Calculate the entry point

The command that runs in a container in a pod in k8s is determined by a
combination of: 1. Any CMD and/or ENTRYPOINT directives in the image (described
[here](https://docs.docker.com/engine/reference/builder/#understand-how-cmd-and-entrypoint-interact)).
2. Any CMD directives in the image's ancestors. 3. Any environment variables set
in the image, or its ancestors. 4. Any command or args elements in the k8s yaml
(described
[here](https://kubernetes.io/docs/tasks/inject-data-application/define-command-argument-container/#notes)).
5. Any environment variables defined in the k8s yaml, and any secrets or
configmaps referenced by those environment variables (described
[here](https://kubernetes.io/docs/tasks/inject-data-application/define-command-argument-container/#use-environment-variables-to-define-arguments)).

Note also that CMD and ENTRYPOINT can be specified in `shell` (plain string) or
`exec` (array of strings) mode, and we'd want to preserve that distinction.

I have seen no public library functions that will perform any of this logic for
us, which leaves us implementing it ourselves (including stuff like: base image
has `ENV FOO=1`, child image has `ENV BAR=${FOO}2`, `ENTRYPOINT ${BAR}`).

Aside from the technical risk of getting those details right, all of the
behaviors described above are for a specific version of Docker and Kubernetes.
I do not know if they have changed in the past or will change in the future. If
a user runs Tilt with a version of Docker or Kubernetes that has different
behavior, we're now potentially running the wrong command for them.

While it doesn't necessarily make sense to rule out ever implementing this
logic, it does seem pretty expensive and risky to do for what currently seems
to be a niche case for which there is an existing workaround.

Another option would be to implement only a subset of the logic, if we could
write guards to detect and give good error messages on cases we don't yet
support (e.g., environment variables). I'm not sure writing those guards is
significantly cheaper than implementing the proper logic itself, but it reduces
exposure to versioning risks (which I'm not sure are significant).

#### Run the image and observe the entry point
We could quickly run the container, grab argv for its process 1, and throw it
away. This seems fraught:
1. If the process has side effects, this is bad.
2. It seems confusing to the user for this ephemeral pod/container to show up.
3. We'd probably want to do this in a way that's as hidden from the user as
   possible, like in a pod that doesn't have a manifest associated with it
   so that its status / logs don't show up in the UI.
4. If the code is in a state such that the process crashes on startup then we
   can't get its argv this way (AFAIK) and we have a new failure mode that's
   kinda awkward to explain to users.
   1. Well, "process crashes on startup" is really an existing failure mode,
      except now we're doing it in a case where the failure log isn't streamed,
      and we're not quite using the existing pod watch flow.

#### Require the user to specify the entrypoint
OK, so we need the entrypoint but it's hard to get. Why don't we just have the
user tell us what entrypoint to use?

1. If we want Tiltfiles to be runtime-independent, this means we need to require
   the entrypoint even for people using docker. This seems like it's making life
   worse for the (current) majority to help out a (currently) small minority,
   who already have a workaround available.
2. Specifying the entrypoint in the Tiltfile is ugly duplication and an
   opportunity for bugs and really kind of an embarrassing UI, since a user
   would have a reasonable expectation that Tilt should be able to figure this
   out.

#### Can we just, like, not?
1. Deprecate the `container_restart` directive
2. Implement entry point overriding at the image level (already done).
3. Document how to use `live_update` with non-hot-reloading runtimes. This makes
   the UI:
   1. Commit the restart wrapper script to your repo.
   2. Add the wrapper script to your docker image (or we need to add a
      feature to allow users to specify extra image files in their Tiltfile).
   3. Override the image's entrypoint in the Tiltfile to use the wrapper script.
   4. Add a `run` to your `live_update` that triggers on restart.txt.

This is a really nasty UI for a feature in which we've previously invested a lot
of time to make as easy to onboard as possible. It also causes a user to worry
about having different Dockerfiles for dev and prod.

### Recommendation

I think requiring the user to specify the entrypoint is the least bad option
here. entrypoint inference is a feature we could build on top of that, if people
complain.

I think we want to leave in the existing `container_restart` behavior for
docker, since that's still a majority of our customers (AFAIK), and it doesn't
currently seem to buy us much to disable that behavior.

## Other concerns
1. We're "magically" changing an image's entrypoint.
  1. This might make users scratch their heads. Naming the wrapper script
     something like tilt-live-update-container-restart-wrapper.sh, while not
     providing perfect transparency, would at least make the culprit very
     evident and be a pointer for further understanding.
  2. The user might want to disable the entrypoint-overriding behavior for
     debugging purposes. At the moment, their only recourse will be to remove
     the `container_restart` directive.
