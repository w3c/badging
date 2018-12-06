# Open choices

This document covers all of the choices we're considering and haven't yet nailed
down for the design.

Many of these are also covered by issues, but I want a summary doc (this).

## Location of the API functions

Issue: [#14](https://github.com/WICG/badging/issues/14)

Options:

- `[window.]Badge.xxx` (incumbent)
- `[window.]badge.xxx`
- `navigator.Badge.xxx`
- `navigator.badge.xxx`
- `navigator.xxxBadge` (e.g., `setBadge` and `clearBadge`)

## Function arrangement / argument types

Issue: [#19](https://github.com/WICG/badging/issues/19)

Options:

- `set(number)` to set a number (0 clears), `set()` to set a flag. `clear()`
  clears. (incumbent)
- `set(number)` to set a number (0 clears), `set(bool)` to set a flag (false
  clears). No "clear" method.
- `setNumber(number)` to set a number (0 clears), `setFlag(bool)` to set a flag
  (false clears). No "clear" method.
- `setNumber(number)` to set a number (0 clears), `show()` to set a flag,
  `hide()` clears.

## Should there be a function to read the badge back out?

Issue: [#18](https://github.com/WICG/badging/issues/18)

Options:

- No (incumbent)
- Yes

## Data model

Issue: [#13](https://github.com/WICG/badging/issues/13)

Committed to supporting positive integers and "flag" (badge with no data).
Options for further content values:

- Nothing further (incumbent)
- Strings, where we take just the first [grapheme
  cluster](http://unicode.org/reports/tr29/#Grapheme_Cluster_Boundaries)
  (i.e., the first character). (previous design before we took it out)
- A curated list of semantic values with user-agent-defined symbols, similar to
  the [UWP
  API](https://docs.microsoft.com/en-us/windows/uwp/design/shell/tiles-and-notifications/badges).

## Scope of a badge (origin vs app vs window vs tab)

Issues: [#1](https://github.com/WICG/badging/issues/1),
[#12](https://github.com/WICG/badging/issues/12)

- Badge must be set from an installed app scope (foreground or service worker).
  Badge state is global for that app, and appears on all windows of that app.
  (incumbent, but at risk)
- Setting from a service worker is weird, because doesn't necessarily correspond
  to app scope. May need to have an explicit scope parameter.
- Implementation: On several platforms (at least Windows 7 and macOS), it is
  harder or impossible to set a badge on an app with no open windows.
- May want to decouple from being installed, and allow the user agent to set
  badges per-tab.
- For installed apps, may want to set badges per-window, rather than for the
  whole app. (However, on some platforms, e.g., Chrome OS and macOS, it doesn't
  make sense to set a badge per-window, since there's only one shelf icon for
  all windows of an app.)
- It may make sense to add this choice as a runtime optional argument.
