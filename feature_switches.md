# Feature Switches

## Problem
When we implement a particularly risky new feature, we don't necessarily want to
turn it on for everyone before we've dogfooded / QA'd it.

Sometimes, we might want some of our more adventurous users to use it for a bit
before we turn it on for everyone.

For a few features now, we've had simple arg/envvar-controlled feature switches,
which has worked well enough, but there are a few reasons we might want to
invest in a more standard way of doing this:

1. Making it easier to enable for groups of people. For previous features,
we've missed out on a lot of dogfooding because, for the most part, only the
person/people working on the feature opted into the feature. We could
dramatically increase the amount of dogfooding by enabling the feature
developer to turn on the feature for all tilt maintainers. Similarly, we could
turn it on for a customer team and get feedback from them without having to
ask them to, e.g., all add an environment variable to their .bashrc, or
change the args they use to invoke Tilt.

## Solution
Add a new, non-publicly-documented Tiltfile builtin that specifies a feature flag value.

Make it an error if the builtin is called after any other Tilt builtins,
to ensure that other Tilt builtins can access feature flag values.

syntax: `feature_flags({'k8s_events':true})`

Pros:
1. Easy to implement
2. Easy for users to understand
3. If we (or a customer) enable a feature flag in a Tiltfile, it'll be on for
  everyone running tilt on that Tiltfile, which makes it much easier to get
  usage of new features.

Cons:
1. This means feature flags can only affect the behavior of features that come
  after the builtin is called, which means we can't use it for things like
  "do we start the HUD" or "do we start the web server"
2. This also sets the expectation that changing the feature flag value in the
  Tiltfile will take effect immediately, while our existing, ad-hoc feature flag
  implementations have assumed that feature flags are immutable. This has might
  be worth a set of pros and cons on its own (basically expanding on Con #1 vs
  being able to control features without restarting Tilt, but this is a
  potentially controversial constraint / feature).

## Other options
1. Make a new feature flag file that we read very early on Tilt startup, and
  don't filewatch/reread during execution.

  Pros:
  1. simple
  2. allows the use of feature flags on startup features
  3. saves us from having to make our flag-controlled features responsive
  Cons:
  1. kinda weird to put stuff in a separate file
  2. it's nice to be able to control features at runtime
2. standardize on one of the previously-used cmd-line args / env var approaches
  Pros:
  1. same as above
  Cons:
  1. doesn't allow one to centrally control feature flags for a team
3. something fancier
  Pros:
  1. better!
  Cons:
  1. we're not really sure what we want right now
  2. more expensive, and we don't really have enough features to justify
     more than dirt cheap right now.

## Potential Future Work
1. Remote control
  1. e.g., Tilt fetches a feature switch config on startup, enabling Tilt
     maintainers to:
    1. remotely enable or disable a feature for a team (e.g., if we know we
       don't want to risk a new feature for a team with an unusual cluster
       configuration)
       1. requires knowing who people are, but this seems likely to be a thing
          we'd mostly want to use with people comfortable identifying themselves
          anyway
    2. gradual rollout (e.g., ship to 10% of users and see if we get bug reports)
