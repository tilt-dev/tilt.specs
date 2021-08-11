# Extension Distribution Spec

## Problem

Tilt's current extension system is optimized to support libraries of
Tiltfile/Starlark functions.

Rather than copy-and-pasting snippets of Tiltfiles, you load
those snippets from a common location. To use an extension, the Tiltfile
you're working on needs to manually load the extension.

We know that most extensions get very low usage. The ones that do get a lot of
usage are linked directly from the main Tilt docs.

How can we better encourage an ecosystem where users discover and distribute
more sophisticated extensions that interact with the Tilt API?

## Primary Goal

Support reactive extensions that watch the Tilt API server, and add reactive
functionality that augments whatever servers you have.

The flagship example is a "kill command" button that watches for Cmds,
then dynamically adds a button to kill them.

## Secondary Goal

### Extension Ecosystem

Set us up for a more robust ecosystem of extensions that are easier to:

- discover
- prototype
- install
- distribute

### API Consistency


We see the future of Tilt extensibility as the API Server and Kubernetes
operator pattern.

What that means: there's an HTTP server that allows you can read, watch, and
write data models. Long-term, to add a new feature to Tilt, you will add a new
resource/controller pair.

Users should be able to add extensions from the Tiltfile or from the CLI with a
similar API.

## Non-Goals

### A Long-term Extension Roadmap

This isn't intended to be a long-term roadmap of the extensions ecosystem.  We
gesture a bit at the future. But we aren't promising any specific timelines.

For example, I think we will eventually need some sort of extensions
marketplace.  But I don't think we should do that right now. Maybe a good
analogue is Helm: right now we're focused on the analgue for Helm repos, not the
analogue for ArtifactHub or CharmHub.

### Tiltfile 

Per above, we want the Tiltfile API to converge towards Kubernetes API conventions.

We do NOT think that adding special hooks and extension points to the Tiltfile
execution model is the right answer.  We don't want special Tiltfile data flow or overrides.

## Prior Art

### Other Extension Systems

Some other extension systems that we looked to for inspiration:

#### VSCode extensions (https://code.visualstudio.com/docs/editor/extension-marketplace)

You install them into VSCode itself (rather than a particular repo).

They typically add things to the Command Palette. But there are also extensions
that manage other tools on your machine (e.g., the Minikube extension makes sure
Minikube is up to date).

#### Browser extensions

These typically come in two flavors:

