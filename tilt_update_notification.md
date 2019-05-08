# Tilt Update Notifications

### Problem
Our analytics show a significant number of our users are on old versions of Tilt. This means 1) they are not benefitting from bug fixes and feature enhancements, and 2) we do not learn about their Tilt usage via our improving analytics.

### Proposal
TL;DR: do what Chrome does -- periodically check if there's a newer version, and display a little icon/link in the corner of the browser if there is.

Pros:
* Chrome users will be used to it
* It informs users who leave Tilt running for long periods
* It's not super naggy

Cons:
* Not dismissable, so might be mildly annoying, but Chrome does it, so people will probably be fine with it.
* Making it not annoying means it's more likely to be ignored, but c'est la vie.
* Only communicates the information to users when they check the web UI. We're operating on the unvalidated but seemingly reasonable assumption that users see the web UI sufficiently often for this mode of notification to be useful (especially given that we now auto-open the web on startup).

We'll perform the version check on startup, and every 4 hours thereafter (number pulled out of a hat).

Chrome's update colors are:
* 2-4 days: green
* 4-7 days: orange
* 7+ days: red

Presumably, there's no notification before two days so that they're not prompting users to upgrade until a new version has baked a bit (or, if that's not the motivation, it seems like a nice side effect).

I think our user base isn't big enough to worry about that, so we should just go:
* 0-4 days: green
* 4-7 days: orange
* 7+ days: red

Work to do:
* implement version check (maybe using [this](https://github.com/google/go-github/blob/master/github/repos_releases.go)?)
* add the icon to the web UI
* ensure we have a landing page for the icon's link (maybe just the installation guide, with a bit of edits?)

For simplicity for now, we'll simply disable the version check on dev versions. If metrics show many users on old dev versions, we can revisit this decision.

### Alternatives
1. Actually perform the upgrade for the user.
  1. This significantly increases complexity since there are several means of installation, some of which require sudo.
2. On startup (before the TUI loads), check if there is a newer release available on GitHub (maybe via [go-latest](https://github.com/tcnksm/go-latest)). If there is, inform the user of the new version, link to upgrade instructions (also, ensure we have a doc to link to), and prompt the user with options (copy TBD):
  * `<enter>` OK (do nothing)
  * `i` Ignore this version
  * `d` Disable version check forever.
  Implement and honor those options by saving a setting somewhere in ~/.windmill/tilt.
  Cons:
  * Can feel mildly naggy.
  * Only prompts on startup, which means we're not informing users who leave Tilt running for long periods.
