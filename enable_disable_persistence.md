# Enable/Disable State Persistence Spec

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
system. We have mocks and a pretty good idea of how we want the narrowly-focused
single-click enable/disable interaction to work.

For more details, see [Enable/Disable Resource API Spec](./enable_disable_resources.md).

But we have different ideas on how we want the enable/disable interaction to work
across multiple Tilt runs, and how the state should persist! [^1]

## Primary Goal

The goal of this doc is to give a product specification by example: describe
specific sequences of user actions, and how we want Tilt to behave. (Note that
it is not intended as a description of how Tilt behaves today.)

## Non-Goals

### Groups

We expect that we will eventually want to control groups of services, but
we don't believe that changes our analysis here about how single services behave.

## Terminology

### Enabled

We say a server is "enabled" if it's visible in the Tilt UI and is in status
waiting on trigger, pending on dependency, running, completed, or error.

### Disabled

We say a server is "disabled" if it's described by the Tilt API and its status
"disabled". We do not expect any new logs or status changes until the
service is intentionally "enabled"

### Session

We say a Tilt "session" is the time from when the user runs "tilt up" to when
the tilt process exits.

### Service Names

For readability, we'll use NY pizza places as service names (Paulie, Juliana, Roberta, Joe).

## Prior Art

### Resource Pinning

Today, when you pin a resource in the Tilt UI, the pin persists across Tilt sessions.

The state is stored the browser's LocalStore, and is tied to the user's browser.

### Manual Trigger

Today, when you switch a resource between manual trigger mode and auto trigger
mode, that state persists until you reload the Tiltfile. When the Tiltfile
loads/reloads, the trigger mode reverts to what's specified in the Tiltfile.

### systemd

[systemd and
systemctl](https://www.commandlinux.com/man-page/man1/systemctl.1.html) is
probably the most widely known service manager. You tend to see its verbs in
other service management tools (like
[Edward](http://engblog.yext.com/edward/commands/#start)).

`systemd` is a persistent daemon that runs admin-level services on linux.

You use `systemctl start` and `systemctl stop` to start/stop services.

And `systemctl enable` and `systemctl disable` to enable/disable services to
auto-boot at system startup.

Note that `enable` does not start a service immediately. In other words, systemd
models "start on boot" and "start now" as independent actions.

## Product Specification

### Run all, Disable from UI

1. Write a Tiltfile with services Paulie and Roberta
1. Run `tilt up` -> Paulie and Roberta are enabled.
1. Click "disable" on "Paulie" -> Roberta is enabled, Paulie is disabled
1. Ctrl-c and re-run `tilt up` -> Roberta is enabled, Paulie is disabled

Possible Principle:

`tilt up` (without arguments) always inherits all the enable/disable overides
from the previous Tilt session.

### Specify services, Disable from UI

1. Write a Tiltfile with services Paulie and Roberta and Joe
1. Run `tilt up paulie roberta` -> Paulie and Roberta are enabled, Joe is disabled.
1. Click "disable" on "Paulie" -> Roberta is enabled, Joe and Paulie are disabled.
1. Ctrl-c and re-run `tilt up paulie roberta` -> Paulie and Roberta are enabled, Joe is disabled.

Possible Principle:

Tilt always respects arguments to `tilt up` when it starts.

### Specify services, Enable from UI

1. Write a Tiltfile with services Paulie and Roberta and Joe
1. Run `tilt up paulie roberta` -> Paulie and Roberta are enabled, Joe is disabled.
1. Click "enable" on "Joe" -> Paulie and Roberta and Joe are enabled.
1. Ctrl-c and re-run `tilt up paulie roberta` -> Paulie and Roberta are enabled, Joe is disabled. 

Possible Principle:

Tilt always respects arguments to `tilt up` when it starts.

Your UI change has been wiped out by the `tilt up` args.

### Run all, Disable from UI, Change Args

1. Write a Tiltfile with services Paulie and Roberta
1. Run `tilt up` -> Paulie and Roberta are enabled.
1. Click "disable" on "Paulie" -> Roberta is enabled, Paulie is disabled.
1. Run `tilt args paulie roberta` in a separate terminal. -> Tiltfile re-executes, Paulie and Roberta are enabled. 

Possible Principle:

Changing the tilt args wipes out any enabled/disabled services from the UI.

### Run one service, Enable from UI, Change Args

1. Write a Tiltfile with services Paulie and Roberta
1. Run `tilt up paulie` -> Paulie is enabled, Roberta is disabled.
1. Click "enable" on "Roberta" -> Paulie and Roberta is enabled.
1. Run `tilt args paulie` in a separate terminal. -> Tiltfile re-executes, Paulie and Roberta stay enabled.
1. Run `tilt args roberta` in a separate terminal. -> Tiltfile re-executes, Roberta is enabled, Paulie is disabled.

Possible Principle:

This is probably the most controversial - that calling 'tilt args project1` twice should
persist the enabled/disabled state (because the args havne't changed).

### set_enabled_resources() override

1. Write a Tiltfile with services Paulie, Roberta, and Joe. The Tiltfile calls `set_enabled_resources(['paulie', 'roberta'])` on ALL inputs.
1. Run `tilt up paulie` -> Paulie and Roberta are enabled, Joe is disabled.
1. Click "disable" on "Paulie" -> Roberta is enabled. Paulie and Joe are disabled.
1. Run ctrl-c, `tilt up paulie` -> Tilt restores previous state: Roberta is enabled. Paulie and Joe are disabled.
1. Run ctrl-c, `tilt up joe` -> Tiltfile re-executes: Paulie and Roberta are enabled, Joe is disabled.

Possible Principle:

This is deliberatly an edge-case tickling case, but demonstrates the ways
that `set_enabled_resources` can ignore what's passed on `tilt up`.

Changing the `tilt up` args wipes out previous state.

## Possible Implementation

The Tiltfile controller persists the TiltfileSpec and any ConfigMaps that
Tiltfile owns under ~/.cache.

(Note that `tilt args` are modelled as `TiltfileSpec.Args`, a field of the Tiltfile spec.)

If Tilt starts with no args specified, the Tiltfile controller should reload the
ConfigMaps used to enable/disable services from ~/.cache.

If the Tiltfile re-executes, it should update the ConfigMaps that store
enable/disable state if and only if TiltfileSpec.Args has changed.

We could use a similar approach for trigger mode overrides, where the state is stored
in ConfigMaps that persist across Tiltfile executions.

---

[^1]: Many few years ago, when I was at Medium, I wrote a product spec about
    "what happens when the cursor is in position X and you hit `Enter` multiple
    times". It's not surprising that people had different opinions on what
    should happen. But what DID surprise me is that people often had internally
    inconsistent opinions: i.e., a user would give different answers for
    equivalent scenarios with different backstories. I expect this problem will
    come up a lot in this doc.