- Extensions that add buttons or capapilities to the browser itself (like [tabby
  cat](https://chrome.google.com/webstore/detail/tabby-cat/mefhakmgclhhfbdadeojlkbllmecialg?hl=en),
  the best extension)

- Extensions that modify pages you visit, like Google Translate.

#### Gradle Plugins (https://docs.gradle.org/current/userguide/implementing_gradle_plugins.html)

Add custom task types to your build system, or decorate existing tasks with new
outputs. Here are some examples: https://github.com/ksoichiro/awesome-gradle

#### Bazel Extensions (https://docs.bazel.build/versions/main/skylark/concepts.html)

Mostly focused on adding macros and custom build rules.

#### Helm Chart Repositories (https://helm.sh/docs/helm/helm_repo/)

The Helm repo command lets you install new repositories of charts. Each chart is a set of resources
to deploy. These charts can add new types and operators to your Kubernetes cluster.

The chart repos are globally installed, but the charts are installed to a particular cluster.

#### yarn add / yarn global add

In the javascript ecoysystem, packages are installed locally (in the current
repo) or globally (in a shared global package tracker). Packages can contain
both library code that you have to explicitly import, but also contains
executables that you can run.

## Proposal

We'll add three new data types to the APIServer:

- `ExtensionRepo`: A repository of Tilt extensions.
- `Extension`: An installed extension.
- `Tiltfile`: A Tiltfile invocation.

Each of these are stored in-memory only, and disappear when Tilt is killed.

But ExtensionRepo and Extension correspond to checked-out extensions on disk,
which are deliberately not cleaned up when Tilt exits.

We'll illustrate how this works with examples.

### ExtensionRepo

```go
type ExtensionRepoSpec struct {
  // The URL of the repo.
  // 
  // Allowed:
  // https: URLs that point to a public git repo
  // file: URLs that point to a location on disk.
  URL string
  
  // A reference to sync the repo to. If empty, Tilt will always update
  // the repo to the latest version.
  Ref string
}
```

```go
type ExtensionRepoStatus struct {
  // Contains information about any problems loading the repo.
  Error string

  // The last time the repo was fetched and checked for validity.
  // This corresponds to the ModTime of the repo on-disk.
  LastFetched metav1.Time
}
```

Once an ExtensionRepo is registered with the API server, Tilt will fetch that repo
under ~/.tilt-dev, even if no extensions fetch that repo.

### Extension

Each `Extension` will manage a child object, a `Tiltfile`, which manages the
extension execution for a particular Tilt session. The Tiltfile and Extension
object should have the same name.

The name of the extension is displayed in the Tilt UI as a new Tiltfile-like
resource.


```go
type ExtensionSpec struct {
  // RepoName specifies the ExtensionRepo object where we should find this extension.
  RepoName string
  
  // RepoPath specifies the path to the extension directory inside the repo.
  RepoPath string
}
```

```go
type ExtensionStatus struct {
  // Contains information about any problems loading the extension.
  Error string
  
  // The path to the extension on disk. This location should be shared
  // and readable by all Tilt instances.
  Path string
}
```

### Tiltfile

We'll adapt the existing Tiltfile executor into an API Object. Most of the
implementation will be copied and pasted from the existing ConfigsController.

Each Tiltfile loads independently, and appears in the UI as its own resource.

```go
type TiltfileSpec struct {
  // The path to the Tiltfile on disk.
  Path string
  
  // A set of labels to apply to all objects owned by this Tiltfile.
  Labels map[string]string
}
```

```go
type TiltfileStatus struct {
  // Details about a waiting tiltfile execution.
  Waiting *TiltfileStateWaiting

  // Details about a running tiltfile execution.
  Running *TiltfileStateRunning

  // Details about a terminated process.
  Terminated *TiltfileStateTerminated
}
```

The TiltfileStatus is analogous to CmdStatus, but we'll sketch out a bit what it should look like:

```go
type TiltfileStateTerminated {
  // The reason why this tiltfile was built. 
  // May contain more than one reason.
  Reason []string
  
  // Non-empty if the last execution was an error.
  Error string
  
  // Time at which previous execution of the tiltfile started
  StartedAt metav1.MicroTime

  // Time at which previous execution of the tiltfile ended
  FinishedAt metav1.MicroTime
  
  WarningCount int
}
```

### Tiltfile API

To start, users will install extensions from the main Tiltfile. The syntax will look like this:

```
extension_repo(name='default', url='https://github.com/tilt-dev/tilt-extensions')
extension(name='cancel_button', repo_name='default', repo_path='cancel_button')
```

We have deliberately designed this API so that it can be auto-generated from the above models.

These functions simply register the API objects with the API
Server. Reconcilation (read: downloading the extension and executing its code)
happens asynchonously.

## Future Work

### CLI Support

A common pattern in Kubernetes is to add CLI sugar for common resources, like
Secrets. These are called generators.

https://kubernetes.io/docs/reference/kubectl/conventions/#generators

We want people to be able to create / delete extensions and extension repos the
same way.

```
tilt create repo my-team https://github.com/my-team/tilt-extensions
tilt create ext my-team/image-builder
```

### Namespacing

Extensions and Tiltfiles can be loaded independently and in parallel.

But they register resources with the same APIServer.

Inevitably, you will have problems where two extensions try to, say, create a
local_resource the same way. We don't currently have any technical solution to
prevent this.

In the short-term, we can use the ostrich approach - stick our heads in the sand
and pretend it can't happen.

### Meta Extensions

Long-term, I expect we'll have services that manage other services, including:

- The tilt-inspector - https://github.com/tilt-dev/tilt-inspector
- A button manager
- An extension manager

which can all, themselves, be extensions implemented on this system, and
can introspect on the list of extensions.

### Resource Grouping

We're currently working on a system for grouping resources by label.

I deliberately intended this with the labeling system in mind. I could imagine 3
different ways labels and extensions play well together:

- Labels communicate which resources belong to which extension, so
  that users can minimize services that belong to extensions.
  
- Labels communicate which extensions are enabled for specific projects (similar
  to how browser content-scripts are enabled for specific domains).
  
- Labels communicate which how an extension should interoperate with 
  an existing resource (similar to how Kubernetes uses annotations
  https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/
  to have resources communicate with custom controllers).
  
But I've punted on the specifics of how this should work.

### Concurrency

Extension repos are installed under `$XDG_DATA_HOME/tilt-dev` (default to `~/.local/share/tilt-dev`).

If you have two tilt instances running at the same time, there's a possibility
they will step on each other's state.

If one tilt instance updates the repo, the other tilt instance will not get notified.

I think this is a real problem, but not worth worrying about for the initial
implementation.

### Vendoring

Extensions and Extension repos are stored in the users' home directory.

They are not copied into the project directory.

In the future, I think it would make sense to add a `Vendor` boolean field to either
ExtensionRepo or Extension. If `Vendor` is set, the reconciler would checkout the
code into the project directory instead of the global directory.

We would need some way to unify this with the existing `tilt_modules` directory
(for `load('ext://...')`). But I don't think it makes sense to unify them at
this stage.  The extension API model is still evolving.

