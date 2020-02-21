# Toolchains Behind Successful Kubernetes Development Workflows

## Abstract

Kubernetes solved a lot of problems, but it created a clumsy development workflow: Every code change requires fiddling with containers, registries, and manifests. Managing config files isn't trivial. Distributed debugging; a mystery. Dev clusters are tricky to set up, and sharing cluster state among team-members is mostly fiction.

L Körbes, an expert in Kubernetes development tooling, outlines successful development workflows in three different settings: a very large enterprise, a small and agile startup, and a popular open source project.

L will share how they set up dev clusters, manage configs, automate the development feedback loop, share context across teams, debug, and, finally, deploy to production.

Come learn how these teams made their Kubernetes dev workflows not only seamless, but amazing to use!

Link on KubeCon’s website: https://sched.co/Zet3

## Research

The questions we want to answer:
- What Kubernetes tooling do they use besides Tilt?
- What kind of development clusters do they use?
- What hack/solution do they use to deal with their sea of config files?
- How do they automate their feedback loop? (That’s Tilt. Anything else?)
- How do they share context across teams? Or: Do they have additions or alternatives to Snapshots?
- Debugging tips & tricks they came up with
- What’s the workflow that takes their code from Tilt to production

## Mindspace notes:

- First devex tool was Helm!
- Tooling besides Tilt: Helm, Circle.
- Used Skaffold before Tilt.
- Config files:
    - Helm solves a lot of the config files.
    - There are Helm charts that Tilt manages.
    - A lot of templating with Helm.
    - Each dev has one (local) helm-values file that feeds a lot of templates. It’s pretty large.
- Dev clusters:
    - Running Linux and Docker for Mac (Why Docker for Mac? Because.)
    - Use docker compose for unit/integration tests because of issues with kubernetes.
    - Local clusters are the same, in terms of what gets deployed, as they have in prod (except e.g. Mongo, replication).
- Feedback loop:
    - Tilt.
    - They do all devs and ops together, wanted strong parity between dev and prod.
    - Before tilt remote dev was a no-go. Even now it’s a slower feedback loop.
- Don't share cluster context because it’s a very tiny company.
- Tricks:
    - Used to have a lot of Tiltfile hacks, but as Tilt's feature-set expanded their Tiltfile got tiny.
    - They have dictionaries and mapping and stuff to apply same logic across various standardized services.
- Remote debugging: IDE connects to node remote debugging; Tilt exposes the ports.
- To production: Before tagging, CircleCI goes through unit tests and integration tests (with Docker Compose—so prod and testing are separate environments [!]). Once they all pass, Docker Compose will build those images, tag them, and push them to Docker Hub. Then Terraform automation sees tag change and upgrades all services to the appropriate tags.
- Of all places where there’s struggle, CI is the main one. Getting k8s running on CI platforms and having CRDs to run tests, that’d be super cool.

## ClusterAPI notes:

- This whole section is still pending. Ignore for now.
- Kind as dev cluster. Something about Kustomize?
- Cluster API can be used with AWS or any other providers.
- "We have to do some stuff with the Dockerfile. We used to build the go binary in the container, so the development image needed the full go toolchain, but now we build it locally. So now we don’t have to do that."
- "Added a convention for all of our provider projects. They should take a tilt-settings.json file and merge the settings with the existing settings. Take the standard thing and overlay on top of it this other stuff."
- They use Snapshots a lot!
- The debugging strategy is Printlns. They want to use debuggers but haven't been able to justify the effort.
- TODOs
    - https://docs.google.com/document/d/1ifPynY4ntYremuoLc0qrOOOuTGGU_VLNoZ4j-MBP_cQ/edit
    - https://docs.google.com/document/d/1IpPp7SG45A52Uf7BVgYCKj-qosvceV5qrFiOg7EFxcE/edit?folder=0ABz6fv6poxWeUk9PVA
    - https://www.youtube.com/watch?v=sCD50fO95hI
    - https://cluster-api.sigs.k8s.io/user/quick-start.html
    - https://kubernetes.slack.com/archives/C8TSNPY4T/p1579124954156500
    - https://docs.google.com/document/d/1ifPynY4ntYremuoLc0qrOOOuTGGU_VLNoZ4j-MBP_cQ/edit#heading=h.ia8d0oni26ft
- To get in touch:
    1. Join the sig-cluster-lifecycle group https://groups.google.com/forum/#!forum/kubernetes-sig-cluster-lifecycle
    2. You’ll eventually get auto-invited to a meeting called “Cluster API Meeting”
    3. It will have a zoom link and a google doc with an agenda in the description
    4. Edit the google doc and add the topic you want to talk about to it at the date you want to talk about
    5. Show up that meeting!

