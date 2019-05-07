# Tilt Update Notifications

### Problem
Our analytics show a significant number of our users are on old versions of Tilt. This means 1) they are not benefitting from bug fixes and feature enhancements, and 2) we do not learn about their Tilt usage via our improving analytics.

### Proposal
On startup (before the TUI loads), check if there is a newer release available on GitHub (maybe via [go-latest](https://github.com/tcnksm/go-latest)).
If there is, inform the user of the new version, link to upgrade instructions (also, ensure we have a doc to link to), and prompt the user with options (copy TBD):
* `<enter>` OK (do nothing)
* `i` Ignore this version
* `d` Disable version check forever.

After the user selects an option, Tilt will continue running as usual.
If the user selects (2), Tilt will write the version number to `~/.windmill/tilt/config.yaml`, and future invocations will skip the prompt if that version is the latest. Tilt will check this after checking the latest version and ignore if the versions match. If the versions don't match, this setting will be deleted (because it's now outdated) and ignored.
If the user selects (3), Tilt will write an option to `~/.windmill/tilt/config.yaml` that says to never perform a version check. Tilt will check this before performing a version check and skip if so specified.

### Alternatives
1. Actually perform the upgrade for the user.
  1. This significantly increases complexity since there are several means of installation, some of which require sudo.
2. Simply inform the user of the new version in the TUI/web UI.
  1. This feels naggy if it's not dismissable, and it's not necessarily cheap to make it dismissable. Maybe it's cheap to do in web? Like, just a dismissable "this site uses cookies" hovery popupy thing. Though, then it doesn't help out non-web users (is that even a persona we are interested in?)
