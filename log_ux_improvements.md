# Log UX Improvements

November 11, 2019

## Overview

The purpose of this document is to enumerate problems with how we handle logs,
and suggest some small, incremental things we can do to move things forward.

## Background

Tilt started as a terminal app. Everything printed to a single-text stream of
logs.
 
As Tilt has grown, users have wanted more features and structure:

- Copy-and-paste from the main log pane to not include divider characters
- The ability to toggle timestamps at runtime
- Toggling/visualizing log levels (info, debug, verbose)
- Build and run logs organized by manifest
- Build logs for the last build
- Pod logs only
- The ability to silence a particular container
- Extracted warnings and errors
- Links to warnings and errors in-context with other logs
- System logs (like docker-prune) separate from build/run logs
- Parallel builds, with logs appropriately separated

Additionally, the way we store and render logs makes them slow in the browser.
 
Each time a new log-line appears, we re-send the entire log state, duplicated
multiple times across the data model.
 
## Goals
 
The logs situation has been so bad for so long. Let’s just propose some small,
quick wins that move us in a better direction towards addressing some of the
concerns above.

## Non-goals / Future-Goals

Logs Storage - There are some improvements we could make around storage to make
the logs easier for Tilt developers to work with. But we want to focus on
end-user UX improvements before we optimize.
 
Third-party Integrations - Users in the #tilt channel have already asked about
setting up Prometheus / Grafana for local clusters. Eventually, many users might
use third-party log viewers with Tilt. But the more I think about it, the more I
expect we’ll end up needing both - an acceptable first-party logs viewer and
better integrations.
 
Tracing/Metrics - Metrics and tracing are useful - for example trying to debug
why Frieda’s build is slow, or identify that build speeds have regressed in the
past N days.
 
But the product stories are more complex (e.g., Laura debugging her own build vs
Frieda debugging her own build vs Laura diagnosing a team-wide regression), and
will require tighter product feedback loops with a user.
 
## Current design

Let’s enumerate how logs currently get to the browser.
 
### Data Model
 
Tilt stores each stream of logs in a LogStore. A LogStore is a very simple data
structure that stores logs by line. It has some simple utility functions for
prefixing each line and tracking timestamps.
 
The EngineState contains multiple LogStores, one for each visualizable “stream”.
A global LogStore with all the logs Per-manifest LogStores for each Manifest’s
logs Per-pod and per-build LogStores for each of those logs
 
A log line may appear in multiple log stores. For example, a pod log will appear
in the per-manifest log store and in the global log store with prefixes
indicating where it came from.
 
### Log Production
 
When code wants to send a log line, it has a few options:

A java-like logging API (Infof, Debugf) that lets you emit a discrete log line.
`logger.Get(ctx).Infof(“log line”)`

An io.Writer API that lets you write bytes to the logger at a pre-determined
level `logger.Get(ctx).Writer(logger.Infof).Write([]byte{})` The io.Writer is
useful for forwarding log streams from other tools (like Buildkit) that don’t
write logs in discrete lines
 
A Pipeline API that lets you break up logs into steps
https://github.com/windmilleng/tilt/blob/d7bc74c9b359290f3af585a34988438643295211/internal/build/pipeline_state.go#L13
Useful for emitting structured build steps

Just emitting an action with the log info. This is how TiltfileLoader emits
warnings, for example.
 
All of these logging APIs eventually goes through the Store/Reducer/Action
system, the central state loop that Tilt uses to accumulate state. Most of them
implement LogAction. The LogAction keeps track of metadata about the log, like
what Manifest it came from.
 
The central state loop atomically replicates the log to all the necessary
LogStores.
 
When the reducer adds a line to the global log, it prefixes that log line with
the manifest name.
 
Other types of prefixes (like build log and PipelineStep prefixes) are added to
the line before it even gets to the Store.
 
### Log Consumption
 
Every time the EngineState changes, the Store broadcasts those changes to all
subscribers.
 
The Browser websocket is just another subscriber.
 
Each time a log comes in, the WebsocketSubscriber serializes the entire state to
a Webview protobuf. The Webview protobuf contains all the same log stores as the
engine state, but serialized as strings.
 
## Suggested improvements

#### (1) Change LogStore to store logs with metadata, and Spans for each Manifest’s log stream

```go
type LogStore struct {
  Spans map[SpanID]Span 
  Lines []LogLine
}
 
Type Span struct {
  ManifestName model.ManifestName
}
 
type LogLine struct {
  SpanID string 
  Time time.Time
  Level logging.Level
  Text []byte
}
```
 
We could make this change under the covers, without changing what we send to the
webview/browser, or what we store in individual build logs.
 
#### (2) Replace the global log in the webview protobuf with a structured log store
 
The browser JS would “pull out” the global log and manifest-log client-side.
 
We wouldn’t change other data we send, like the individual build logs or pod
logs.
 
 
Once steps (1) and (2) are done, there are a bunch of experiments to improve the
UX that we can run in parallel:
 
#### (3) Experiment with different ways of displaying the manifest name with HTML instead of text.
 
At this point, we’re no longer interpolating the manifest-name into the log
lines on the server. We do it on the client. So we can play around with
different ways to show them, like with colors, or by linking the name to the
resource view.
 
#### (4) Experiment with different ways of toggling timestamps or log levels
 
With structured logs, we can decide at render-time which info to show.
 
#### (5) Experiment with moving Tiltfile Warnings into the LogStore
 
This is a bit tricker than it sounds. You have to have some notion of “the
Tiltfile is re-executing, so the Span the covered the last execution is no
longer in-play”. But I think that moving in this direction will make it a lot
easier to emit good warnings (with logger.Warnf) and move us towards dismissable
warnings and errors.
 
## Alternative Approaches
 
### Parsing Structure Out Of Text Logs
 
You could run a lot of the experiments above by “parsing” the manifest name out
of the text logs.
 
This would allow us to experiment a little faster, but I think the cost would be
too high. It would be brittle, and take us too far away from where we want to
go. Once we went too far down this path, it would be hard to go back.
 
### Each Span Maintains its Own Logs

You could imagine a system where each span maintains its own logs.
 
Then, when we need to render the global log, we “merge” all the logs from
individual spans.
 
This approach would also work. But I think there are more unknowns around how
many Spans we’ll end up with, whether they’ll be short-lived (like builds) or
long-lived (like manifests). I could imagine that we start with a single array
of logs, and eventually move to per-span logs as we better understand the
problem.

## Further Reading

OpenTelemetry Spec
https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/overview.md
for some of the primitives around Spans and Traces
 
 
 
 
