# Badging API Explainer

Author: Matt Giuca <mgiuca@chromium.org>

Date: 2018-06-26

## Overview

The **Badging API** is a proposed Web Platform API allowing web applications (as
defined by the [Web App Manifest](https://www.w3.org/TR/appmanifest/) standard)
to set an **application-wide** badge, shown in an operating-system-specific
place associated with the application (such as the shelf or home screen).

![Windows taskbar badge](images/uwp-badge.png)
<br>Windows taskbar badge

![macOS dock badge](images/mac-badge.png)
<br>macOS dock badge

![Android home screen badge](images/android-badge.png)
<br>Android home screen badge

The purpose of this API is:

* To give the application a small but visible place to subtly notify the user
  that there is new activity that might require their attention, without showing
  a full [notification](https://notifications.spec.whatwg.org/).
* To indicate a small amount of additional information, such as an unread count
  or symbol indicating the type of event.
* To allow the application to convey this information regardless of whether any
  of the application's windows are open.

Non-goals are:

* To provide an arbitrary image badge.

Possible areas for expansion:

* Providing badging for sites in a normal web browsing context. The current
  proposal is just for installed apps (designed to show up in the operating
  system shelf area). We could also explore icon badging on the drive-by web.
  This naturally leads into...
* Provide per-window badging information. The current proposal is for a global
  badge for the application, not per-window or per-tab. See
  [#1](https://github.com/mgiuca/badging/issues/1).

Examples of sites that may use this API:

* Chat, email and social apps, to signal that new messages have arrived.
* Productivity apps, to signal that a long-running background task (such as
  rendering an image or video) has completed.
* Apps that want to render a small status indicator (e.g., a music app shows ▶️
  or ⏸️; a weather app shows ⛈️ or ⛅️).
* Games, to signal that a player action is required (e.g., in Chess, when it is
  the player's turn).

Advantages of using the badging API over notifications:

* Can be used for much higher frequency events than notifications, because each
  new event does not disrupt the user.
* There is no need to request permission to use the badging API, since it is
  much less invasive than a notification.

(Typically, sites will want to use both APIs together: notifications for
high-importance events such as new direct messages or incoming calls, and badges
for all new messages including group chats not directly addressed to the user.)

## API proposal

### The model

There is a single global badge associated with each Web
application (as defined in [Web app
manifest](https://www.w3.org/TR/appmanifest/)). At any time, the badge is set to
either:

* Nothing (the badge is "cleared"), or
* A "flag" indicating the presence of a badge with no contents, or
* A non-empty string containing a single *[grapheme
  cluster](http://unicode.org/reports/tr29/#Grapheme_Cluster_Boundaries)*
  (roughly: a single Unicode character), or
* A positive integer.

The model does not allow a badge that is the empty string, a negative integer,
or the integer value 0.

The grapheme cluster allows a single character to be represented, but also
for combining characters to be combined into what the user thinks of as a single
character. Examples of grapheme clusters include:

* "a" (a single character, Latin "a")
* "நி" (a base character + combining character, Tamil "ni")
* 👩🏾‍🔬️ (emoji combination with skin tone modifier, Woman Scientist with
  Medium-Dark Skin Tone)

User agents are encouraged to use integers instead of digit characters, because
some operating systems may *only* support integer badges, so using the digit
character would not display correctly. Also integers can be localized better for
the user.

### The API

The `Badge` interface is a member object on
[`Window`](https://html.spec.whatwg.org/#the-window-object) and
[`Worker`](https://www.w3.org/TR/workers/#worker). It contains two methods:

* `void set(optional USVString or long)`: Sets the associated app's badge to the
  given data, or just "flag" if the argument is not given.
* `void clear()`: Sets the associated app's badge to nothing.

These can be called from either a foreground page or a service worker (in either
case, affecting the whole app, not just the current page).

TODO: An issue is that if the methods are called from a service worker whose
scope is a parent of the web app manifest scope, it would be ambiguous which web
app is being identified. We need to take an optional scope parameter.

Example code (from in a service worker):

```js
self.addEventListener('sync', () => {
  self.Badge.set(getUnreadCount());
});
```

## UX treatment

* Installed Web applications are typically given some presence in an
  operating-system location, such as the shelf, launcher, dock, home screen,
  etc. Usually the application is represented by an icon.
* The badge would appear as a user-agent-defined overlay on top of the app's
  icon, about a quarter of the size and in one of the four corners, as shown in
  the sample images above.
* Operating systems typically provide a badge system for native applications;
  user agents should re-use existing operating system APIs / UI and conventions
  to achieve a native look-and-feel.
* The user agent should make a "best effort" attempt to map the badge data
  structure onto the host operating system's badge format:
  * Some operating systems (e.g., Android) only provide UI for a Flag badge;
    just a coloured dot with no content (see the sample image above). In these
    cases, the user agent should follow this convention, and only show a Flag,
    even if the website sets richer badge data.
* If the operating system doesn't allow the exact representation of the badge
  (e.g., a 2-digit number but the OS only allows a single character, or a
  character but the OS only allows a number), the user agent should try the best
  to map into the OS representation. This may involve:
  * Saturating a number; e.g., 351 -> "99+".
  * Representing a character as a number; e.g., "?" -> "1".
  * Truncating a grapheme cluster; e.g., "நி" -> "ந".

## Specific operating system treatment

This section describes a possible treatment on each major OS. User agents are
free to implement however they like, but this should give an idea of what the
API will look like in practice.

### macOS

Requires macOS 10.5 or higher.

* The host API is a string, truncated from the center to a system-defined
  length, as shown/described
  [here](https://eternalstorms.wordpress.com/2016/10/29/how-to-badge-an-apps-icon-in-the-dock/).
  The API is
  [`NSDockTile.setBadgeLabel`](https://developer.apple.com/documentation/appkit/nsdocktile/1524433-badgelabel).
* String badges are just passed to `setBadgeLabel`. All single-grapheme-cluster
  strings should show without truncation. The character is shown as white text
  in a red circle.
* Flag badges are passed as an empty string to `setBadgeLabel` (if that works,
  otherwise pass a space character). This shows an empty red circle.
* Integer badges are saturated to 3 or 4 digits (with a "+" if overflowing), and
  passed as a string to `setBadgeLabel`.

### Universal Windows Platform

Requires Windows 10 or higher, and requires that the user agent be a "UWP" app.
(TODO: Unless this is one of those UWP APIs that can be called from Win32 code?
Need to investigate.) Note that Google Chrome and Mozilla Firefox do not fall
into this category, and likely don't have access to the necessary APIs.

* The host API is
  [`BadgeNotification`](https://docs.microsoft.com/en-us/uwp/api/windows.ui.notifications.badgenotification),
  which allows either a positive integer (saturated at 99) or a fixed glyph
  (shown
  [here](https://docs.microsoft.com/en-us/windows/uwp/design/shell/tiles-and-notifications/badges)).
* Integer badges are just passed straight through into this API.
* Flag badges are passed as a fixed glyph, perhaps "attention", since I don't
  think there is a way to show an empty circle.
* String badges: if the string equals a Unicode character corresponding to one
  of the fixed glyphs, use that glyph (e.g., "✉" -> "newMessage"). Otherwise,
  fall back to the same as the Flag badge, "attention".

This will show up on both the Taskbar and Start Menu tile.

### Legacy Windows (Win32)

Requires Windows 7 or higher.

* The host API is
  [`ITaskbarList3::SetOverlayIcon`](https://docs.microsoft.com/en-au/windows/desktop/api/shobjidl_core/nf-shobjidl_core-itaskbarlist3-setoverlayicon),
  which allows applications to set a 16x16-pixel overlay icon in the corner of
  the application's main icon in the Taskbar.
* Due to the nature of being a 16x16-pixel icon, the user agent must render the
  text or number into an image. It pretty much has to be 1–2 characters.
* Flag badges are just passed as a coloured circle.
* String badges are passed as a rendering of the single grapheme cluster in a
  coloured circle.
* Integer badges are rendered as the number, if a single digit, or "+", if
  greater than 9.

### Android

Requires Android 8.0 (Oreo) or higher.

* The host API is
  [`NotificationChannel`](https://developer.android.com/training/notify-user/badges),
  which lets you set a number only. The badge is usually shown as a coloured
  dot; the number is only shown on a long-press.
* Tricky: The API is tied to notifications. You can't show a badge unless there
  are pending notifications.

### Chrome OS

Not currently supported, but will [soon be available](https://crbug.com/801014)
for Android apps using the above `NotificationChannel` API. The Chrome browser
itself will be able to re-use this mechanism for showing a coloured dot on web
application icons.

### iOS

* The host API is
  [`UIApplication.applicationIconBadgeNumber`](https://developer.apple.com/documentation/uikit/uiapplication/1622918-applicationiconbadgenumber),
  which lets you set a positive integer only.
* Integer badges are passed directly to the host API.
* Strings that are numeric digits are converted to a number and passed to the
  host API.
* Flag badges and strings other than digits are just represented by the number
  "1".

### Ubuntu

Requires Ubuntu (no general API for Linux).

* The host API is
  [`unity_launcher_*`](https://wiki.ubuntu.com/Unity/LauncherAPI), which lets
  you set an integer only.
* See iOS treatment.

### Summary

* All known host OSes support some form of badge.
* Integer badges are always supported (but sometimes translated).
* Most platforms support showing at least one arbitrary character. UWP limits to
  a small selection of glyphs. iOS and Ubuntu can only show numbers.

Thus, a fallback option for platforms that do not support arbitrary characters
(e.g., choose whether to show a number, or nothing) may be necessary.

## FAQ

### What data types are supported in different operating systems?

See above.

### Could the API take a fallback type?

The proposal is: what if instead of just taking *one of* the string or integer,
we allow sites to pass both, with an order of preference. This way, you could
supply a string, but if strings aren't supported, fall back to a given number,
or vice versa.

This is something we're considering. My concern is that it makes the API too
complex, where practically it isn't required (e.g., I believe all major
platforms support at least one character strings).

### Why limit the support to a single grapheme cluster? Is there a technical limitation?

It isn't a technical limitation. It's an attempt to keep the behaviour as
consistent as possible between different host platforms.

We could say "provide an arbitrarily long string, and we'll truncate it", but
having some platforms truncate to just 1 or 2 characters, and others showing a
bit more text, makes it too unpredictable. Limiting to precisely one character
levels the playing field.

### Is there an upper limit on the size of the integer? And if so, what's the behavior if that limit is reached?

There is no upper limit (besides 2<sup>31</sup>). However, each user agent is
free to impose a limit and silently saturate the value (e.g., display all values
above 99 as "99+").

This is different to the string, since truncating a string may change or destroy
its meaning, whereas the integer always has the semantics of counting, and thus
saturating the integer still preserves most of its meaning (i.e., "lots").

### Are you concerned about apps perpetually showing a large unread count?

Yes. If users habitually leave mail or chats unread, and mail or chat apps
simply call `set(getUnreadCount())`, it could result in several apps simply
showing a large number, presenting several issues:

* Leaving "clutter" on the user's shelf, and
* Making the user unable to tell when new messages arrive.

However, the only solution to this is a much more limited API which only lets
you show the count of notifications (or similar). We wanted to give apps the
full power of showing a native badge.
