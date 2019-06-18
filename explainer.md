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
* To indicate a small amount of additional information, such as an unread count.
* To allow the application to convey this information regardless of whether any
  of the application's windows are open.

Non-goals are:

* To provide an arbitrary image badge.

Possible areas for expansion:

* Providing badging for sites in a normal web browsing context. The current
  proposal is just for installed apps (designed to show up in the operating
  system shelf area). We could also explore icon badging on the drive-by web.
  This naturally leads into...
* Providing per-tab and per-window badging. The current proposal is for a global badge
  for the application. See [#1](https://github.com/WICG/badging/issues/1).
* Support apps that want to render a small status indicator (e.g., a music app shows ▶️	
  or ⏸️; a weather app shows ⛈️ or ⛅️).
* Setting the badge from a service worker (e.g. an email app updating an unread count).

Examples of sites that may use this API:

* Chat, email and social apps, to signal that new messages have arrived.
* Productivity apps, to signal that a long-running background task (such as
  rendering an image or video) has completed.
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
manifest](https://www.w3.org/TR/appmanifest/)). At any time, the badge is set with:

* Nothing (the badge is "cleared"), or
* A "flag" indicating the presence of a badge with no contents, or
* A positive integer.

The model does not allow a badge that is a negative integer, or the integer value 0
(setting the badge to 0 is equivalent to clearing the badge).

### The API

The `Badge` interface is a member object on
[`Window`](https://html.spec.whatwg.org/#the-window-object). It contains two methods:

* `void set(optional unsigned long long)`: Sets the associated app's badge to
  the given data, or just "flag" if the argument is not given.
* `void clear()`: Sets the associated app's badge to nothing.

These can be called from a foreground page only (calling from a service worker is being
considered for the future).

TODO: An issue is that if the methods are called from a service worker whose
scope is a parent of the web app manifest scope, it would be ambiguous which web
app is being identified. We need to take an optional scope parameter.

Example code (from the main window):

Setting an integer badge (as in an email app):
```js
Badge.set(getUnreadCount());
```

Setting and clearing a boolean flag (as in a game of chess):
```js
if (myTurn())
  Badge.set();
else
  Badge.clear();
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
* Flag badges are passed as an empty string to `setBadgeLabel` (if that works,
  otherwise pass a space character). This shows an empty red circle.
* Integer badges are saturated to 3 or 4 digits (with a "+" if overflowing), and
  passed as a string to `setBadgeLabel`.

### Universal Windows Platform

Requires Windows 10 or higher, and requires that the user agent be a "UWP" app.
Note that Google Chrome and Mozilla Firefox do not fall into this category, and
hence don't have access to the necessary APIs.

* The host API is
  [`BadgeNotification`](https://docs.microsoft.com/en-us/uwp/api/windows.ui.notifications.badgenotification),
  which allows either a positive integer (saturated at 99) or a fixed glyph
  (shown
  [here](https://docs.microsoft.com/en-us/windows/uwp/design/shell/tiles-and-notifications/badges)).
* Integer badges are just passed straight through into this API.
* Flag badges are passed as a fixed glyph, perhaps "attention", since I don't
  think there is a way to show an empty circle.

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
* Integer badges are rendered as the number, if a single digit, or "+", if
  greater than 9.

Note: This API only allows badging on a currently open window.

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

### What data types are supported in different operating systems?	

See above.

### Why limit support to just an integer? What about other characters?

It isn't a technical limitation, it's an attempt to keep behavior as consistent as possible
on different host platforms (UWP only supports a set of symbols, while iOS, Android
and Ubuntu don't support them at all).

Limiting support to integers makes behavior more predictable, though we are considering
whether it might be worth adding support for other characters or symbols in future.

### Why isn't this API available from a Service Worker?

Ideally, it would be, and we're considering this for a future version of this API. 
However, the API will require some more thought, as there can be multiple apps installed
for a single service worker, so we would need some way of specifying which one should
be badged.

### Is there an upper limit on the size of the integer? And if so, what's the behavior if that limit is reached?

There is no upper limit (besides 2<sup>31</sup>). However, each user agent is
free to impose a limit and silently saturate the value (e.g., display all values
above 99 as "99+").

### Are you concerned about apps perpetually showing a large unread count?

Yes. If users habitually leave mail or chats unread, and mail or chat apps
simply call `set(getUnreadCount())`, it could result in several apps simply
showing a large number, presenting several issues:

* Leaving "clutter" on the user's shelf, and
* Making the user unable to tell when new messages arrive.

However, the only solution to this is a much more limited API which only lets
you show the count of notifications (or similar). We wanted to give apps the
full power of showing a native badge.

### Security and Privacy Considerations
The API is set only, so data badged can't be used to track a user. Whether the API is present could possibly be used as a bit of entropy to fingerprint users, but this is the case for all new APIs.

