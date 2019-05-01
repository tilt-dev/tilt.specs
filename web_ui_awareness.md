# Web UI Awareness


&nbsp;
## Overview

The Web UI gives you at-a-glance awareness of what Tilt is doing, so you can see whether your app is updating properly, and if you need to investigate an issue. 

When this sprint is complete, we should be able to demo Tilt's main value propositions with the Web UI. (The current [demo video](https://tilt.dev) shows only the Terminal UI).


&nbsp;
## Background

At the start of this sprint, `tilt up` starts the TUI, and you launch the Web UI by hitting keys shown in the key legend footer.

This is because we first built the Web UI to browse logs better than you can with the TUI. After receiving positive feedback from customers, we are addressing this next question: how might we make the Web UI a standalone experience? 

As we plan and build this standalone Web UI, we keep recent lessons in mind:

- We can learn from shortcomings of the TUI. For example, the TUI's overview of resources is a list, with each item independently expandable. When there are many errors, the respective resources automatically expand, and push other resources offscreen. Seeing error detail should not make the overview less useful.

- We can also learn from past challenges in getting teams to adopt Tilt. We've been successful engaging implementers who get Tilt to work, but less so in selling their teammates. Can we help "show, not tell" the value of Tilt with our Web UI?



&nbsp;
## Requirements

Changes to the Sidebar, Statusbar, and a new Error pane make it more visible when Tilt responds to file system updates, builds/updates and deploys your resources, and encounters exceptions. 

Since this is the first pass at a standalone Web UI, we limit our focus to two common use cases: 

1. Normal iterative flow. You can see Tilt is working properly.

2. Errors. You see when and where the error occurred, and see log context.


&nbsp;
## Non-Requirements

**SURFACE MORE INFO:**

Tilt has lots of potentially relevant and useful data that we don't show (or surface) in the Web UI. We need more feedback on this first pass, but we should eventually think through whether or how to add other data to our UI:

- Build history for individual resources
- The YAML Tilt read or deployed
- The history of changes to a Pod (e.g., `kubectl get pods -w`)

&nbsp;

**"SMARTER" ERRORS:**

We can also better explain specific types of errors, rather than let the user infer:

- All images depend on one base image, and there’s an error there.
- Your cluster is down. Or your Internet is down.


&nbsp;
## Detailed Design

In the default interface, the **overview** panes allow you to navigate to **detail** panes, which either show more information about a single resource (Ex: `Sidebar > doggos > Log`), or show a custom filtered view of your data (Ex: `Sidebar > ALL > Errors`).

***
The changes support these use cases, which assume you already have a Tiltfile set up:

1. You change code, then see a successful rebuild and deploy
    - You see Tilt detect the change and start working
    - You see Tilt is done

2. You introduce errors, see what happened, and how to resolve them.
    - Build error
    - Startup error
    - Runtime error [1]

[1] The UI shows the error once you encounter it. Ex: You load the page and see a 500 error. 

![](/web_ui_awareness_mock.png)

&nbsp;
### STATUSBAR
1. Show a single-line message to give a sense of what Tilt is doing. Messages should be informative, but legible (i.e., if some status would only display for 1ms, show instead the parent process that displays for ~1s)

	Example messages:

	```
	Reading Tiltfile
	Updating: fe
	Updating: fe — Building Dockerfile
	Updating: fe — Pushing Docker image 
	Updating: fe — Deploying
	Updating: vigoda 
	...
	```


2. Show "Last Edit" section with last detected file change.

	```
	Last Edit: main.go (fe)
	```
	
	Multiple:
	
	```
	Last Edit: main.go (fe) [+2 more]
	```



&nbsp;
### SIDEBAR
Make each resource item have a more visible status.

- Last deployed timestamp
- When building, animations to show activity


&nbsp;
### ERROR PANE
In `Sidebar > ALL > Errors`, show a single page with all errors.

- Even if there are no errors, you can keep this pane open so you'll see errors as soon as they appear.
- When there are no errors, you see a design that looks intentionally empty.


&nbsp;
### ADDITIONAL INFO
Show info about resources. We consider pod ID and endpoints to be some of the most useful information, which helps you investigate issues using your own process. (Even `kubectl`)

- Log Pane shows pod ID and and endpoints


&nbsp;
## Alternatives Considered
1. We had internal debates about whether to deprecate the TUI, and make the Web UI work more like a default interface. No decision yet.

2. We talked extensively about whether errors automatically show up. DB advocated for Tilt to show you error log context, so you can fix the issue without having to navigate anywhere in the Web UI. But if we show error pop-ups, this might take your focus away from something you care about more. For now, we are addressing this use case by creating an Error Pane where errors will automatically appear.

&nbsp;
## Appendix: Demo Script from DB

Here's Tilt.
Each microservice is a line in the UI.
I can select/expand a service to see its pod ID and that Tilt's port-forwarding it to localhost:9000. I can open the page in a new tab.

I'm going to change a service. I can see that Tilt saw the change, and is dowing a docker build, docker push, and kubectl apply. I can see where time went in that sequence. When it's done, I can go to the webapp and reload to see my change.

I can introduce a syntax error, and see Tilt build it. When it fails, I see the relevant error text, pinned on my screen until I fix it. (See #2 in Alternative Considered)

If I fix the build error, but make the service die on startup, like there's a missing file. I can see the pod is in CrashLoopBackOff without any intervention.

If I fix the startup error but introduce a request time error, I can see that Tilt goes green. Until I make the request, at which time I see that it restarted. The relevant logs (from just before the restart) stay surfaced. Even though the pod is back to green, Tilt makes sure I don't miss this relevant data.

If I introduce an error in the Tiltfile, I see that error. The other services stay running: Tilt's smart enough not to blink them out of existence because of a momentary typo.

I can update a service that has Live Update. When I hit save, I can see it doesn't create a new pod but is much faster.