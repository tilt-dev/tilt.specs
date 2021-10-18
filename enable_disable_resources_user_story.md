# Enable/Disable Resource User Stories

## Background

Tilt currently has a system for changing the arguments to `tilt up` at
runtime. We usually call it `tilt args`.

https://docs.tilt.dev/tiltfile_config.html

Most teams that use this system are using it for enabling and disabling sets of
services, as demonstrated in this example:

https://docs.tilt.dev/tiltfile_config.html#run-a-defined-set-of-services

This helps save CPU when you only need to run a subset of your services.

A number of teams have built wrappers around `tilt args`. This has helped us to
learn a lot about the ergonomics that people want, and the pains of the current
system. We have mocks and a pretty good idea of how we want this feature to behave.

## Goals

The goal of this document is to write down the user stories & use-cases we're
focusing on right now, and which user stories are secondary.

We expect these user stories to guide:

- How we ask for feedback from users

- How we resolve design ambiguities

- How we announce and describe the feature in documentation

## Non Goals

We think it's useful to have 100% agreement on the basic user stories we're solving for.

We don't think it's useful to require 6 people to make a decision on the nuances
of every design mock (e.g., where every button should go), or how to resolve
every technical edge case (e.g., what happens when you remove a database that
another service depends on). The [Safe to
Try](https://github.com/tilt-dev/company/tree/master/communications#to-make-hard-things-safe-to-try)
principle may help here.

This doc may contain examples of how the user stories may play out in different 
cases. But the doc isn't intended to be an exhaustive list of decisions.

## User Story

These are use cases are part of our core messaging about the feature:

As an app developer,

- I have a list of servers I can run, which may include 1,000+ servers.

- I need a way to browse the list of servers available to me.

- I frequently run a subset of those servers in my environment (which may be 1-25).

- Most people on my team run the same subset of servers to iterate on a given
  feature. 
  
- My team may work on multiple features that use different subsets.

- I need to run new servers to either make a change or verify a change I made to
  the currently active servers. So I need to be able to start the new server,
  and stop it when I'm done.

### User Story Corrolaries

These principles are necessarily implied by the above:

- I don't remember the complete list of servers that I'm running now. (There are too many.)

- I don't remember the list of servers that are available to me. (There are way too many.)

- I only need to focus on the servers that are active in this session.

- The ratio of available servers to running servers is large.

- I should be able to enable/disable servers from either the UI or the CLI.

### User Non-Stories

These are use-cases that we may support. But they are not part of our core messaging
or storytelling about the feature. They are not use-cases we're explicitly designing for.

- A server is misbehaving, so I want to pause/stop it for a few minutes.

- I want to trim down the servers I need for this particular session to save CPU.

## User-Facing Terminology

We use the terminology in user-facing docs:

- "Enabled Resources" - all resources in my environment

- "Resource Catalog" - all available resources (this is quickly becoming a term of art)

- "Enable/Disable" - run a resource in my environment / stop running a resource in my environment

Notably, we stay away from: "add", "remove", or "service". We think these words are too likely
to be confused with other things.

## Examples

Here are a few examples of how these user stories play out in the existing designs.

### Auto/manual Trigger

Currently, Tilt has a mode denoted `trigger_mode=manual, auto_init=False`. This indicates
a server that doesn't start until the user has explicitly specified it.

We expect this mode will eventually go away, because it fulfills the same user
need as a disabled resource.

But we don't think we should invest time right now in removing it. We expect
the mode to exist, but be heavily de-emphasized in all UIs and documentation.

### Disabled Resource Display

Disabled resources should be much less prominent in the UI than enabled resources.

They should take up less visual space in the default view. It's OK if they're harder to navigate to.

We should not invest a lot of time right now in how to display logs or display
historical status of disabled resources. It may NEVER make sense to show this
data.
