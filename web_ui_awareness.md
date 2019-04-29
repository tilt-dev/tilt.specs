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
Tilt has lots of potentially relevant and useful data that we don't show (or surface) in the Web UI. We need more feedback on this first pass, but we should eventually think through whether or how to add other data to our UI:

- Build history for individual resources
- The YAML Tilt read or deployed
- The history of changes to a Pod (e.g., `kubectl get pods -w`)

We can also better explain specific types of errors, rather than let the user infer:

- All images depend on one base image, and there’s an error there.
- Your cluster is down. Or your Internet is down.


&nbsp;
## Detailed Design

In the default interface, the **overview** panes allow you to navigate to **detail** panes, which either show more information about a single resource (Ex: `Sidebar > doggos > Log`), or show a custom filtered view of your data (Ex: `Sidebar > ALL > Errors`).

***
The changes support these use cases, which assume you already have a Tiltfile set up:

1. You change code, then see a successful rebuild and deploy
    - You see Tilt detect the change and start working (and what triggered it)
    - You see Tilt is done (and know your service is ready to check)

2. You introduce errors, see what happened, and how to resolve them.
    - Build error
    - Runtime error (after you trigger the error)

![](/web_ui_awareness_mock.png)

&nbsp;
### STATUSBAR
1. A single-line message gives you a sense of what Tilt is doing. Messages should be informative, but legible (i.e., if some status would only display for 1ms, show instead the parent process that displays for ~1s)

	Example messages:

	```
	Reading Tiltfile
	Building: fe
	Building: fe — Building Dockerfile
	Building: fe — Pushing Docker image 
	Building: fe — Deploying
	Building: vigoda 
	...
	```


2. "Last Edit" updates with last detected file change.

	```
	Last Edit: main.go (fe)
	```
	
	Multiple:
	
	```
	Last Edit: main.go (fe) [+2 more]
	```



&nbsp;
### SIDEBAR
- Each resource item has additional information to show its status.
	- Last deployed timestamp
	- When building, animations to show activity


&nbsp;
### ERROR PANE
`Sidebar > ALL > Errors`

- Even if there are no errors, you can keep this pane open so you'll see errors as soon as they appear.
- When there are no errors, you see a design that looks intentionally empty.


&nbsp;
## Alternatives Considered

We are having internal debates about whether to deprecate the TUI, and make the Web UI work more like a default interface. For now, we are deferring this question. 
