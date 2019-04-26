# Slim Build Context (Proposal)

## Problem
Generally, Tilt does not give users great control over what kinds of changes
cause builds to happen. Tilt (reasonably) aims to ensure that the user is
never running incorrect or out-of-date code, and the current implementation
leads to the user often waiting for builds and deploys that they really don't
need.

e.g.:
1. In [tiltdemo](https://github.com/windmilleng/tiltdemo/blob/f4a6aa3d0c43357ce0bb39bbc4ff5f37cfd2ffb8/Tiltfile),
we have two services, which each have '.' as the build context. If you change
demoserver1, you get a live update of demoserver1 and a full image build of
demoserver2, and vice versa. In order to prevent this, we'd have to dramatically
restructure the repo.
2. In tilt.build, Nick tried the [intuitive](https://github.com/windmilleng/tilt.build/commit/bd251498ceffae56f0493eb772f4a3399517f406) option to switching to `live_update`,
and this led to `tilt-site` doing a full build when files in `docs` changed,
and he had to restructure things to get around that.
3. Images with very simple base images, e.g.:
```
docker_build('base', '.', dockerfile='base.dockerfile')
img = docker_build('app', '.')
img.add('.', '/app')
```
(with `app`'s Dockerfile containing a `FROM base`).
It is perfectly reasonable for a user to define their config like this, with any changes
to `app` only relevant to the `app` image. However, since the build context for
the base image is `.`, any changes to `app` will invalidate `base`, causing
a full image build for `base`, `app`, and any other images/resources that depend
on `base`. It's effectively impossible to configure live updates for `app`
without moving `base` into a subdirectory, which is not very "meet the customer
where they are".

## Non-goals
Reducing how much data is sent to the docker daemon. While this is often excessive,
it's a thing people are used to, because it's how docker works, and it's not clearly
a source of pain.

## Solution Options
Brief descriptions here, pros/cons in the next section.
1. Watch only an image's actual dependencies instead of its full build context.

   e.g., parse each Dockerfile, find ADDs and COPYs, and only watch the files
     that get added and copied.
2. Add a `docker_build` option `only_rebuild_on` - only files listed here will
   trigger builds, but the build still has access to the full build context, e.g.:
   ```
   docker_build('base', '.', dockerfile='base.dockerfile', only_rebuild_on=['package.json'])
   ```
3. Add `docker_build` options `ignore` and/or `only`, which filter the build
   context - both the files that are watched and the files that are included
   for the build, e.g.:
   ```
   docker_build('base', '.', dockerfile='base.dockerfile', ignore=['**', '!package.json'])
   ```
4. Allow `docker_build`'s context param to take either the string (directory)
   it currently does, or a new `BuildContext` type that specifies a directory
   and also watch/build rules, e.g.:
   ```
   docker_build('base', build_context(dir='.', exclude=['app')])
   ```

## Solution pros/cons

1. ##### Watch only an image's actual dependencies instead of the full build context directory.

   Pros:

   1. Everything just works magically, with no new syntax or configuration.
   2. Behavior is potentially more intuitive, and user doesn't need to expand
      their mental model of tilt.
   3. Byproduct - if we're learning an image's ADDs/COPYs from the config, that's
      a step towards making `live_update` easier by either inferring all syncs
      entirely, or at least inferring sync destinations (and potentially
      chowns).

   Cons:

   1. Magic is great when it works, and extra-frustrating when it doesn't.
      We'd want features to allow users to diagnose when it's not working and
      potentially escape hatches to allow them to work around it not working.
      If these escape hatches take the form of other solutions in this spec
      anyway, maybe we should just do those first and then we can implement
      this solution later if needed.
   2. It's unclear how hard this will be to get right. It feels fairly
      corner-casey, and like we'll potentially end up discovering and
      reimplementing various docker logic and features. I only spent an hour
      digging into this feeling of corner-casiness, and it's possible there's
      an option that lets us do this robustly. Our [current implementation](https://github.com/windmilleng/tilt/blob/e65742e96b9a7b0684a2e75547cc8089f3ac76ba/internal/dockerfile/dockerfile.go#L147)
      calls out a few cases. There's also [ARG](https://docs.docker.com/engine/reference/builder/#arg) and
      [environment replacement](https://docs.docker.com/engine/reference/builder/#environment-replacement)
      support. We could just implement the 90% cases and worry about corner cases
      as they're reported, but then they're corner cases of the corner case of
      this whole spec, which makes them hard to prioritize and leaves the feature
      feeling buggy regardless.

2. ##### Add a `docker_build` option `only_rebuild_on` - only files listed here will trigger builds, but the build still has access to the full build context.

   Pros:

   1. Simple implementation.

   Cons:

   1. The user has to explicitly list dependencies, which feels like boilerplate/
      bookkeeping, repeating whatever's in the Dockerfile. Also, as is often the
      case with such boilerplate, introduces a source of annoyance/user error,
      in that this might not match the Dockerfile, either because the user fails
      to keep it in sync with the Dockerfile, or simply typos.
   2. A simple dependency list isn't very flexible, e.g., what if you want to
      include go but exclude unit tests, or include js but exclude ts.
   3. Introduces a distinction between "files we include in the build" and
      "files we watch to trigger a build", making the user's required
      mental model a bit more complicated.

3. ##### Add `docker_build` options `ignore` and/or `only`, which filter the build context.

   Pros:

   1. Simple implementation.
   2. Similar to #2, but more flexible - e.g., can include `internal` and
      exclude `internal/**/*_test.go`.

   Cons:

   1. Risks feeling boilerplatey, as with option #2 above.
   2. The flexibility over #2 is nice, but also kind of weird that we'd end up
      having to do, e.g., `['**', '!package.json']` sometimes (and we don't have
      good evidence people are very aware of this concept from Dockerfiles,
      but maybe we can mostly mitigate that by advertising in our docs?). An
      simpler, less flexible alternative would be `includes=[...], excludes=[...]`.
      That might be flexible enough.

4. ##### Allow `docker_build`'s context param to take either the string (directory) it currently does, or a new `BuildContext` type that specifies a directory and also watch/build rules.

   Pros:

   1. Simple implementation.
   2. As a byproduct, might be a good replacement for `deps` in `custom_build`
      (i.e., could provide better control over which files get fed to the command)
      I'm not sure if this actually solves a problem, but it at least keeps
      syntax more consistent between `docker_build` and `custom_build`.

   Cons:

   1. Feels mostly like syntactic overhead on top of #3
   2. Adds extensibility, but there seems to be little reason to add this extra
      syntax when we're not sure it's the direction we want to go in or that
      we'll need that extensibility, and it's easy to add later, as needed with
      little wasted work.

## Proposal:
Option #3 - Add an `ignore` option to `docker_build`, which takes either a
string or array of strings, and is treated as a .dockerignore file.

(`ignore` over `only` because
a. it's feels a bit more self-documenting and b. it can be understood as simply
a per-image `.dockerignore`.)

This is a cheap, straightforward solution that we can get done quickly
and doesn't preclude other options in the future.

If it's cheap, also add it to `custom_build`.

## Questions
1. This is introducing a new class of user error - bad file patterns. Is there
some form of assistance we can provide for this? e.g., producing warnings
for patterns that match all files or 0 files.

  1. This might be a Tiltfile principle that needs discussion, but I think all
  warnings should be avoidable. We don't want to train users to ignore warnings,
  and we want to retain the ability to make warnings at least a little annoying.
  If a user is reasonably doing something supported, we shouldn't give them a
  warning. I can think of two cases in which it's reasonable to have a pattern
  that matches no files: 1) if you're sharing a list of patterns across
  multiple resources, and not all patterns apply to all resources, and 2) if
  your list of patterns contains temp files that might not always exist.
  I'm not aware of a user interface that provides users with a better feedback
  loop for this. gitignore and dockerignore both also pretty much just give you
  the "it works or it doesn't" result (though in gitignore's case, it's
  easier to inspect for individual files; for dockerignore, the file just gets
  left out). This is relevant both because it means users will be used to it,
  and because it means we don't have a good solution at hand.

2. Prior to this feature, a user could replicate Tilt's docker build by simply
running docker build outside of Tilt, on the command line. With this change,
there is no longer a command they can run to match the build that Tilt is
performing. How do we expect users to debug the image?

  1. I suspect this doesn't affect most image debugging. Performing the docker
  build outside of Tilt is one of many possible debugging steps, with the aim
  of eliminating Tilt misconfiguration and Tilt bugs as possible sources of
  error. Additionally, we expect this feature to mostly be used to reduce
  the number of builds that happen, but not really affect the functionality.
  If the user wants to debug an image for any reason other than ignore patterns,
  doing a command-line docker build should be fine; it'll just produce an image
  with some more files than were there from the Tilt build. I think for someone
  to be bitten by this, they would need to make it through all four of: 1) using
  this feature, 2) running into problems caused by misconfigured ignore patterns,
  3) performing a docker build outside of Tilt to diagnose those problems, 4)
  observing it working in the non-Tilt build but not the Tilt build, 5) failing
  to blame Tilt after observing #4. I don't see a good way to mitigate this,
  and it seems like an acceptable risk.
