# Web UI Awareness

## Overview

The Tilt Web UI should give you more awareness into what Tilt is doing Right Now
and what you need to pay attention to.

The goal of this spec is to flesh out the awareness UI enough that we can
make a useful demo video, similar to the existing video on https://tilt.dev/

## Background

TODO(anyone): History of existing TUI awareness UI, user feedback, problems.

## Requirements

- You can see that Tilt is reacting to file system events.
- When you save a file, Tilt should communicate that it saw the save.
- You san see that Tilt is reacting to cluster events.
- You should be able to find the pod ID in the cluster.
- Tilt should communicate when a service has an endpoint URL that you can open.
- You should be able to see what Tilt is doing Right Now, with some explanation/justification for why it's doing that.
- Errors should pop into your view. You shouldn't have to remember to look in an error bucket.

## Non-Requirements

These are things that are non-essential right now, but which we may want to tackle in the future.

- The build history of an image
- The duration of an image build
- The YAML Tilt read
- The YAML Tilt deployed
- The history of changes to a Pod (e.g., `kubectl get pods -w`)

## Detailed design

TODO

## Alternatives Considered

TODO

