# Manual Update Control Spec

## Background
Tilt updates your cluster each time you save, staying in sync without active tending. This supports our brand promise of “liveness” and speed. But users and potential customers are put off by certain negative effects:

- WIP builds are often broken → alert fatigue 
- Deploying a half-done version → problems. e.g., DB gets into an inconsistent state

## Goals
- You can set which Resources you want to be in Manual TriggerMode.
- In the Web UI, you can see which Resource are Auto and which are Manual.
- In the Web UI, You can trigger Manual Resources to update.


## Non-Goals

### Update All Resources
A button to update all Manual TriggerMode Resources at once. Might be convenient for use cases with lots of Resources set as Manual. We should only do it if we get feedback that users want to simultaneously update several manual resources at once.

### Abort ongoing updates
This would require more engine changes. We should only do it if we have evidence that users have builds that they want to abort.

### Configure TriggerMode via Web UI
Right now, you can only configure TriggerMode via the Tiltfile. Changing these settings via the Web UI could appeal to those who like GUIs rather than long config files. But it’s high cost for uncertain impact, since it’s the first instance of altering the Tiltfile without directly editing it.

![Manual Update Control Config](/manual_update_control_config.png)


### Onboarding
For now, the main way for users to discover this feature is through our Blog Post, Docs, or speaking with Tilters. If we want this feature to be more discoverable for all Tilt users, we can improve onboarding so first-time users see a message like, “Tilt automatically updates your Resources when you make changes in your filesystem. But you can set Resources to Manual to have more control.”

We should do this as part of a larger Onboarding epic, so we can properly prioritize the features we want to surface for first-time users.

![Manual Update Control Onboarding](/manual_update_control_onboarding.png)

## Future Goals

### Force Update Single Resource / All Resources
Sometimes Resources can get into a weird state, and users might want to re-trigger updates to try and fix the issue. 

We can also offer the option of whether the user wants to trigger a Full Update or a LiveUpdate.

### Pause/Unpause 
A button to temporarily make everything Manual, which won’t persist past a web session. Tilt still “remembers” which items are persistently Manual, as configured in Tiltfile.

When you Unpause, Tilt updates everything that’s “dirty”.

Use case: keep Tilt from freaking out when you do a Git checkout.

### Keyboard Control
Engineering culture tends to demand full keyboard control for tasks that are repetitive, and otherwise require a lot of sniping and clicking with a mouse.

### Show Errors for Failed Updates
Right now, if we get non-OK responses from triggering a Manual Update, we log it to console. Users will be able to understand that an error happened if we change the UI. 


## Detailed Design

In Tiltfile, you can change the Trigger Mode of individual Resources to Manual. (Default is Auto.)

A Resource with changes that can be built/updated/deployed is not in sync with your filesystem, and has an indicator to show it’s “dirty”.

Any Resource in Manual TriggerMode has a button to trigger updates. For now, you can only trigger updates when a Resource is “dirty”. Otherwise the button is still visible, but disabled.

![Manual Update Control GIF](/manual_update_control.gif)