# Lyft notes:

- Tools besides Tilt: 1. Self made ksonnet; 2) Kibana for logs; 3) kubectl wrappers.
- Dev cluster:
    - A single very large cluster.
    - Custom made, with a custom *ctl tool for it—simpler to use than kubectl.
    - Devs work in "contexts" (a namespace with customizations): two per user, expire a week from last activity, public and shareable. `devctl context use <context-name>`
    - TODO: see their devctl doc - https://docs.google.com/document/d/1eXIRWgCFr_3Xn2FRd1rb0DBuysgHHfEmqLKI_Rm4pK8/edit#
    - Runs on AWS with Terraform.
    - For cultural/historical reasons, they always run the whole thing—thus it doesn't fit in a laptop/local cluster.
- Config files:
    - They use Datum, a YAML to YAML generator. All services have a manifest that specifies how to build, run, etc., which Datum ingests and then spits out YAML. (This is prod-only for now.)
- Feedback loop: 
    - Plain Tilt, nothing else.
    - Devs don't build images. Devkube copies a base-image from prod to dev, reads the metadata, adds layers (what the dev is doing), and pushes only the layers. Skopeo copies that image between registries, and Tilt then fetches the new one to use for dev, without devs ever building an image themselves.
- Sharing cluster state:
    - Any dev can log in to another dev's context.
- Debugging, tips&tricks:
    - Most devs don't interact directly with Kubernetes.
    - Their wrapper for kubectl makes things easier e.g. setting a pod to debug mode adds to it netstat, top, etc.
    - The automated golden path command sets up 150 services.
    - They have tooling to simulate passenger and driver behavior, that runs locally.
- To production:
    - Pretty standard: PR, Jenkins, staging, tests, master.
- Challenge:
    - Their old environment uses immense images that are challenging to deal with.
- Note: Mention their KubeCon talk, which talks about Tilt.

## Unu notes:

- L lost their notes.
- TODO: re-build notes from Slack convos
- TODO: schedule another meeting
- They do the same as ClusterAPI re: "Added a convention for all of our provider projects. They should take a tilt-settings.json file and merge the settings with the existing settings. Take the standard thing and overlay on top of it this other stuff."
- They *want* to do the same as Lyft re: below but haven't managed to, and want us to make it possible for them:
    - Devs don't build images. Devkube copies a base-image from prod to dev, reads the metadata, adds layers (what the dev is doing), and pushes only the layers. Skopeo copies that image between registries, and Tilt then fetches the new one to use for dev, without devs ever building an image themselves.
- They have "live Tilt service discovery." If you have Tilt running and you clone a new service into your project, Tilt detects it and gets it up and running automatically.
- They wrote little automations (potential extensions) for tons of things.

## Takeaways

Patterns:
- Everyone uses a templating solution for their config files. A big company like Lyft can roll their own; small companies use Helm.
- Dev clusters. Generally, small companies use local clusters and big companies use remote clusters. Generally local clusters are easier, and they migrate to remote once things don't fit a laptop anymore.
- Companies try to keep the number of tools small. Helm, Tilt, CI* is a common pattern.
    - Integration with tracing & debugging tools is generally missed but not implemented.
- There's a pattern of standardising services to fit a same mold. This way you can recycle configs, live reload settings, etc., for free once the template is done.
- For sharing cluster state:
    - Big companies: you have a namespace and your colleague just logs into it
    - Small companies: each dev does one thing, no sharing
    - Middle ground: snapshots (CAPI)
- Everyone uses normal CI to send things to prod, and everyone hates (TODO: find out why exactly) it but there's no clear solution.

Ideas (from chat with dmiller—to be expanded/clarified):
- The patterns we see are very low-level, infra layer. There are higher level patterns still for development that are not codified. Like debuggers, tracing, running tests.
- Everyone has pre-packaged YAML/Dockerfile, but we don’t have pre-packaged dev workflow.
- Microservices, the benefit is you don’t have to care about the language/framework. For dev you do care about it. You can make microservice dev be as awesome as microservice prod.

What we're doing about this:
- Extensions so people stop reinventing the wheel
- Extension sets do standardise higher level patterns

## Outline

- Hi
- The general problem
- Why I’m qualified to talk about this
- The problem set
    - Dev cluster set up
    - Dealing with config files
    - Feedback loop automation
    - Cluster context sharing
    - Debugging
    - Taking all of this to prod
- Solutions apply for different settings
- Intro to Lyft, Mindspace, Unu, Cluster API
- Problem set vs. Lyft
- Problem set vs. Mindspace & Unu
- Problem set vs. CAPI
- Takeaways
- What we're doing about it
