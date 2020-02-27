# Onboarding Guide Outline

## Overview

Each major language that Tilt supports should have an onboarding guide that demonstrates:
- The simplest way to set up Tilt with that language
- The recommended way to set up Tilt with that language ("the fastest deployment time")
- Walkthrough a clear path between those two setups

This guide should have working code samples that the user can check out and explore themselves.

We're basically cannibalizing the narrative of "The Quest for the Fastest Deployment Time"

Slides: https://docs.google.com/presentation/d/1Vs4a4SSI4BE2094bloBY410FJkaH1WucHvVXcOJ78H8/edit#slide=id.g76703e551a_2_427
Code: https://github.com/windmilleng/fast

But with a couple changes in scope:

- More detailed explanation of what the Tiltfile does
- Fewer intermediate experiments
- A simpler app

## Code Structure

All repos should borrow the code structure from https://github.com/windmilleng/fast

The repo name should be `windmilleng/tilt-example-LANGUAGE`. So Go would be
`tilt-example-go`, Java would be `tilt-example-java`, etc.

Each subdirectory would be a version of the app:

- 0-base: The simplest possible app
- 1-measured: The simplest possible app, but modified to measure its own deployment time.
- 2-name-a: First optimization
- 3-name-b: And so on
- 4-recommended: The recommended version of the app.

## Narrative Structure

The best indicator of a healthy development workflow is a short feedback loop.

Kubernetes is a huge wrench in the works.

Let’s fix this.

If you just want the final version that is ready to use, click here to skip the exploration.

### The Initial Setup

A simple Hello World server.

The simplest possible setup for developing on Kuberentes with Tilt

Link to deployment yaml and Dockerfile

Introduce Tiltfile

Line 1 builds a docker image

Line 2 loads the Kubernetes yaml. The image name must match the image name in the yaml.

Line 3 sets up port-forwarding so we can see the server.

Link to snapshot of the server running

### Let's Add Benchmark Trickery

Before we try to make this faster, let's measure it.

Add a local_resource that records the current time, and kicks off a build.

Add code to the app that records start up time.

Let's see it!

Current score of our naive hello world app.

### Attempt #1

Leave Tilt running.

Optimize the Dockerfile.

Current Score

### Attempt #2

Add live_update

Introduce sync()

Maybe introduce run()

Current Score

### Attempt #3

Add local_resource + live_update binary

Current score

### Our Recommendation

Final Score

Link to recommended repo directory

Caveats / gotchas

Other optimization tricks / other example projects (both internal and external links)



