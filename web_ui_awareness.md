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


## Script
Here's Tilt.
Each microservice is a line in the UI.
I can select/expand a service to see its pod ID and that Tilt's port-forwarding it to localhost:9000. I can open the page in a new tab.

I'm going to change a service. I can see that Tilt saw the change, and is dowing a docker build, docker push, and kubectl apply. I can see where time went in that sequence. When it's done, I can go to the webapp and reload to see my change.

I can introduce a syntax error, and see Tilt build it. When it fails, I see the relevant error text, pinned on my screen until I fix it.

If I fix the build error, but make the service die on startup, like there's a missing file. I can see the pod is in CrashLoopBackOff without any intervention.

If I fix the startup error but introduce a request time error, I can see that Tilt goes green. Until I make the request, at which time I see that it restarted. The relevant logs (from just before the restart) stay surfaced. Even though the pod is back to green, Tilt makes sure I don't miss this relevant data.

If I introduce an error in the Tiltfile, I see that error. The other services stay running: Tilt's smart enough not to blink them out of existence because of a momentary typo.

I can update a service that has Live Update. When I hit save, I can see it doesn't create a new pod but is much faster.

## Detailed design

TODO

## Alternatives Considered

TODO
